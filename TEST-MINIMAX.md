# Test — MiniMax-M2.7 (NVFP4 quantized)

Prerequisite: cluster is up and `hf auth login` completed — see [VLLM.md](VLLM.md).

## Download the model

```bash
hf download lukealonso/MiniMax-M2.7-NVFP4P4
```

## Serve the model

```bash
docker exec -it $VLLM_CONTAINER bash -c '
  export VLLM_NVFP4_GEMM_BACKEND=flashinfer-cutlass
  export VLLM_USE_FLASHINFER_MOE_FP4=1
  export VLLM_ALLOW_LONG_MAX_MODEL_LEN=1
  export VLLM_FLASHINFER_MOE_BACKEND=throughput
  export VLLM_FLOAT32_MATMUL_PRECISION=high
  export VLLM_MEMORY_PROFILER_ESTIMATE_CUDAGRAPHS=1
  export OMP_NUM_THREADS=8

  vllm serve lukealonso/MiniMax-M2.7-NVFP4 \
    --trust-remote-code \
    --tensor-parallel-size 2 \
    --distributed-executor-backend ray \
    --dtype auto \
    --quantization modelopt_fp4 \
    --kv-cache-dtype fp8 \
    --attention-backend flashinfer \
    --mamba_ssm_cache_dtype float32 \
    --disable-custom-all-reduce \
    --max-num-batched-tokens 8192 \
    --gpu-memory-utilization 0.80 \
    --max-model-len 32768 \
    --max-num-seqs 5 \
    --enable-auto-tool-choice \
    --tool-call-parser minimax_m2 \
    --reasoning-parser minimax_m2_append_think \
    --served-model-name MiniMax-M2.7 \
    --host 0.0.0.0 --port 8000
'
```

## Query it

```
curl http://172.31.0.100:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M2.7",
    "messages": [
      {"role": "user", "content": "Write a haiku about a GPU"}
    ],
    "max_tokens": 128,
    "temperature": 0.7
  }'
```
