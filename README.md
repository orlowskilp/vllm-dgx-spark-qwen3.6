# Qwen3.6 Inference Server on DGX Spark

[![vLLM v0.23](https://img.shields.io/badge/vLLM-0.23-blue)](https://vllm.ai/)
[![Qwen3.6](https://img.shields.io/badge/Qwen-3.6-purple)](https://qwen.ai/blog?id=qwen3.6-35b-a3b)
[![HF model](https://img.shields.io/badge/HuggingFace-nvidia-green)](https://huggingface.co/nvidia/Qwen3.6-35B-A3B-NVFP4)
[![MIT License](https://img.shields.io/badge/license-MIT-olive)](/LICENSE)

Local LLM inference server running Qwen3.6-35B-A3B via vLLM, deployed on an NVIDIA DGX Spark. Exposes an OpenAI-compatible API on port `8000` and is wired into Claude Code as a custom model.

This orchestration is used for paired programming with Qwen3.6 as a copilot and has worked surprisingly well for my use cases.

## Configuration

This project uses [direnv](https://direnv.net) for environment management. Once installed, it will automatically load `.envrc` when you enter the directory. Add the following to your `~/.bashrc` (or `~/.zshrc`):

```bash
eval "$(direnv hook bash)"
```

Copy `.envrc.example` to `.envrc` and fill in your Hugging Face token:

```bash
cp .envrc.example .envrc
direnv allow
```

## Running

Run vLLM in daemon mode:

```bash
docker compose up -d
```

View the logs with:

```bash
docker compose logs -f vllm
```

The API will be available at `http://<dgx-host>:8000`. If you intend to use it on a shared network, you will want to use either an SSH tunnel or an API gateway.

## Model Details

| Property               | Value                         |
| ---------------------- | ----------------------------- |
| Model                  | nvidia/Qwen3.6-35B-A3B-NVFP4  |
| Context length         | 262,144 tokens                |
| Max sequences          | 16                            |
| GPU memory utilization | 70%                           |
| Speculative decoding   | MTP (2 speculative tokens)    |
| Attention backend      | FlashInfer                    |
| vLLM version           | v0.23.0 (cu129, Ubuntu 24.04) |

### Model quantization

Switching to NVFP4 quantization on DGX Spark makes the model feel noticeably faster — output streams out quicker and the speculative decoding loop is more responsive. The quality trade-off compared to the full-precision unquantized version is small enough that the responsiveness gain is almost always worth it. If you can afford to drop the quantization (e.g. by increasing GPU memory availability), the BF16 model produces better outputs.

### Optimization opportunities

If possible, switch your DGX Spark to a non-graphical runlevel to free up GPU memory.

The maximum GPU memory utilization is set to leave headroom for load spikes. It can be raised to 80%, though OOMs are likely to occur under heavy load. The server comfortably handles 16 parallel sequences; increasing it further is possible but requires lowering GPU memory utilization to preserve KV cache. With 6 concurrent Claude Code agents, I observed prefix cache hit rates around 98% and KV cache hit rates around 60% on longer tasks.

Speculative decoding uses MTP set to 2 with triton-based MoE backend. Draft acceptance rates with MTP=2 are around 85–98% for the first token, the second lands at around 66-78%, depending on the nature of the task and number of agents.

The average token generation rate is around 55–80 tokens/s.

## Claude Code Integration

Claude Code `2.1.153` broke compatibility with vLLM instances prior to `0.23`, so this deployment targets v0.23 or later.

Add the following to your Claude Code environment (with e.g. `.envrc`) to use this server as the default model:

```bash
export ANTHROPIC_BASE_URL="http://<dgx-host>:8000"
export ANTHROPIC_AUTH_TOKEN="vllm"
export ANTHROPIC_CUSTOM_MODEL_OPTION="qwen3.6"
export ANTHROPIC_CUSTOM_MODEL_OPTION_NAME="Qwen3.6 (NVFP4)"
export ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION="Local deployment on DGX Spark"
```

### Why not LiteLLM?

Do not use LiteLLM as a proxy in front of this server. LiteLLM rewrites request headers and body payloads (adding its own tracing fields, modifying the prompt structure, etc.), which changes the request fingerprint that vLLM uses for prefix caching. Prefix caching relies on exact byte-for-byte matching of the prompt tokens — any proxy that mutates the request invalidates the cache, reducing prefix cache hit rates from ~98% to near 0% and destroying one of vLLM's strongest performance optimizations. Always connect directly to the vLLM instance.

---

Copyright (c) 2026 Lukasz P. Orlowski <lukasz@orlowski.io>
All rights granted under [MIT License](LICENSE)
