# Qwen3.6 Inference Server on DGX Spark

[![MIT License](https://img.shields.io/badge/license-MIT-green)](/LICENSE)

Local LLM inference server running Qwen3.6-35B-A3B via vLLM, deployed on an NVIDIA DGX Spark. Exposes an OpenAI-compatible API on port 8000 and is wired into Claude Code as a custom model.

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

| Property               | Value                     |
| ---------------------- | ------------------------- |
| Model                  | Qwen/Qwen3.6-35B-A3B      |
| Context length         | 262,144 tokens            |
| Max sequences          | 8                         |
| GPU memory utilization | 87%                       |
| Speculative decoding   | MTP (1 speculative token) |
| Attention backend      | FlashInfer                |

### Optimization opportunities

If possible, switch your DGX Spark to a non-graphical runlevel to free up GPU memory.

The maximum GPU memory utilization is set to leave headroom for load spikes. It can be raised to 90%, though OOMs may occur under heavy load. The server comfortably handles 8 parallel sequences; increasing it to 32 is possible but requires lowering GPU memory utilization to preserve KV cache. With 6 concurrent Claude Code agents, I observed prefix cache hit rates around 98% and KV cache hit rates around 60% on longer tasks.

Speculative decoding uses MTP set to 1. Increasing it to 2 drops the draft acceptance rate to around 60%, which negates the benefit. With MTP=1, draft acceptance rates are 85–98%.

The average token generation rate is around 55–80 tokens/s.

## Claude Code Integration

Add the following to your Claude Code environment to use this server as the default model:

```bash
export ANTHROPIC_BASE_URL="http://<dgx-host>:8000"
unset ANTHROPIC_API_KEY
export ANTHROPIC_CUSTOM_MODEL_OPTION="qwen3.6"
export ANTHROPIC_CUSTOM_MODEL_OPTION_NAME="Qwen3.6"
export ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION="Local deployment on DGX Spark"
```

---

Copyright (c) 2026 Lukasz P. Orlowski <lukasz@orlowski.io>
All rights granted under [MIT License](LICENSE)
