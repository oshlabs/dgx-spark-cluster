# Test — Meta Llama-3.3-70B-Instruct

Prerequisite: cluster is up and `hf auth login` completed — see [VLLM.md](VLLM.md).

## Unlock the model

Unlock the llama-3.3-70b-instruct model:
* Go to: https://huggingface.co/meta-llama/Llama-3.3-70B-Instruct
* Fill in all your details to request access
* Check here to see if access was approved: https://huggingface.co/settings/gated-repos

## Download the model

```bash
hf download meta-llama/Llama-3.3-70B-Instruct
```

## Serve the model

Start the model from the ray head:
```bash
screen

export VLLM_CONTAINER=$(docker ps --format '{{.Names}}' | grep -E '^node-[0-9]+$')
docker exec -it $VLLM_CONTAINER /bin/bash -c '
  vllm serve meta-llama/Llama-3.3-70B-Instruct \
    --tensor-parallel-size 2 --max_model_len 2048'
```

## Query it

```
curl http://172.31.0.100:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.3-70B-Instruct",
    "prompt": "Write a haiku about a GPU",
    "max_tokens": 32,
    "temperature": 0.7
  }'
```

Well.. *that was slow* ..
