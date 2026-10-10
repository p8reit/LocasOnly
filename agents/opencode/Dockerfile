# Pin the agent runtime: update deliberately after testing.
FROM node:22-bookworm-slim

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       git python3 curl jq ripgrep ca-certificates \
    && rm -rf /var/lib/apt/lists/* \
    && npm install -g opencode-ai@1.18.35

RUN groupadd --gid 1000 agent \
    && useradd --uid 1000 --gid 1000 --create-home agent \
    && mkdir -p /workspace /home/agent/.config/opencode \
       /home/agent/.local/share /home/agent/.cache \
    && chown -R agent:agent /workspace /home/agent

USER agent
ENV HOME=/home/agent
WORKDIR /workspace
