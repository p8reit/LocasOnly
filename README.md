# LocasOnly

LocasOnly is a self-hosted, Docker-based local AI platform designed to keep LLM inference on hardware you control.

The initial stack uses mature open-source projects instead of custom infrastructure:

- **Ollama** — local model runtime and API
- **Open WebUI** — browser-based chat, conversation history, knowledge/RAG, and tool integration
- **SearXNG** — self-hosted web search for current information

The long-term goal is for other local applications such as Patch Notes and TheHunter to consume the same Ollama service over a shared Docker network.

## Design principles

1. LLM inference stays local.
2. No OpenAI, Anthropic, Gemini, Groq, or other cloud LLM fallback.
3. Prefer mature open-source/self-hosted projects over custom code.
4. Dockerize shared services.
5. Keep the model runtime independent of the chat UI.
6. Internet access is treated as a tool/data source, not as an AI provider.
7. Other applications should be able to consume the same local model API.

## Architecture

```text
                        WSL / Docker Host
                              |
                         RTX 3060 GPU
                              |
                              v
                         +---------+
                         | Ollama  |
                         +----+----+
                              |
                       locasonly-ai network
                  +-----------+-----------+
                  |                       |
                  v                       v
             +-----------+           +---------+
             | Open WebUI|           | SearXNG |
             +-----------+           +---------+
                  |
                  +---- Human chat / RAG / tools

Future clients:
  Patch Notes ----+
  TheHunter ------+----> Ollama API
  Coding agents --+
```

## Prerequisites

This project is intended for Linux/WSL2 with Docker and NVIDIA GPU support.

Before starting LocasOnly, verify the GPU is visible inside WSL:

```bash
nvidia-smi
```

Then verify Docker can access it:

```bash
docker run --rm --gpus all \
  nvidia/cuda:12.9.0-base-ubuntu22.04 \
  nvidia-smi
```

Do not continue with a GPU-backed model deployment until both commands show the NVIDIA GPU correctly.

## Initial setup

Clone the repository:

```bash
git clone git@github.com:p8reit/LocasOnly.git
cd LocasOnly
```

Create the local environment file:

```bash
cp .env.example .env
```

Generate a WebUI secret:

```bash
openssl rand -hex 32
```

Put that value into `.env` as `WEBUI_SECRET_KEY`.

Also replace the placeholder SearXNG secret in:

```text
searxng/settings.yml
```

A second random value can be generated with:

```bash
openssl rand -hex 32
```

## Start the stack

```bash
docker compose pull
docker compose up -d
```

Check status:

```bash
docker compose ps
```

Follow logs:

```bash
docker compose logs -f
```

Open the chat UI:

```text
http://localhost:3000
```

The first Open WebUI account created becomes the administrator.

## Pull the first model

Start with the general-purpose local model:

```bash
docker exec -it locasonly-ollama ollama pull qwen3:14b
```

Confirm the model is installed:

```bash
docker exec -it locasonly-ollama ollama list
```

You can also test Ollama directly:

```bash
docker exec -it locasonly-ollama ollama run qwen3:14b
```

## Open WebUI configuration

Ollama is already wired internally through:

```text
http://ollama:11434
```

Cloud OpenAI-compatible providers are explicitly disabled in the Compose configuration:

```text
ENABLE_OPENAI_API=false
```

This is intentional.

### SearXNG web search

After signing in as the Open WebUI administrator, configure Web Search to use the internal SearXNG service:

```text
http://searxng:8080/search?q=<query>
```

Use SearXNG as the search engine/provider and keep inference on Ollama.

Searching the public web necessarily sends search requests to public search/web services, but no external LLM is required for processing the results.

## Shared Docker network

The stack creates a named Docker network:

```text
locasonly-ai
```

Other Compose projects can attach to it:

```yaml
services:
  my-app:
    networks:
      - app-network
      - locasonly-ai

networks:
  app-network:

  locasonly-ai:
    external: true
```

From that application container, Ollama is reachable at:

```text
http://locasonly-ollama:11434
```

This allows Patch Notes, TheHunter, coding agents, and future applications to share one local inference service.

## Basic Ollama API test from another container

```bash
curl http://locasonly-ollama:11434/api/chat \
  -d '{
    "model": "qwen3:14b",
    "messages": [
      {
        "role": "user",
        "content": "Explain what service you are running on."
      }
    ],
    "stream": false
  }'
```

## GPU usage

Watch GPU utilization from WSL:

```bash
watch -n 1 nvidia-smi
```

While a prompt is being processed, the Ollama process should consume GPU memory and show GPU activity.

## Useful commands

Start:

```bash
docker compose up -d
```

Stop:

```bash
docker compose down
```

Restart:

```bash
docker compose restart
```

Update images:

```bash
docker compose pull
docker compose up -d
```

View Ollama logs:

```bash
docker compose logs -f ollama
```

View Open WebUI logs:

```bash
docker compose logs -f open-webui
```

View SearXNG logs:

```bash
docker compose logs -f searxng
```

## Planned next steps

- Validate RTX 3060 GPU pass-through under WSL/Docker.
- Validate Qwen3 14B performance and VRAM usage.
- Finish SearXNG integration in Open WebUI.
- Add a dedicated local coding model after the base platform is stable.
- Connect Patch Notes to the shared `locasonly-ai` Docker network.
- Connect TheHunter to the same local Ollama API.
- Evaluate an open-source coding agent such as OpenCode.
- Add shared RAG/vector infrastructure only if application requirements justify it.

## Security notes

The initial configuration exposes only Open WebUI to the host.

Ollama and SearXNG are available to containers on the `locasonly-ai` network but are not published directly to the host.

Do not expose Ollama or SearXNG directly to the public Internet.

## License

No project license has been selected yet.
