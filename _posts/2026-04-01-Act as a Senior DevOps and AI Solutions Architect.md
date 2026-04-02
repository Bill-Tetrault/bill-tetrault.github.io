---
layout: post
title: "AI Homelab Beginner's Guide with Prompt"
author: "Bill Tetrault"
date: 2026-04-01
description: "Agentic AI Home Lab on Rocky Linux 9.7 — Complete Build Guide"
tags: [Linux, Rocky, Docker, Tutorial, Guide, DevOps, MCP, AI]
categories: [guides, tutorials, linux]
---
<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Act as a Senior DevOps and AI Solutions Architect. Provide a comprehensive, step-by-step technical guide to building an Agentic AI Home Lab on a minimal Rocky Linux 9.7 host.

Project Goals:
Core Tech: Use Docker Compose to manage the lab environment.
Agent Framework: Implement an MCP (Model Context Protocol) architecture. Recommend a specific MCP Client (e.g., Claude Desktop, Goose, or a CLI-based client) and an MCP Server that allows a Gemini-powered agent to interact with the local Linux filesystem and Docker socket.
LLM Integration: Configure the agent to use the Google Gemini API via API key.

System Administration \& Security:
Initial Setup: Detailed steps for creating a non-root sudo user and hardening the minimal Rocky install (disabling root SSH, configuring firewalld).
Connectivity: Instructions for generating Ed25519 SSH keys on Windows 11 and deploying them to the Rocky host.
Persistence: Configuration for Docker volumes and environment variable management (using .env files) to keep API keys out of compose files.

Technical Requirements:
Provide the specific dnf commands for dependencies (Node.js, Python, Docker-CE).
Include a sample docker-compose.yml for a starter service (like Nginx or Portainer) that the agent will eventually manage.
Explain the 'loop' of how the Agent uses MCP to execute shell commands or modify files on the host safely.
Tone: Technical, concise, and security-first.

Now I have enough to write a comprehensive, accurate guide. Let me compose this.

***

# Agentic AI Home Lab on Rocky Linux 9.7 — Complete Build Guide

This guide walks you through standing up a security-hardened Rocky Linux 9.7 host, wiring in Docker, and deploying a **Gemini-powered AI agent** using **Goose CLI** as the MCP client with the official `@modelcontextprotocol/server-filesystem` and `mcp-server-docker` as MCP servers. The agent loop lets Gemini reason over your host and issue tool calls that are executed locally via MCP — with you staying in control of what surfaces are exposed.

***

## Architecture Overview

Before writing a single command, understand the data flow:[^1]

```
  ┌────────────────────────────┐
  │  Goose CLI (MCP Host/Client) │  ◄── You interact here
  │  Provider: Google Gemini     │
  └─────────────┬──────────────┘
                │  JSON-RPC 2.0 over stdio
       ┌────────┴────────────────────┐
       │         MCP Servers          │
       │  ① mcp/filesystem (Docker)   │  ← R/W scoped dirs
       │  ② mcp-server-docker         │  ← Docker socket
       └──────────────────────────────┘
```

MCP uses **JSON-RPC 2.0** messages exchanged over `stdio` between the host (Goose) and each server subprocess. The Gemini model sees a tool schema for every MCP capability and decides *when* to call it — it never touches the socket or filesystem directly.[^2][^1]

***

## Phase 1 — Rocky Linux 9.7 Initial Setup

### Create a Non-Root Sudo User

```bash
# As root on first boot
useradd -m -s /bin/bash ailab
passwd ailab
usermod -aG wheel ailab
# Verify
id ailab  # should show wheel group
```

Lock the root account from password login immediately:

```bash
passwd -l root
```


### firewalld Hardening

Rocky 9 ships with firewalld active. Drop everything not explicitly needed:[^3]

