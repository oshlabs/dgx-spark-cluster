# vLLM container + Ray cluster

Prerequisite: system setup done on both nodes — see [SYSTEM.md](SYSTEM.md).

External reference: https://build.nvidia.com/spark/vllm/stacked-sparks

## Install Huggingface cli

```bash
sudo apt install pipx
```

Install huggingface client:
```bash
pipx install "huggingface_hub[cli]"
```

(exit and re-ssh into the machine so-that ~/.local/bin is in your $PATH)

## Download the run_cluster helper and vLLM image

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

## Start Ray head node

Following steps in a screen on the first spark (in my case spark-702b / 172.31.0.100 / 172.31.5.3):
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

## Start Ray worker node

Following steps in a screen on the second spark (in my case spark-20db / 172.31.0.101 / 172.31.5.4).
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

## Verify the Ray cluster

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
