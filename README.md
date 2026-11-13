# llama.cpp: Qwen3.8-27B (GSQ-RCO) + Qwen3.5-9B on AMD ROCm

A single [llama.cpp](https://github.com/ggml-org/llama.cpp) server in **router (multi-model) mode** that serves two models over one OpenAI-compatible API:

| Request model ID | Model | GGUF | Size | Role |
| --- | --- | --- | --- | --- |
| `qwen3.8` (alias `Qwen3.8-gsq-rco`) | [Qwen3.8-27B, GSQ+RCO IQ3_S, with MTP head](https://huggingface.co/ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF) | `Qwen3.8-27B-GSQ-RCO-IQ3_S-mtp.gguf` | 11.8 GB | main coding model — 98k context, MTP speculative decoding |
| `qwen3.5` (alias `Qwen3.5`) | [Qwen3.5-9B, Q8_0](https://huggingface.co/unsloth/Qwen3.5-9B-GGUF) | `Qwen3.5-9B-Q8_0.gguf` | 9.5 GB | historian for opencode's magic-context — 98k context |

Each model fits in the GPU VRAM on its own, but not both at the same time. The stack is
configured so only one model is resident at a time: `--models-max 1` makes the router
unload the idle model and load whichever one a request asks for, automatically. opencode
talks to the same endpoint for both, just using a different `model` field
(`qwen3.8` vs `qwen3.5`).

## Model notes

### Qwen3.8-27B (GSQ+RCO IQ3_S)

Non-uniform GSQ+RCO quantization from ISTA Austria's [DASLab](https://huggingface.co/ISTA-DASLab) —
11.8 GB, with the paper's task-lossless operating point at IQ3_S. The `-mtp` build also
carries the model's Multi-Token Prediction head (15 tensors, ~0.3% of the file); `spec-type
draft-mtp` enables speculative decoding from it.

### Qwen3.5-9B (Q8_0)

[unsloth](https://huggingface.co/unsloth/Qwen3.5-9B-GGUF)'s near-lossless 8-bit quantization,
9.5 GB. Used as the **historian** model for opencode's magic-context.

## Download

```sh
mkdir -p ~/llama-cpp-models

# main coding model (ISTA Austria DASLab GSQ+RCO, MTP build)
hf download ISTA-DASLab/Qwen3.8-27B-GSQ-RCO-GGUF Qwen3.8-27B-GSQ-RCO-IQ3_S-mtp.gguf \
    --local-dir ~/llama-cpp-models

# historian model (unsloth Q8_0)
hf download unsloth/Qwen3.5-9B-GGUF Qwen3.5-9B-Q8_0.gguf \
    --local-dir ~/llama-cpp-models
```

## How the model swap works

- **Router mode.** `llama-server` runs as a router that spawns a child server per model,
  driven by the preset file `models.ini` (embedded in [`docker-compose.yml`](docker-compose.yml)
  as a named config and mounted at `/run/configs/models.ini`). Each `[section]` names one
  model; the section name — plus any `alias =` lines — are the IDs you use in API requests.
- **`--models-max 1`.** At most one model is ever loaded. When a request comes in for the
  other model, the router queues it, unloads the idle (least-recently-used) model once its
  in-flight requests finish, and loads the requested one.
- **`load-on-startup = false`** (both models). Nothing sits in VRAM at boot; the first
  request to a model pays a one-off load cost.

Optional knobs, if you want them (add to the `command:` list):

- `--sleep-idle-seconds N` — offload the resident model entirely after N idle seconds, so
  an idle server holds ~0 VRAM (the next request reloads it).
- `--models-memory-margin N` — keep N MiB free per device when loading; the router unloads
  LRU models until the new one fits.

## Running

```sh
export LLAMA_MODELS=~/llama-cpp-models   # where the GGUF files live
docker compose up -d
```

The compose file assumes a ROCm/AMD GPU: the `full-rocm` image, `/dev/kfd` and `/dev/dri`
device passes, and `n-gpu-layers = 999` (all layers on the GPU). If model + KV cache don't
fit, lower the pinned `c =` value in `models.ini` — or drop it entirely and let the server
auto-fit the largest context that fits.

## Using the API

```sh
curl http://localhost:11434/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3.8",
    "messages": [
      {"role": "user", "content": "Hello!"}
    ]
  }'
```

- List models and their load state: `curl http://localhost:11434/v1/models`
- Force a swap on demand:
  ```sh
  curl -X POST http://localhost:11434/models/load \
    -H 'Content-Type: application/json' -d '{"model": "qwen3.5"}'

  curl -X POST http://localhost:11434/models/unload \
    -H 'Content-Type: application/json' -d '{"model": "qwen3.8"}'
  ```

## Environment

| Variable | Default | Purpose |
| --- | --- | --- |
| `LLAMA_MODELS` | `~/llama-cpp-models` | Host directory with the GGUF files, mounted read-only at `/models` |