```bash
sudo firewall-cmd --set-default-zone=drop
sudo firewall-cmd --zone=drop --add-service=ssh --permanent
# Only open additional ports as needed, e.g., Portainer HTTPS:
sudo firewall-cmd --zone=drop --add-port=9443/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

> ⚠️ **Critical:** Keep your current session open. Verify the new hardened session connects *before* closing the old one.

***

## Phase 2 — Ed25519 SSH Keys on Windows 11

Run these in **Windows Terminal (PowerShell)** on your Windows 11 machine:[^4]

```powershell
# Generate Ed25519 key pair with 100 bcrypt rounds
ssh-keygen -t ed25519 -C "bill@ailab" -a 100 -f "$env:USERPROFILE\.ssh\id_ed25519_ailab"

# Deploy public key to Rocky host (do this BEFORE disabling password auth)
type "$env:USERPROFILE\.ssh\id_ed25519_ailab.pub" | ssh ailab@<ROCKY_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# Test the key-based login
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_ailab" ailab@<ROCKY_IP>
```

Add a **host alias** to `~/.ssh/config` on Windows for convenience:

```
Host ailab
    HostName 192.168.x.x
    User ailab
    IdentityFile ~/.ssh/id_ed25519_ailab
    ServerAliveInterval 60
```
### SSH Hardening

Create a drop-in config file — never edit `sshd_config` directly:[^4]

```bash
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf << 'EOF'
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
PubkeyAcceptedAlgorithms ssh-ed25519
HostKeyAlgorithms ssh-ed25519
KexAlgorithms curve25519-sha256
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MaxAuthTries 3
LoginGraceTime 20
X11Forwarding no
AllowTcpForwarding no
AllowUsers ailab
LogLevel VERBOSE
EOF

sudo sshd -t  # test — silence means valid
sudo systemctl restart sshd
```

***

## Phase 3 — Install Dependencies via DNF

### System Update \& Packages

```bash
sudo dnf update -y
sudo dnf install -y git curl wget tar unzip vim bash-completion
```


### Node.js 22 (LTS) — Required for MCP servers

```bash
# Use NodeSource for the current LTS
curl -fsSL https://rpm.nodesource.com/setup_22.x | sudo bash -
sudo dnf install -y nodejs
node --version  # v22.x.x
npm --version
```


### Python 3.11+ \& pip

```bash
sudo dnf install -y python3 python3-pip
python3 --version  # 3.11.x on Rocky 9
```


### Docker CE + Docker Compose Plugin

Rocky ships with podman/buildah — remove them first:[^5][^3]

```bash
sudo dnf remove -y podman buildah containers-common
sudo dnf install -y dnf-utils
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo systemctl enable --now docker

# Add ailab user to docker group (eliminates need for sudo on docker commands)
sudo usermod -aG docker ailab
newgrp docker  # activate in current session
docker --version
docker compose version
```


***

## Phase 4 — Project Structure \& Secret Management

A clean directory structure keeps secrets, configs, and compose files separated:

```bash
mkdir -p ~/ailab/{compose,mcp,data/{portainer,nginx}}
cd ~/ailab
```


### `.env` File — API Keys Stay Out of Compose Files

```bash
cat > ~/ailab/.env << 'EOF'
# Google Gemini
GEMINI_API_KEY=your_gemini_api_key_here

# Portainer admin bootstrap password (hashed via htpasswd)
PORTAINER_PASSWORD=yourpassword

# Paths
DATA_ROOT=/home/ailab/data
EOF

chmod 600 ~/ailab/.env
```

> **Security rule:** `.env` is never committed to git. Add it to `.gitignore` immediately if you init a repo.

***

## Phase 5 — Docker Compose Starter Stack

This is the **target infrastructure** the Gemini agent will eventually manage. Save as `~/ailab/compose/docker-compose.yml`:[^6]

```yaml
# ~/ailab/compose/docker-compose.yml
# Managed by Gemini agent via MCP

networks:
  ailab-net:
    driver: bridge

volumes:
  portainer_data:

services:

  nginx:
    image: nginx:alpine
    container_name: ailab-nginx
    restart: unless-stopped
    networks:
      - ailab-net
    ports:
      - "8080:80"
    volumes:
      - ${DATA_ROOT}/nginx:/usr/share/nginx/html:ro
    labels:
      managed-by: "gemini-agent"

  portainer:
    image: portainer/portainer-ce:latest
    container_name: ailab-portainer
    restart: unless-stopped
    networks:
      - ailab-net
    ports:
      - "9443:9443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    labels:
      managed-by: "gemini-agent"
