# Local OpenCode agent (optional)

This overlay adds **OpenCode**, using only the existing Ollama server. The requested models are `qwen3.5:9b` (planning) and `qwen2.5-coder:7b` (building). No cloud LLM provider is configured.

## Before deploying

- Start the existing LocasOnly stack; verify NVIDIA passthrough.
- The agent has **no internet access** through its private, internal-only Docker network. Building the image and pulling Ollama models still requires internet access.
- Only `./workspaces` is mounted as a writable source-code directory. Do not put production secrets there.
- A model's ability to call tools is not guaranteed; validate the sandbox tests before trusting changes.
- OpenCode generally recommends much larger context windows than the 16K VRAM-conscious setting used here. With a 12 GB RTX 3060, large repository tasks may fail or perform poorly.
- This overlay is designed to be applied alongside `docker-compose.yml`; use both files for agent-related commands.

## Deploy

From the repository root:

```bash
cp .env.example .env  # only if you have not already created .env
mkdir -p workspaces/sandbox-project
docker compose -f docker-compose.yml -f docker-compose.agents.yml config --quiet
docker compose up -d  # if the existing stack has not been started
docker compose -f docker-compose.yml -f docker-compose.agents.yml build opencode
docker compose -f docker-compose.yml -f docker-compose.agents.yml up -d ollama opencode
docker exec locasonly-ollama ollama pull qwen3.5:9b
docker exec locasonly-ollama ollama pull qwen2.5-coder:7b
docker exec locasonly-ollama ollama list
```

The `up -d ollama opencode` command may recreate Ollama to apply environment/network changes. Open WebUI can temporarily lose access to inference while this happens.

## Launch (interactive)

```bash
docker compose -f docker-compose.yml -f docker-compose.agents.yml \
  exec -it -w /workspace/sandbox-project opencode opencode
```

Change `sandbox-project` to a repository placed within `workspaces/`. Choose **plan** (Qwen 3.5) to inspect a task, then **build** (Qwen 2.5 Coder) to edit and test with approval.

## Smoke tests

```bash
# Model API is reachable from private agent network
docker compose -f docker-compose.yml -f docker-compose.agents.yml \
  exec opencode curl -fsS http://ollama:11434/api/tags

# Agent executable installed
docker compose -f docker-compose.yml -f docker-compose.agents.yml \
  exec opencode opencode --version

# No Docker socket should be mounted
docker compose -f docker-compose.yml -f docker-compose.agents.yml \
  exec opencode sh -c 'test ! -S /var/run/docker.sock'

# The container should have no route to public HTTPS
docker compose -f docker-compose.yml -f docker-compose.agents.yml \
  exec opencode sh -c 'if curl -fsS --connect-timeout 3 https://example.com; then exit 1; else echo "Agent egress blocked"; fi'

# Inspect active model placement while a request is running
docker exec locasonly-ollama ollama ps
```

Create a simple Python file in `workspaces/sandbox-project`, then ask the build agent to add `unittest` coverage and run tests. Confirm each write/shell approval and inspect `git diff` yourself.

## Security boundaries

- No exposed OpenCode port, Docker socket, privileged container, host home, or SSH mounts.
- Runtime outbound network blocked for OpenCode; Ollama remains reachable over `agent-internal` while Open WebUI and SearXNG retain `locasonly-ai`.
- The container is non-root with dropped Linux capabilities, a read-only root filesystem, and writable workspace/cache volumes.
- OpenCode prompts for edits and most commands; **permission prompts are not a sandbox**. Treat third-party repositories, shell commands, and their output as untrusted.
- Files in `workspaces` may be deleted or exfiltrated within that volume by approved commands. Use disposable Git clones or worktrees. Keep secrets out.
- Do not expose Ollama or OpenCode to the public internet.
- Only model **inference** is local: SearXNG in the original stack intentionally searches external websites.

## Troubleshooting

If the build fails, check availability of the pinned npm package version and the agent image build logs. If the TUI claims the context is too small, try an 8K task first or increase Ollama/model context limits cautiously. If the GPU is out of memory, reduce context or close other GPU workloads. Models load one at a time (`OLLAMA_MAX_LOADED_MODELS=1`), not in parallel.
