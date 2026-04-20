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

## Start the Ray cluster

Create `~/start-ray.sh` on **both** nodes. The script auto-detects its role: if `HEAD_IP` is bound to any local interface it runs as head, otherwise as worker.

```bash
tee ~/start-ray.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

HEAD_IP=172.31.5.1
MN_IF_NAME=enp1s0f0np0
VLLM_IMAGE=nvcr.io/nvidia/vllm:26.03.post1-py3

VLLM_HOST_IP=$(ip -4 addr show "$MN_IF_NAME" | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

if ip -4 addr show | grep -q "inet $HEAD_IP/"; then
    ROLE=head
else
    ROLE=worker
fi

if [[ -z "${RAY_INNER:-}" ]]; then
    echo "Detected role: $ROLE (this node $VLLM_HOST_IP, head $HEAD_IP)"
    exec env RAY_INNER=1 screen -S "ray-$ROLE" bash "$0"
fi

exec bash ~/run_cluster.sh "$VLLM_IMAGE" "$HEAD_IP" "--$ROLE" ~/.cache/huggingface \
  --device=/dev/infiniband \
  --cap-add=IPC_LOCK \
  --ulimit memlock=-1:-1 \
  -e NCCL_IB_DISABLE=0 \
  -e NCCL_IB_HCA=rocep1s0f0,roceP2p1s0f0 \
  -e NCCL_IB_GID_INDEX=3 \
  -e VLLM_HOST_IP="$VLLM_HOST_IP" \
  -e UCX_NET_DEVICES="$MN_IF_NAME" \
  -e NCCL_SOCKET_IFNAME="$MN_IF_NAME" \
  -e OMPI_MCA_btl_tcp_if_include="$MN_IF_NAME" \
  -e GLOO_SOCKET_IFNAME="$MN_IF_NAME" \
  -e TP_SOCKET_IFNAME="$MN_IF_NAME" \
  -e RAY_memory_monitor_refresh_ms=0 \
  -e MASTER_ADDR="$HEAD_IP"
EOF
chmod +x ~/start-ray.sh
```

Run on both nodes — no arguments needed:
```bash
~/start-ray.sh
```

The script re-execs itself inside a `screen` session named `ray-head` or `ray-worker` and attaches to it, so you see live container logs. Detach with **Ctrl+A D** (screen keeps running); re-attach later with `screen -r ray-head` or `screen -r ray-worker`.

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

## Verify NCCL / RDMA (optional but recommended)

Confirms NCCL actually uses RDMA over both RoCE twins — not TCP fallback. Cheap to run, easy to interpret. Skip during normal operation; re-run after cable swaps, firmware updates, or any env/flag changes that might perturb the transport.

The GPUs must be free, so this has to happen **before** `vllm serve` is launched (or after stopping it). The Ray containers can stay up.

Create the benchmark script on both nodes. We put it in `~/.cache/huggingface/` because that directory is bind-mounted into the container as `/root/.cache/huggingface/` (set up by `run_cluster.sh`), so the script is reachable inside the container without adding another volume mount:
```bash
tee ~/.cache/huggingface/nccl_bench.py <<'EOF'
#!/usr/bin/env python3
import os
import torch
import torch.distributed as dist


def main():
    rank = int(os.environ["RANK"])
    world_size = int(os.environ["WORLD_SIZE"])
    local_rank = int(os.environ.get("LOCAL_RANK", 0))

    torch.cuda.set_device(local_rank)
    dist.init_process_group(backend="nccl")

    if rank == 0:
        print(f"world_size={world_size}, backend=nccl, device=cuda:{local_rank}")
        print(f"{'size(MB)':>10} {'time(ms)':>10} {'algbw(GB/s)':>12} {'busbw(GB/s)':>12}")

    sizes_mb = [8, 32, 64, 128, 256, 512, 1024, 2048]
    warmup = 5
    iters = 20

    for size_mb in sizes_mb:
        nbytes = size_mb * 1024 * 1024
        nfloats = nbytes // 4
        x = torch.ones(nfloats, dtype=torch.float32, device="cuda")

        for _ in range(warmup):
            dist.all_reduce(x)
        torch.cuda.synchronize()

        start = torch.cuda.Event(enable_timing=True)
        end = torch.cuda.Event(enable_timing=True)
        start.record()
        for _ in range(iters):
            dist.all_reduce(x)
        end.record()
        torch.cuda.synchronize()

        ms = start.elapsed_time(end) / iters
        algbw = nbytes / (ms / 1000) / 1e9
        busbw = algbw * 2 * (world_size - 1) / world_size

        if rank == 0:
            print(f"{size_mb:>10} {ms:>10.3f} {algbw:>12.2f} {busbw:>12.2f}")

    dist.destroy_process_group()


if __name__ == "__main__":
    main()
EOF
```