```

Launch it:

```bash
cd ~/ailab/compose
docker compose --env-file ../.env up -d
docker compose ps
```


***

## Phase 6 — MCP Servers Setup

You need two MCP servers: one for **filesystem** access and one for **Docker socket** access.

### MCP Server 1 — Official Filesystem Server

This runs as a scoped Docker container; it can only touch directories you explicitly mount:[^7][^8]

```bash
# Pull the official image
docker pull mcp/filesystem
```

You will reference this in Goose's config (next phase) — no separate process needed, Goose spawns it.

### MCP Server 2 — Docker Socket MCP Server

```bash
cd ~/ailab/mcp
git clone https://github.com/ckreiling/mcp-server-docker.git
cd mcp-server-docker
npm install
npm run build
```

This server exposes Docker API operations (list containers, start/stop, inspect, run) as MCP tools that the Gemini agent can call.[^9]

***

## Phase 7 — Goose CLI MCP Client + Gemini Configuration

**Goose** (by Block) is the recommended MCP client — it is CLI-native, runs headlessly on Linux, and has first-class Google Gemini support:[^10]

```bash
# Install Goose CLI
curl -fsSL https://github.com/block/goose/releases/latest/download/install.sh | bash
export PATH="$HOME/.local/bin:$PATH"

# Configure Gemini as the provider
goose configure
# Interactive wizard: select "Google" → "gemini-2.0-flash" → paste API key
```

Alternatively, set the env var directly from your `.env`:

```bash
source ~/ailab/.env
goose configure --set provider=google --set model=gemini-2.0-flash --set api_key=$GEMINI_API_KEY
```


### Wire the MCP Servers to Goose

Edit `~/.config/goose/config.yaml`:

```yaml
# ~/.config/goose/config.yaml
provider: google
model: gemini-2.0-flash

extensions:
  # MCP Server 1 — Filesystem (scoped to ailab project dir only)
  filesystem:
    type: stdio
    cmd: docker
    args:
      - run
      - -i
      - --rm
      - -v
      - /home/ailab/ailab:/home/ailab/ailab
      - mcp/filesystem
      - /home/ailab/ailab
    enabled: true

  # MCP Server 2 — Docker socket control
  docker:
    type: stdio
    cmd: node
    args:
      - /home/ailab/ailab/mcp/mcp-server-docker/dist/index.js
    env:
      DOCKER_HOST: "unix:///var/run/docker.sock"
    enabled: true
```

> The `ailab` user is in the `docker` group, so the MCP Docker server inherits socket access without `sudo`.[^3]

***

## Phase 8 — The Agentic Loop Explained

Understanding how the agent operates helps you scope permissions safely:[^2][^1]

```
1. You:    "Check if ailab-nginx is running and show me its access logs."

2. Goose:  Sends prompt + tool schemas to Gemini API (GEMINI_API_KEY)

3. Gemini: Returns a tool_call:
           { "name": "docker_inspect", "args": { "container": "ailab-nginx" } }

4. Goose:  Executes the tool call against mcp-server-docker via stdio JSON-RPC

5. MCP Server: Runs docker inspect ailab-nginx → returns JSON result

6. Goose:  Feeds result back to Gemini as a tool_response message

7. Gemini: Returns a tool_call:
           { "name": "read_file", "args": { "path": "/home/ailab/ailab/data/nginx/access.log" } }

8. Goose:  Executes against mcp/filesystem MCP server (scoped to /home/ailab/ailab)

9. Gemini: Synthesizes final natural-language answer to you
```

This loop repeats until Gemini has enough context to produce a final answer. **The model never directly touches the socket or filesystem** — every action is mediated and logged by the MCP server subprocess.[^11]

### Security Guardrails for the Loop

- **Filesystem server scope** — mount *only* `~/ailab`, never `/` or `/etc`[^7]
- **Docker socket** — the `ailab` user's group membership limits blast radius; never run Goose as root
- **API key isolation** — `GEMINI_API_KEY` lives only in `~/.config/goose/config.yaml` or sourced from `.env`, never hardcoded in compose files
- **Prompt injection defense** — review any file the agent reads before it feeds back into context if that file is externally writable
- **firewalld** — Gemini API calls go *outbound* only; no inbound AI port is needed

***

## Quick-Start Cheat Sheet

```bash
# Start the stack
cd ~/ailab/compose && docker compose --env-file ../.env up -d

