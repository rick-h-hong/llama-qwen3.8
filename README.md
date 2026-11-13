# llama.cpp server for Qwen3.8-27B (GSQ-RCO)

Docker Compose setup that serves **Qwen3.8-27B** locally with `llama.cpp` over an OpenAI-compatible API (`http://localhost:11434`).

## Model

The served model is the **IQ3_S (MTP)** GGUF from [ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF](https://huggingface.co/ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF/tree/main):

- **File:** `Qwen3.8-27B-GSQ-RCO-IQ3_S-mtp.gguf` (~11.8 GB)
- A non-uniform GGUF quantization produced by **GSQ** + **RCO** at the [Deep Algorithms and Systems Lab, IST Austria](https://github.com/IST-DASLab): each weight tensor gets its own quantization type, assigned by a gradient-based search under a size budget (papers: [GSQ](https://arxiv.org/abs/2604.18556), [RCO](https://arxiv.org/abs/2605.00649)).
- IQ3_S is the recommended, "task-lossless" operating point: it matches the BF16 base model on AIME25 and LiveCodeBench v6 at 1/4.6th of the BF16 size.
- The `-mtp` build additionally carries the model's Multi-Token Prediction head, which the compose file enables for speculative decoding (`--spec-type draft-mtp --spec-draft-n-max 3`).

The model is licensed under Apache-2.0 (inherited from the Qwen3.8-27B base).

## Getting the model files

The container mounts `${LLAMA_MODELS:-~/llama-cpp-models}` read-only at `/models`. Download the file (plus the BF16 vision projector if you want multimodal input):

```bash
pip install -U "huggingface_hub[cli]"
hf download ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF Qwen3.8-27B-GSQ-RCO-IQ3_S-mtp.gguf --local-dir ~/llama-cpp-models
hf download ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF mmproj-Qwen3.8-27B-BF16.gguf --local-dir ~/llama-cpp-models
```

## Running

```bash
docker compose up -d
```

Notes:

- The image is `llama.cpp:full-rocm` and the compose file exposes `/dev/kfd` and `/dev/dri`, so it targets AMD GPUs via ROCm; all layers are offloaded (`--n-gpu-layers 999`).
- Context is set to 98,304 tokens with Q4 KV cache, flash attention on, and `--jinja` for the Qwen chat template.
- The model is exposed under the alias `Qwen3.8-gsq-rco`.

## Environment variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `LLAMA_MODELS` | `~/llama-cpp-models` | Host directory mounted at `/models` |
| `LLAMA_MODEL_PATH` | `/models/Qwen3.8-27B-GSQ-RCO-IQ3_S-mtp.gguf` | Model file inside the container |

## Querying

```bash
curl http://localhost:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen3.8-gsq-rco", "messages": [{"role": "user", "content": "Hello"}]}'
```
