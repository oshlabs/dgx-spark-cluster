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
    --max-num-seqs 32 \
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

## Benchmark — throughput vs concurrency

Numbers on the 2-node GB10 cluster with the RDMA config from [VLLM.md](VLLM.md). Decode-only measurement: prompt is short (~50 tokens) so prefill is negligible; completion is fixed at 500 tokens so all the work is decode.

### Method

Run N concurrent identical requests in the background and let `wait` block until all finish. The engine log's `Avg generation throughput: X tokens/s` line during steady-state (when `Running: N reqs`) is the aggregate decode throughput.

```bash
for i in $(seq 1 N); do (curl -s http://172.31.0.100:8000/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"MiniMax-M2.7","messages":[{"role":"user","content":"Count from 1 to 500 as a comma-separated list."}],"max_tokens":500,"temperature":0,"stream":false}' | jq '.usage') & done; wait
```

Watch the engine logs on the head node (`screen -r ray-head`) during the run for the `Running: N reqs` and `Avg generation throughput` lines. `Temperature: 0` and identical prompts give stable output lengths and high prefix-cache hit rates, which is fine for measuring the decode curve but means these numbers are an **upper bound** for real workloads (unique prompts force real prefill work and reduce aggregate decode throughput).

### Results

| concurrent streams | aggregate tok/s | per-stream tok/s |
|---|---|---|
| 1  |  23 | 23  |
| 2  |  40 | 20  |
| 4  |  64 | 16  |
| 8  |  94 | 12  |
| 16 | 146 | 9   |
| 32 | 210 | 6.6 |

Scaling is still positive at 32 but clearly tapering. KV cache usage stays under 3% even at 32 concurrent, so memory isn't the limit — compute / expert-routing is.

### Practical operating points

- **Interactive chat**: 8-16 concurrent users gives each ~9-12 tok/s — comfortable reading speed.
- **Throughput-max batch workloads**: 32 concurrent pushes aggregate to ~210 tok/s if per-user latency doesn't matter.
- Below ~8 tok/s per stream the experience feels noticeably slow for interactive use.

`--max-num-seqs 32` in the serve command above is chosen to allow the full throughput curve without queueing.
