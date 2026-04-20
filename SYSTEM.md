# System setup — DGX Spark cluster

Perform following steps while ssh'ed into the system (watch out for the steps changing the network - so may need console)

## Create nvidia user

```bash
sudo useradd -m nvidia -s /bin/bash
sudo passwd nvidia
sudo usermod -aG sudo nvidia
sudo usermod -aG docker nvidia
```

Make sure group sudo may sudo without password:
```bash
sudo perl -pi -e 's/^%sudo.*/%sudo ALL=(ALL) NOPASSWD: ALL/' /etc/sudoers
```

Exit and ssh back into the machine as user nvidia to make it effective.

## Clean up the system

```bash
# disable gui
sudo systemctl set-default multi-user.target
sudo systemctl stop gdm3

# disable desktop-oriented system services
sudo systemctl disable --now cups cups-browsed cups.socket cups.path
sudo systemctl disable --now bluetooth
sudo systemctl disable --now ModemManager
sudo systemctl disable --now avahi-daemon avahi-daemon.socket
sudo systemctl mask colord
sudo systemctl disable --now rtkit-daemon
sudo systemctl disable --now wpa_supplicant
sudo systemctl disable --now multipathd multipathd.socket
sudo systemctl disable --now fwupd fwupd-refresh.timer fwupd-refresh.service
sudo systemctl mask fwupd
sudo systemctl disable --now udisks2 && sudo systemctl mask udisks2
sudo systemctl disable --now upower && sudo systemctl mask upower

# clean up user session
systemctl --user mask pipewire pipewire-pulse wireplumber pipewire.socket pipewire-pulse.socket
systemctl --user mask xdg-document-portal xdg-permission-store
systemctl --user stop pipewire pipewire-pulse wireplumber xdg-document-portal xdg-permission-store
sudo loginctl disable-linger $USER

# remove everything installed with snap, then disable snap itself
sudo snap remove snapd-desktop-integration
sudo snap remove --purge firefox thunderbird snap-store firmware-updater
sudo snap remove --purge gnome-46-2404 gnome-42-2204 gtk-common-themes mesa-2404
sudo snap remove --purge core24 core22 bare
sudo systemctl disable --now snapd snapd.socket snapd.seeded.service
```

## Configure networking

The ConnectX-7 on a DGX Spark exposes two QSFP56 ports, but internally each port is wired to two PCIe x4 links — so plugging in a single DAC cable brings up **two** net devices per node ("twin" pair). Each twin is a separate RoCE device and must live on its own subnet; sharing a subnet confuses NCCL/RDMA autodiscovery.

With one DAC cable connected, `enp1s0f0np0` and `enP2p1s0f0np0` are the active pair; the other pair (`.f1.np1`) belongs to the unused QSFP port and stays DOWN.

Topology:
```
                    172.31.0.0/23  (10GbE management LAN)
    ═══════════╤═════════════════════════════════════════╤═══════════
               │                                         │
               │ 172.31.0.100 (enP7s7)                   │ 172.31.0.101 (enP7s7)
       ┌───────┴───────┐                         ┌───────┴───────┐
       │   spark-702b  │                         │   spark-20db  │
       │               │                         │               │
       │  enp1s0f0np0  │     172.31.5.0/29       │  enp1s0f0np0  │
       │  172.31.5.1   ╞═════════════════════════╡  172.31.5.2   │
       │               │                         │               │
       │               │       QSFP56 DAC        │               │
       │               │  (one cable — two PCIe  │               │
       │               │    links / RoCE twins)  │               │
       │               │                         │               │
       │ enP2p1s0f0np0 │     172.31.5.8/29       │ enP2p1s0f0np0 │
       │  172.31.5.9   ╞═════════════════════════╡  172.31.5.10  │
       └───────────────┘                         └───────────────┘
```

Configuration (use `.1` / `.9` on node 1, `.2` / `.10` on node 2):
```bash
sudo rm -f /etc/netplan/*

sudo tee /etc/netplan/20-mediatek.yaml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    enP7s7:
      mtu: 1500
      dhcp4: true
      dhcp6: false
      link-local: []
EOF

sudo tee /etc/netplan/40-cx7.yaml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0f0np0:
      addresses: [172.31.5.1/29]
      mtu: 9000
      dhcp4: false
      dhcp6: false
      link-local: []
    enP2p1s0f0np0:
      addresses: [172.31.5.9/29]
      mtu: 9000
      dhcp4: false
      dhcp6: false
      link-local: []
EOF

sudo chmod 0600 /etc/netplan/*.yaml
sudo netplan generate
sudo netplan apply
```

Verify:
```bash
ip -br addr show | grep -E 'enp1s0f|enP2p1s0f|enP7s7'
ibdev2netdev
# jumbo-frame sanity check from node 1 to node 2 on both twins:
ping -c2 -M do -s 8972 172.31.5.2
ping -c2 -M do -s 8972 172.31.5.10
```

Expected `ip -br addr` on node 1:
```
enP7s7           UP             172.31.0.100/23 metric 100
enp1s0f0np0      UP             172.31.5.1/29
enp1s0f1np1      DOWN
enP2p1s0f0np0    UP             172.31.5.9/29
enP2p1s0f1np1    DOWN
```

Expected `ibdev2netdev`:
```
rocep1s0f0 port 1 ==> enp1s0f0np0 (Up)
rocep1s0f1 port 1 ==> enp1s0f1np1 (Down)
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Down)
```

The two RoCE devices paired with the active twins — `rocep1s0f0` and `roceP2p1s0f0` — are what NCCL should be pointed at via `NCCL_IB_HCA` (see VLLM.md).

## SSH between nodes

Following taken from: https://raw.githubusercontent.com/NVIDIA/dgx-spark-playbooks/refs/heads/main/nvidia/connect-two-sparks/assets/discover-sparks
Except not using the avahi mdns stuff, so doing it manually..

Create shared-key on the first node:
```bash
ssh-keygen -t ed25519 -N "" -f "$HOME/.ssh/id_ed25519_shared" -q -C "shared-cluster-key"
cp $HOME/.ssh/id_ed25519_shared.pub $HOME/.ssh/authorized_keys
cat $HOME/.ssh/config <<EOF
Host *
        IdentityFile ~/.ssh/id_ed25519_shared
EOF
chmod 0600 $HOME/.ssh/*
```

Copy the files to the other node:
```bash
scp -r $HOME/.ssh 172.31.5.2:~
```

You can now ssh from one node to the other without entering passwords.

Next: set up the vLLM container and Ray cluster — see [VLLM.md](VLLM.md).
