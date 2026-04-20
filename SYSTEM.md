# System setup — DGX Spark cluster

```
                172.31.0.0/23  (10GbE management LAN)
    ═══════════╤═════════════════════════════════════╤═══════════
               │                                     │
               │ 172.31.0.100 (enP7s7)               │ 172.31.0.101 (enP7s7)
       ┌───────┴───────┐                     ┌───────┴───────┐
       │   spark-702b  │                     │   spark-20db  │
       │               │                     │               │
       │  enp1s0f0np0  │                     │  enp1s0f0np0  │
       │  172.31.5.3   │    172.31.5.0/28    │  172.31.5.4   │
       │               ├═════════════════════┤               │ 
       │ enP2p1s0f0np0 │      QSFP56 DAC     │ enP2p1s0f0np0 │
       │ (no ip addr)  │                     │ (no ip addr)  │
       └───────────────┘                     └───────────────┘
```

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

## Fix networking

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

# use 172.31.5.3 on the first node, 172.31.5.4 on the next, etc
sudo tee /etc/netplan/40-cx7.yaml <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    enp1s0f0np0:
      addresses: [172.31.5.3/28]
      mtu: 9216
      dhcp4: false
      dhcp6: false
      link-local: []
    enP2p1s0f0np0:
      mtu: 9216
      dhcp4: false
      dhcp6: false
      link-local: []
EOF

sudo chmod 0600 /etc/netplan/*.yaml
sudo netplan generate
sudo netplan apply
```

This results in following network configuration:
```
# the 10gbe interface:

2: enP7s7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:81:70:2b brd ff:ff:ff:ff:ff:ff
    altname enP7p1s0
    inet 172.31.0.100/23 metric 100 brd 172.31.1.255 scope global dynamic enP7s7
       valid_lft 3085sec preferred_lft 3085sec

# the two qsfp58 cx7 nics:

3: enp1s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9216 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:81:70:2c brd ff:ff:ff:ff:ff:ff
    inet 172.31.5.3/28 brd 172.31.5.15 scope global enp1s0f0np0
       valid_lft forever preferred_lft forever
4: enp1s0f1np1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 4c:bb:47:81:70:2d brd ff:ff:ff:ff:ff:ff
5: enP2p1s0f0np0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9216 qdisc mq state UP group default qlen 1000
    link/ether 4c:bb:47:81:70:30 brd ff:ff:ff:ff:ff:ff
6: enP2p1s0f1np1: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether 4c:bb:47:81:70:31 brd ff:ff:ff:ff:ff:ff
```

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
scp -r $HOME/.ssh 172.31.5.4:~
```

You can now ssh from one node to the other without entering passwords.

Next: set up the vLLM container and Ray cluster — see [VLLM.md](VLLM.md).
