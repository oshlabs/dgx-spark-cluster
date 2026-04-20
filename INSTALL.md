# Install procedure — DGX Spark cluster

External references:
* https://github.com/NVIDIA/dgx-spark-playbooks/tree/main/nvidia/connect-two-sparks
* https://build.nvidia.com/spark/vllm/stacked-sparks

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

## Install Huggingface cli

```bash
sudo apt install pipx
```

Install huggingface client:
```bash
pipx install "huggingface_hub[cli]"
```

(exit and re-ssh into the machine so-that ~/.local/bin is in your $PATH)

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

You can now ssh from one node to the other without entering passwords

## vLLM container + Ray cluster

Further following https://build.nvidia.com/spark/vllm/stacked-sparks

```bash
# Download on both nodes
wget https://raw.githubusercontent.com/vllm-project/vllm/refs/heads/main/examples/online_serving/run_cluster.sh
chmod +x run_cluster.sh
```

The doc advises to use vllm 25.09-py3, but seeing here: https://catalog.ngc.nvidia.com/orgs/nvidia/containers/vllm/tags that version 26.03.post1-py3 is a lot newer (15th. Apr 2026 instead of 1st. Oct 2025)

Trying that -- run on both hosts:
```bash
docker pull nvcr.io/nvidia/vllm:26.03.post1-py3
```

And following steps in a screen to start Ray head node (on the first spark - in my case spark-702b / 172.31.0.100 / 172.31.5.3):
```bash
screen

export VLLM_IMAGE=nvcr.io/nvidia/vllm:26.03.post1-py3
export MN_IF_NAME=enp1s0f0np0
export VLLM_HOST_IP=$(ip -4 addr show $MN_IF_NAME | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

echo "Using interface $MN_IF_NAME with IP $VLLM_HOST_IP"

bash run_cluster.sh $VLLM_IMAGE $VLLM_HOST_IP --head ~/.cache/huggingface \
  -e VLLM_HOST_IP=$VLLM_HOST_IP \
  -e UCX_NET_DEVICES=$MN_IF_NAME \
  -e NCCL_SOCKET_IFNAME=$MN_IF_NAME \
  -e OMPI_MCA_btl_tcp_if_include=$MN_IF_NAME \
  -e GLOO_SOCKET_IFNAME=$MN_IF_NAME \
  -e TP_SOCKET_IFNAME=$MN_IF_NAME \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=$VLLM_HOST_IP
```

And following steps in a screen to start the Ray worker node (on the second spark - in my case spark-20db / 172.31.0.101 / 172.31.5.4):
Make sure to set the ip of the Ray head node (node 1):
```bash
screen

export HEAD_NODE_IP=172.31.5.3

export VLLM_IMAGE=nvcr.io/nvidia/vllm:26.03.post1-py3
export MN_IF_NAME=enp1s0f0np0
export VLLM_HOST_IP=$(ip -4 addr show $MN_IF_NAME | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

echo "Worker IP: $VLLM_HOST_IP, connecting to head node at: $HEAD_NODE_IP"

bash run_cluster.sh $VLLM_IMAGE $HEAD_NODE_IP --worker ~/.cache/huggingface \
  -e VLLM_HOST_IP=$VLLM_HOST_IP \
  -e UCX_NET_DEVICES=$MN_IF_NAME \
  -e NCCL_SOCKET_IFNAME=$MN_IF_NAME \
  -e OMPI_MCA_btl_tcp_if_include=$MN_IF_NAME \
  -e GLOO_SOCKET_IFNAME=$MN_IF_NAME \
  -e TP_SOCKET_IFNAME=$MN_IF_NAME \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR=$HEAD_NODE_IP
```

In another cli on the primary (ray head) node:
```bash
export VLLM_CONTAINER=$(docker ps --format '{{.Names}}' | grep -E '^node-[0-9]+$')
echo "Found container: $VLLM_CONTAINER"
docker exec $VLLM_CONTAINER ray status
```

Should show something like:
```bash
======== Autoscaler status: 2026-04-19 17:08:02.711002 ========
Node status
---------------------------------------------------------------
Active:
 1 node_83e4b7723832f251eff177c9faf45a5305de7b1755a7f91ce1f3c911
 1 node_4dcd7509dc73bd2e758e4048d40233b19eb019dfb54c66095a01e30f
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Total Usage:
 0.0/40.0 CPU
 0.0/2.0 GPU
 0B/221.76GiB memory
 0B/19.46GiB object_store_memory

From request_resources:
 (none)
Pending Demands:
 (no resource demands)
```

## Huggingface login

Get a token after logging in to https://huggingface.co
(top-right -> Access Tokens -> Create new Token -> read -> token name 'spark' -> Create token)

Now login:
```bash
hf auth login
```

The cluster is now ready to serve models. See [TEST-LLAMA.md](TEST-LLAMA.md) and [TEST-MINIMAX.md](TEST-MINIMAX.md).