Grab the container name on each node:
```bash
export VLLM_CONTAINER=$(docker ps --format '{{.Names}}' | grep -E '^node-[0-9]+$')
```

Launch — **worker first**, so the rendezvous on node 1 has something to connect to. Each command is a single line; don't split with `\` (trailing-space hazard on paste).

On node 2 (worker):
```bash
docker exec -e NCCL_DEBUG=INFO $VLLM_CONTAINER torchrun --nnodes=2 --nproc_per_node=1 --node_rank=1 --master_addr=172.31.5.1 --master_port=29500 /root/.cache/huggingface/nccl_bench.py 2>&1 | tee /tmp/nccl_bench.log
```

On node 1 (head):
```bash
docker exec -e NCCL_DEBUG=INFO $VLLM_CONTAINER torchrun --nnodes=2 --nproc_per_node=1 --node_rank=0 --master_addr=172.31.5.1 --master_port=29500 /root/.cache/huggingface/nccl_bench.py 2>&1 | tee /tmp/nccl_bench.log
```

### What a healthy run looks like

Bandwidth table (rank 0 only, printed on node 1):
```
  size(MB)   time(ms)  algbw(GB/s)  busbw(GB/s)
         8      0.425        19.74        19.74
        32      1.657        20.25        20.25
       128      7.194        18.66        18.66
       512     26.552        20.22        20.22
      1024     48.494        22.14        22.14
      2048     90.790        23.65        23.65
```

Asymptotic ~23–24 GB/s at large sizes is the target. QSFP56 is ~200 Gbps physical (25 GB/s); 23.65 GB/s ≈ 189 Gbps ≈ 95% of line-rate, which is as good as this hardware gets.

Key NCCL log lines (in `/tmp/nccl_bench.log`):
```
NET/IB : Using [0]rocep1s0f0:1/RoCE [1]roceP2p1s0f0:1/RoCE [RO]
Channel 00/0 : 0[0] -> 1[0] [send] via NET/IBext_v11/0
Channel 01/0 : 0[0] -> 1[0] [send] via NET/IBext_v11/1
Channel 02/0 : 0[0] -> 1[0] [send] via NET/IBext_v11/0
Channel 03/0 : 0[0] -> 1[0] [send] via NET/IBext_v11/1
```
- `NET/IB` (not `NET/Socket`) → RDMA is active
- Channels alternating `/0` and `/1` → both RoCE twins carry traffic

### Why ~24 GB/s is the ceiling — and why we bother with both twins anyway

The DGX Spark's dual-port ConnectX-7 exposes four net devices, but only one QSFP56 cable is physically connected. That cable carries both internal PCIe x4 links, so aggregate RDMA throughput is bounded by the cable's ~200 Gbps regardless of how many twins NCCL uses.

So why set `NCCL_IB_HCA=rocep1s0f0,roceP2p1s0f0` at all? Each PCIe x4 link is roughly 100 Gbps — using only one twin caps you at ~12.5 GB/s. Using both parallelizes the PCIe path so the cable can be filled. Skipping this env would roughly halve your bandwidth.

If you see ~12 GB/s asymptotic instead of ~24, NCCL is likely using only one twin — re-check `NCCL_IB_HCA` and the channel-alternation pattern in the log. If you see single-digit GB/s, NCCL fell back to sockets — re-check `NCCL_IB_DISABLE`, `--device=/dev/infiniband`, `--cap-add=IPC_LOCK`, and the `--ulimit memlock` setting.

## Huggingface login

Get a token after logging in to https://huggingface.co
(top-right -> Access Tokens -> Create new Token -> read -> token name 'spark' -> Create token)

Now login:
```bash
hf auth login
```

The cluster is now ready to serve models. See [TEST-LLAMA.md](TEST-LLAMA.md) and [TEST-MINIMAX.md](TEST-MINIMAX.md).