# Launch the agent interactively
source ~/ailab/.env && goose session start

# Example agent prompts:
# "List all running containers and their uptime."
# "Edit ~/ailab/compose/docker-compose.yml to add a Watchtower service."
# "Restart the ailab-nginx container and tail its last 50 log lines."
```

The `managed-by: gemini-agent` label on each Compose service lets you later build automation that queries `docker ps --filter label=managed-by=gemini-agent`, giving the agent a clean, scoped inventory of what it owns.[^6]
<span style="display:none">[^12][^13][^14][^15][^16][^17][^18][^19][^20][^21][^22][^23][^24][^25][^26][^27][^28][^29]</span>

<div align="center">⁂</div>

[^1]: https://modelcontextprotocol.io/specification/2025-06-18

[^2]: https://ubos.tech/news/creating-custom-mcp-clients-with-gemini-a-guide-to-enhancing-ai-development/

[^3]: https://orcacore.com/install-docker-rocky-linux-9/

[^4]: https://zeonedge.com/lo/blog/ssh-hardening-2026-complete-guide-linux-server

[^5]: https://www.wilivm.com/blog/how-to-install-docker-on-rocky-linux-2/

[^6]: https://christianmendieta.ca/building-mcp-servers-3-python-examples-from-docker-to-ai-search/

[^7]: https://mcpservers.org/servers/modelcontextprotocol/filesystem

[^8]: https://hub.docker.com/r/mcp/filesystem

[^9]: https://github.com/ckreiling/mcp-server-docker

[^10]: https://blog.marcnuri.com/goose-on-machine-ai-agent-cli-introduction

[^11]: https://www.reddit.com/r/MCPservers/comments/1k7vy8w/mcp_meets_gemini_25_pro_a_deep_dive_with_step_by/

[^12]: https://blog.bytebytego.com/p/ep163-12-mcp-servers-you-can-use

[^13]: https://www.shareuhack.com/en/posts/best-mcp-servers-guide-2026

[^14]: https://www.docker.com/blog/docker-mcp-catalog-secure-way-to-discover-and-run-mcp-servers/

[^15]: https://dev.to/aws/running-model-context-protocol-mcp-servers-on-containers-using-finch-kj8

[^16]: https://blog.jetbrains.com/idea/2025/05/intellij-idea-2025-1-model-context-protocol/

[^17]: https://www.stackone.com/connectors/googlegemini/mcp/

[^18]: https://gist.github.com/manesec/efc649575974f4bc2902a2b338f920af

[^19]: https://developer.microsoft.com/blog/10-microsoft-mcp-servers-to-accelerate-your-development-workflow

[^20]: https://www.stackone.com/connectors/googlegemini/mcp-v2a/

[^21]: https://geminicli.com/docs/tools/mcp-server/

[^22]: https://docs.cloudbees.com/docs/cloudbees-unify-mcp-server/latest/install/google-gemini-cli

[^23]: https://inventivehq.com/knowledge-base/gemini/how-to-configure-mcp-integrations

[^24]: https://docs.snyk.io/integrations/snyk-studio-agentic-integrations/quickstart-guides-for-snyk-studio/gemini-cli-guide-1

[^25]: https://www.youtube.com/watch?v=h4YdImjE1RQ

[^26]: https://github.com/jaysm03/gemini-grounded-search

[^27]: https://docs.aws.amazon.com/solutions/latest/generative-ai-application-builder-on-aws/steps-to-build-mcp-server-docker-image.html

[^28]: https://developers.notion.com/guides/mcp/mcp

[^29]: https://docs.docker.com/ai/mcp-catalog-and-toolkit/toolkit/

