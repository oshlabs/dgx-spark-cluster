# NVIDIA DGX Spark Cluster

Documentation for setting up and running a 2-node NVIDIA DGX Spark (GB10) cluster for large-model inference.

Each node is a Grace Blackwell GB10 machine with 120 GB unified memory. The two nodes are interconnected via ConnectX-7 QSFP58 NICs and serve models tensor-parallel across both GPUs using vLLM + Ray.

## Contents

- [INSTALL.md](INSTALL.md) — system prep, networking, SSH, vLLM container, Ray head/worker setup
- [TEST-LLAMA.md](TEST-LLAMA.md) — serving Meta Llama-3.3-70B-Instruct
- [TEST-MINIMAX.md](TEST-MINIMAX.md) — serving MiniMax-M2.7 (NVFP4 quantized)

## External references

- https://github.com/NVIDIA/dgx-spark-playbooks/tree/main/nvidia/connect-two-sparks
- https://build.nvidia.com/spark/vllm/stacked-sparks
