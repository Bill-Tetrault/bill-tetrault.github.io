---
layout: post
title: "OpenClaw Silicon Valley Build with Prompt"
author: "Bill Tetrault"
date: 2026-05-12
description: "OpenClaw Agentic AI Home Lab on Rocky Linux 9.7 — Complete Build Guide"
tags: [Linux, Rocky, Docker, Tutorial, Guide, DevOps, MCP, AI]
categories: [guides, tutorials, linux]
---
<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

## Overview

This guide walks you through installing and wiring up a local-first Pied-Piper HQ assistant stack on a fresh Rocky Linux 9.7 minimal install, using:

- Ollama with a local **Gemma 4 E4B** model as the LLM backend.
- **OpenClaw** as the agent framework and gateway, with **Mission Control** as the web UI.
- A **Telegram bot** as your primary chat interface.

You’ll end up with a small team of themed agents (inspired by the *Silicon Valley* cast) that you can talk to from Telegram and manage in a Mission Control dashboard.

***

## Architecture plan

At a high level, your setup will look like this:

1. **Rocky Linux 9.7 host**
    - NVIDIA drivers + CUDA stack for your RTX 2070.[^1][^2]
    - Systemd services for Ollama and OpenClaw.
2. **Ollama**
    - Installed natively on Rocky.
    - Runs as a background service exposing a local HTTP API on `http://localhost:11434`.[^3]
    - Stores Gemma 4 model weights and related data under `/data/ollama`.
3. **Model choice: Gemma 4 E4B via Ollama**
    - Gemma 4 comes in E2B, E4B, 26B A4B, and 31B.[^4][^3]
    - According to Google’s Gemma 4 memory table, E4B in 4‑bit quantization (`Q4_0`) needs about **5 GB VRAM**, while E2B needs about 3.2 GB, and the 26B/31B variants require **15.6 GB–17.4 GB VRAM or more**.[^4]
    - With an RTX 2070 (8 GB VRAM), the **largest Gemma 4 tier that realistically fits for general use is Gemma 4 E4B in Q4_0**, exposed in Ollama as `gemma4:e4b`.[^3][^4]
    - Larger 26B/31B tiers are not practical on 8 GB VRAM; E4B is the “max safe” choice, and E2B is the fallback if you see OOMs or slowdowns.[^4]
4. **OpenClaw + Mission Control**
    - OpenClaw is installed and configured via Ollama (`ollama launch openclaw`).[^5][^6]
    - The gateway runs as a background process; Mission Control is accessible via browser (e.g. `http://localhost:18789`).[^7]
    - OpenClaw is configured to use your **local** `gemma4:e4b` model as its default provider, not cloud APIs.[^5][^3]
5. **Telegram**
    - A Telegram bot created via **BotFather** with an API token and chat ID.[^8][^9]
    - OpenClaw’s `channels` configuration wired to that bot using `openclaw configure --section channels`.[^6][^5]
6. **Pied-Piper HQ**
    - A themed agent roster (Richard, Gilfoyle, Dinesh, Big Head, Jared, Monica).
    - Mission Control dashboard views tuned around: coding, infra, security, docs, coordination, and research.

***

## Prerequisites

Before you start:

1. **Base system**
    - Rocky Linux 9.7 minimal installed and booting cleanly.
    - You have a non-root user with `sudo` access (examples will use `tetraserv`).
    - System is connected to the internet (for package and model downloads).
2. **Access**
    - SSH access or local console with copy–paste capability.
    - A browser on your LAN to open Mission Control (or use text-mode via local browser/SSH tunneling).
3. **Accounts**
    - Telegram account on your phone or desktop.
    - Ability to talk to **@BotFather** in Telegram to create a bot.[^9][^8]
4. **Hardware**
    - Intel i5‑8700K, 32 GB RAM, RTX 2070 8 GB VRAM, 1 TB NVMe, 5 TB HDD (as provided).
    - Enough free space and patience for model downloads (several GB per model).[^3]

***

## Storage layout under `/data`

You’ll keep everything for this project under `/data` so it’s easy to back up, move, and snapshot.

### Recommended layout

We’ll use:

- **On NVMe (fast, low latency)**
    - Active models, apps, configs, logs:
        - `/data/ollama` – models and Ollama state.
        - `/data/openclaw/app` – OpenClaw app files (if we choose to clone or store local bits).
        - `/data/openclaw/config` – OpenClaw config files, agent definitions, routing configs.
        - `/data/openclaw/logs` – OpenClaw and Mission Control logs.
        - `/data/mission-control` – Mission Control-specific dashboards, layouts, and assets.
- **On HDD (big and slower)**
    - Backups, archives, exported logs:
        - `/data/backups` – config and snapshot backups.
        - `/data/archive` – old logs, export dumps from Mission Control.

> If the HDD is a separate block device and not mounted yet, mount it at e.g. `/mnt/hdd` and then bind-mount parts into `/data/backups` later. That is environment-specific, so adjust as needed.

### Create directories and set ownership

1. Create base layout (run as `root` or with `sudo`):
```bash
sudo mkdir -p \
  /data/ollama \
  /data/openclaw/app \
  /data/openclaw/config \
  /data/openclaw/logs \
  /data/mission-control \
  /data/backups \
  /data/archive
```

2. Make your main user the owner so OpenClaw and tools can write there:
```bash
sudo chown -R tetraserv:tetraserv /data
sudo chmod -R 750 /data
```

- Replace `tetraserv` with your actual username if different.
- `750` gives full access to the owner, read+execute to the group, no access to others (a decent default for homelab).

You can later tighten specific subdirectories (e.g. backups) with `chmod 700`.

***

## Rocky Linux prep

These steps assume a fresh Rocky 9.7 minimal installation.

### 1. Update the base system

```bash
sudo dnf update -y
sudo dnf install -y epel-release
```

- `epel-release` enables the Extra Packages for Enterprise Linux (EPEL) repository, which many tools rely on.[^2][^1]


### 2. Install common tools

```bash
sudo dnf install -y \
  git curl wget vim tmux htop \
  unzip tar bzip2 jq \
  firewalld policycoreutils-python-utils \
  bash-completion
```

- These give you basic CLI tooling, text editors, JSON tools, firewall support, and SELinux utilities.

Enable and start firewalld:

```bash
sudo systemctl enable --now firewalld
```

You’ll open ports later as needed.

### 3. Install Node.js (for OpenClaw)

OpenClaw requires modern Node.js and npm. The general guidance is Node.js 22 or newer.[^7]

Rocky’s built-in modules may not ship Node 22 yet. Use the NodeSource RPM repo (this is a **common pattern, but verify in upstream docs before running**; version names may change):

```bash
curl -fsSL https://rpm.nodesource.com/setup_22.x | sudo bash -
sudo dnf install -y nodejs
node -v
npm -v
```

- `node -v` and `npm -v` should show recent versions.
- If Node 22 is not required by your OpenClaw release, any supported LTS listed in OpenClaw’s docs is fine.[^6][^7]

***

## NVIDIA driver install

You want the official NVIDIA driver with CUDA support for your RTX 2070 so Ollama can use GPU acceleration.

Rocky Linux has an official guide using the NVIDIA CUDA repo and `nvidia-driver` module.[^1][^2]

### 1. Enable required repos and tools

```bash
sudo dnf config-manager --set-enabled crb
sudo dnf install -y epel-release
sudo dnf groupinstall -y "Development Tools"
sudo dnf install -y \
  kernel-devel-matched kernel-headers \
  dkms \
  pciutils elfutils-libelf-devel \
  libglvnd-opengl libglvnd-glx libglvnd-devel \
  acpid
```

- `kernel-devel-matched` and `kernel-headers` ensure you have matching headers for your running kernel.[^2]
- `dkms` automatically rebuilds kernel modules after kernel upgrades.[^1][^2]


### 2. Add NVIDIA CUDA repo

```bash
sudo dnf config-manager --add-repo \
  https://developer.download.nvidia.com/compute/cuda/repos/rhel9/$(uname -i)/cuda-rhel9.repo
sudo dnf clean expire-cache
```

- This adds the official NVIDIA CUDA repo for RHEL9/Rocky9.[^2][^1]


### 3. Disable the Nouveau driver

Disable the open-source Nouveau driver to avoid conflicts:

```bash
sudo grubby --args="nouveau.modeset=0 rd.driver.blacklist=nouveau" --update-kernel=ALL
```

- This adds kernel parameters to disable Nouveau across all installed kernels.[^1]

If you have Secure Boot enabled, you may need to enroll DKMS keys with `mokutil` (see Rocky’s NVIDIA docs for details; **verify in upstream docs before running**).[^2][^1]

### 4. Install NVIDIA driver (proprietary, compute+desktop)

For a compute + desktop capable setup on Rocky 9, using the proprietary kernel module:

```bash
sudo dnf module enable -y nvidia-driver:latest-dkms
sudo dnf install -y nvidia-driver nvidia-driver-cuda kmod-nvidia-latest-dkms
```

- `nvidia-driver` and `kmod-nvidia-latest-dkms` provide the core driver.[^1][^2]
- `nvidia-driver-cuda` enables CUDA compute support used by Ollama.[^2]


### 5. Reboot and verify GPU

```bash
sudo reboot
```

After reboot, check:

```bash
nvidia-smi
```

You should see your RTX 2070 listed with driver version, CUDA version, and usage. If `nvidia-smi` fails, revisit the Rocky + NVIDIA driver docs before continuing.[^1][^2]

***

## Ollama install and model setup

Ollama is your local model runtime, exposing a simple HTTP API and CLI.[^3]

### 1. Install Ollama on Rocky Linux

Ollama provides a generic Linux install script for x86_64 systems.[^3]

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

- This script typically installs the `ollama` binary, creates a dedicated `ollama` user, and sets up a `systemd` service.[^10][^3]

Check version:

```bash
ollama --version
```

If `ollama` is not found, log out/log in or check `echo $PATH`. If still missing, **verify in upstream docs before running**, as install paths or scripts may have changed.[^11][^3]

### 2. Ensure Ollama service is running

```bash
sudo systemctl enable --now ollama
sudo systemctl status ollama
```

You should see the service as `active (running)`. If your install script uses a different service name (e.g. `ollama.service` vs `ollama-daemon.service`), adjust accordingly (**example configuration – adjust for current release naming**).[^12][^10]

Test the local API:

```bash
curl http://localhost:11434/api/tags
```

You should get a JSON list of models (initially empty).[^3]

### 3. Point Ollama models at `/data/ollama` (optional but recommended)

By default, Ollama stores models under a system directory (e.g. `/usr/share/ollama` or `/var/lib/ollama` depending on your version; **verify in upstream docs**). To keep your model data in `/data/ollama`, you can use environment variables via systemd.[^11][^3]

Create an environment file (this is based on an example from Ollama systemd discussions; treat as **example configuration**).[^10]

```bash
sudo tee /etc/ollama.conf >/dev/null <<'EOF'
# Example: put models under /data/ollama
OLLAMA_MODELS=/data/ollama
EOF
```

Edit the Ollama service unit:

```bash
sudo sed -i 's#^\[Service\]#[Service]\nEnvironmentFile=/etc/ollama.conf#' /etc/systemd/system/ollama.service
sudo systemctl daemon-reload
sudo systemctl restart ollama
```

- If your `ollama.service` already uses `EnvironmentFile` or has a different layout, merge this manually instead of using `sed`.[^10]
- Check `systemctl status ollama` again to verify it starts cleanly.

> Note: The exact env var (`OLLAMA_MODELS`) and supported overrides can change. Check the latest Ollama README before relying on this pattern.[^11]

### 4. Pull Gemma 4 models

Ollama installs no models by default; you pull them explicitly.[^3]

We’ll pull both E4B (primary) and E2B (fallback):

```bash
ollama pull gemma4:e4b
ollama pull gemma4:e2b
```

- Ollama’s Gemma integration exposes Gemma 4 in sizes E2B, E4B, 26B A4B, and 31B.[^3]
- E4B Q4_0 requires about 5 GB VRAM just for weights, leaving ~3 GB for context and overhead on an 8 GB GPU.[^4]
- E2B Q4_0 requires about 3.2 GB VRAM; it’s safer if you experience OOM issues or need more context length on the 8 GB card.[^4]

List models:

```bash
ollama list
```

You should see at least:

- `gemma4:e4b`
- `gemma4:e2b`


### 5. Smoke test Gemma 4 E4B from CLI

```bash
ollama run gemma4:e4b "Summarize the mission of Pied-Piper HQ in two sentences."
```

If this is slow or crashes with CUDA/VRAM errors, fall back to:

```bash
ollama run gemma4:e2b "Summarize the mission of Pied-Piper HQ in two sentences."
```

On an RTX 2070:

- **E4B** should run but may be noticeably slower on long prompts or concurrent loads.
- **E2B** will be faster and more stable but slightly less capable on complex reasoning and coding tasks.[^13][^4]

***

## OpenClaw install

OpenClaw integrates tightly with Ollama. The recommended path from Ollama’s docs is to let **Ollama install and configure OpenClaw for you** via `ollama launch openclaw`.[^5][^6]

### 1. Install OpenClaw via Ollama

From your normal user (e.g. `tetraserv`):

```bash
ollama launch openclaw --model gemma4:e4b
```

This flow (per Ollama’s integration docs):[^6][^5]

- Installs OpenClaw via `npm` if it’s not already present.
- Configures OpenClaw’s provider to point at your local Ollama instance.
- Sets your chosen model (`gemma4:e4b`) as the primary default.
- Starts the OpenClaw gateway and opens the text UI.

If you instead want to configure without launching the gateway right away:

```bash
ollama launch openclaw --config
```

This lets you adjust settings (like model choice) and then start the gateway later.[^5]

> If `ollama launch openclaw` fails due to Node.js or npm version issues, verify your Node version and check OpenClaw’s official docs for current requirements.[^7][^6]

### 2. Verify OpenClaw CLI

After installation, you should have the `openclaw` CLI available globally:

```bash
openclaw --version
```

You can check gateway status:

```bash
openclaw gateway status
```

You should see the gateway as **Running** (or similar) when started.[^7][^5]

### 3. Configure OpenClaw to use local Gemma 4 by default

If you didn’t already specify the model during `ollama launch`, you can set or change it later:

```bash
ollama launch openclaw --model gemma4:e4b
```

- This updates OpenClaw’s configuration and restarts the gateway if needed.[^5]
- If you need to downgrade to E2B later due to VRAM or speed issues:

```bash
ollama launch openclaw --model gemma4:e2b
```


### 4. Mission Control basic access

OpenClaw’s Windows install guides reference **Mission Control** as the dashboard accessible at `http://localhost:18789` once the agent is running. On Linux installs via Ollama, the same Mission Control service is exposed by the gateway.[^7]

From a browser on the Rocky box or via SSH tunnel, open:

- `http://localhost:18789`

You should see Mission Control showing the OpenClaw gateway status, logs, and memory usage.[^7]

***

## Mission Control install

Mission Control is part of the OpenClaw stack; you don’t install it separately. You do, however, need to make sure:

- The gateway is running.
- The HTTP port (18789) is reachable from your admin machine.


### 1. Confirm Mission Control is reachable

From the Rocky host:

```bash
curl -I http://localhost:18789
```

You should see an HTTP 200/302/other non-error code if the UI is up.

If you want to reach it from another machine on your LAN, open the firewalld port:

```bash
sudo firewall-cmd --permanent --add-port=18789/tcp
sudo firewall-cmd --reload
```

Then from your workstation browser, visit:

- `http://<rocky-host-ip>:18789`


### 2. Example systemd service for gateway persistence (optional)

If `ollama launch openclaw` doesn’t already set up a persistent service, you can add a simple `systemd` unit to start the gateway on boot. Treat this as **example configuration – adjust for current release naming and paths**.

```bash
sudo tee /etc/systemd/system/openclaw-gateway.service >/dev/null <<'EOF'
[Unit]
Description=OpenClaw Gateway
After=network.target

[Service]
Type=simple
User=tetraserv
Group=tetraserv
WorkingDirectory=/home/tetraserv
ExecStart=/usr/bin/openclaw gateway start
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway
```

- Replace `/usr/bin/openclaw` and `/home/tetraserv` as appropriate for your system.

Check status:

```bash
systemctl status openclaw-gateway
```


***

## Telegram integration

You’ll wire Telegram into OpenClaw so you can talk to the Pied-Piper HQ agents from your phone.

### 1. Create a Telegram bot with BotFather

On your phone or desktop:

1. Open Telegram.
2. Search for `@BotFather` and start a chat.[^8][^9]
3. Send `/newbot`.
4. Follow the prompts:
    - **Bot name**: e.g. `Pied-Piper HQ Bot`.
    - **Username**: must end with `bot`, e.g. `Pied-Piper_hq_bot`.[^8]
5. BotFather replies with a message containing:
    - A link to your bot (`https://t.me/your_bot_username`).
    - An **HTTP API token** (keep this secret).[^9][^8]

Copy the bot token; you’ll need it for OpenClaw.

### 2. Get your chat ID

Simplest method (from a browser):[^9]

1. Start a conversation with your bot in Telegram (send any message).
2. In a browser, go to:

```
https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates
```

3. In the JSON output, find the `"chat"` object and copy the `"id"` value.[^9]

Keep both:

- `BOT_TOKEN`
- `CHAT_ID`


### 3. Configure Telegram in OpenClaw

OpenClaw provides a `channels` configuration section for messaging platforms.[^6][^5]

Run:

```bash
openclaw configure --section channels
```

- Follow the interactive prompts to add a Telegram channel.
- When asked for bot token and chat ID, paste your `BOT_TOKEN` and `CHAT_ID`.
- When finished, choose “Finished” or equivalent to save your configuration.[^6]

> Because OpenClaw’s configuration flows can change over time, treat the exact prompts as **example configuration** and rely on the on-screen help and current docs if you see differences.[^5][^6]

After configuration, restart the gateway if it doesn’t restart automatically:

```bash
openclaw gateway restart
# or, if using systemd:
sudo systemctl restart openclaw-gateway
```

Send a test message to your bot; you should see replies from Gemma 4 via OpenClaw.

***

## Pied-Piper HQ agent roster

This section defines a themed agent team for Pied-Piper HQ, lightly inspired by *Silicon Valley* but written to be genuinely useful in a homelab.

You can represent these as separate OpenClaw “agents” or routing profiles in Mission Control (exact implementation depends on your OpenClaw version; treat this as **example configuration**).

### Richard – Lead Architect \& Refactorer

- **Purpose**
High-quality coding help, refactoring, and systems design advice for Pied-Piper HQ.
- **Personality prompt/style**
Calm, thoughtful, slightly anxious about correctness. Prefers clean architecture and long-term maintainability over hacks. Explains trade-offs and points out edge cases.
- **Primary responsibilities**
    - Design service boundaries and APIs.
    - Refactor messy scripts into clean modules.
    - Guide migration paths (e.g., from ad hoc scripts to containerized services).
- **Example tasks**
    - “Refactor this Bash backup script into a robust Python tool with logging.”
    - “Design a small microservice for querying OpenClaw logs with an HTTP API.”
- **Recommended guardrails**
    - Avoid making irreversible infrastructure changes directly; suggest steps as code or commands instead.
    - For destructive operations (dropping DBs, formatting disks), require explicit human approval in Mission Control.
- **Suggested placement/routing**
    - Mission Control **Coding** and **Architecture** panes.
    - Routed from Telegram when user labels a request as “code”, “refactor”, or similar.


### Gilfoyle – Infrastructure \& Reliability

- **Purpose**
Infrastructure operations, observability, and performance tuning.
- **Personality prompt/style**
Dry, sarcastic, but technically precise. Assumes a reasonably competent operator on the other end. Values automation and reproducibility.
- **Primary responsibilities**
    - Design and validate backup strategies.
    - Suggest monitoring/alerting improvements.
    - Optimize resource usage (CPU, RAM, VRAM) and service placement.
- **Example tasks**
    - “Review this `systemd` unit for OpenClaw and suggest improvements.”
    - “Propose a backup rotation scheme for `/data` and logs.”
- **Recommended guardrails**
    - Any infrastructure-changing action (installing packages, editing system configs) must:
        - Be proposed in dry-run form (`bash` snippets, playbook pseudo-code).
        - Require explicit approval via Mission Control before execution.
    - No direct credential handling; instruct humans where to store secrets.
- **Suggested placement/routing**
    - Mission Control **Infra** pane.
    - Default reviewer for any Pied-Piper HQ infrastructure tasks.


### Dinesh – Coding \& Experiments

- **Purpose**
Rapid prototyping, code experiments, and alternate implementations.
- **Personality prompt/style**
Competitive, a bit defensive, enjoys “clever” solutions but will accept more robust patterns when asked.
- **Primary responsibilities**
    - Try out new libraries, APIs, or agent patterns.
    - Generate quick proof-of-concept scripts and test harnesses.
    - Provide alternate implementations to Richard’s proposed designs.
- **Example tasks**
    - “Draft a quick Python script to query the Ollama HTTP API and log responses.”
    - “Give me a more concise version of this Bash function.”
- **Recommended guardrails**
    - All outputs are prototypes; label them clearly as such.
    - No production infra commands; send anything impactful to Gilfoyle or Jared for review.
- **Suggested placement/routing**
    - Mission Control **Coding Experiments** or **Lab** pane.
    - Telegram “sandbox” or “dev” queue.


### Big Head – Documentation \& Onboarding

- **Purpose**
Documentation, simplified explanations, and gentle onboarding.
- **Personality prompt/style**
Easygoing, friendly, a bit clueless at first glance but surprisingly helpful. Explains things in simple terms without jargon.
- **Primary responsibilities**
    - Turn complex infra or code into beginner-friendly docs.
    - Generate README files, runbooks, and quickstart guides.
    - Create “explain like I’m new to homelabs” content.
- **Example tasks**
    - “Write a simple README for how to restart OpenClaw and Mission Control.”
    - “Explain what GPU VRAM means and why Gemma 4 E4B is the right choice for this box.”
- **Recommended guardrails**
    - Avoid editing live configs; output documentation only.
    - When unsure, explicitly state assumptions and encourage verification.
- **Suggested placement/routing**
    - Mission Control **Docs \& Onboarding** pane.
    - Default responder for new user questions in Telegram (labelled as “help” or “intro”).


### Jared – Project Coordination \& Safety

- **Purpose**
Coordination, task breakdown, prioritization, and safety checks.
- **Personality prompt/style**
Helpful, meticulous, occasionally over-enthusiastic. Focused on checklists and making sure nothing falls through the cracks.
- **Primary responsibilities**
    - Turn vague goals into task lists.
    - Coordinate which agent should handle which part of a request.
    - Perform safety/impact reviews before infra changes are approved.
- **Example tasks**
    - “Create a plan for migrating Pied-Piper HQ to a new host.”
    - “Review Gilfoyle’s backup strategy and flag missing tests.”
- **Recommended guardrails**
    - Always ask for confirmation before scheduling or suggesting impactful changes.
    - When in doubt, escalate to human with clearly highlighted risks.
- **Suggested placement/routing**
    - Mission Control **HQ Overview** pane and **Approvals** queue.
    - Approver for infra-changing tasks and production code deployments.


### Monica – Research \& Product Strategy

- **Purpose**
Research, comparison, and homelab/product support for Pied-Piper HQ.
- **Personality prompt/style**
Direct, analytical, supportive but demanding of clarity. Prefers evidence-based recommendations.
- **Primary responsibilities**
    - Compare tools, models, and architectures (e.g. Gemma 4 E4B vs E2B, other LLMs).
    - Research best practices for homelab security, backup, and automation.
    - Help prioritize features and improvements for Pied-Piper HQ.
- **Example tasks**
    - “Compare local Ollama + Gemma vs cloud LLMs for this workflow.”
    - “Propose a roadmap for improving Mission Control dashboards over the next 3 months.”
- **Recommended guardrails**
    - Label all recommendations with confidence level and assumptions.
    - Avoid giving legal or compliance advice; keep things at best-practices level for homelabs.
- **Suggested placement/routing**
    - Mission Control **Research \& Strategy** pane.
    - Telegram queries tagged as “compare”, “research”, or “roadmap”.

***

## First-run validation

Run these checks in order to confirm that the stack is working end-to-end.

### 1. GPU and drivers

```bash
nvidia-smi
```

- Confirm your RTX 2070 is visible, with correct VRAM and driver version.[^2][^1]


### 2. Ollama service and model

```bash
sudo systemctl status ollama
curl http://localhost:11434/api/tags
ollama run gemma4:e4b "Say 'Pied-Piper HQ online' in one sentence."
```

- Ensure `ollama` is `active (running)` and that `gemma4:e4b` responds reasonably quickly.[^3]

If you see VRAM errors, test with E2B:

```bash
ollama run gemma4:e2b "Say 'Pied-Piper HQ online' in one sentence."
```


### 3. OpenClaw gateway

```bash
openclaw gateway status
```

If not running:

```bash
openclaw gateway start
```

or, if using the systemd unit:

```bash
sudo systemctl restart openclaw-gateway
sudo systemctl status openclaw-gateway
```

You should see the gateway running without errors.[^5]

### 4. Mission Control UI

From the Rocky host:

```bash
curl -I http://localhost:18789
```

Then from your workstation browser:

- `http://<rocky-host-ip>:18789`

Confirm you can:

- See overall agent/gateway status.
- Access logs and any available dashboards.[^7]


### 5. Telegram loop

From Telegram:

1. Send “`/start`” or a simple message to your bot.
2. You should see a reply handled via OpenClaw using Gemma 4:
    - If you’ve set routing, you could test by saying “Ask Richard to refactor this function: …” or “Gilfoyle, check my infra idea.”

If no response, revisit the Telegram integration section, checking:

- Bot token and chat ID.
- `openclaw configure --section channels` settings.
- Gateway logs in Mission Control or CLI.

***

## Troubleshooting

Here are common failure modes and how to approach them.

### NVIDIA / GPU issues

- **`nvidia-smi` not found or errors**
    - Confirm `nvidia-driver` packages are installed and `nouveau` is disabled.[^1][^2]
    - Verify kernel headers match the running kernel (`kernel-devel-matched` etc.).[^2]
    - Reboot after driver changes.
    - Check `dmesg | grep -i nvidia` for module load errors.
- **Ollama falls back to CPU or fails on CUDA**
    - Check `nvidia-smi` while running `ollama run gemma4:e4b` to see GPU usage.
    - If no GPU usage, check Ollama logs (`journalctl -u ollama`) for CUDA-related messages.[^3]


### Ollama / model issues

- **`ollama` command not found**
    - Re-run the install script or ensure `/usr/local/bin` is in your `PATH`.[^11][^3]
    - Check location with `which ollama`.
- **Cannot reach `http://localhost:11434`**
    - `systemctl status ollama` for errors.
    - `journalctl -u ollama` for logs.
    - Ensure nothing else is bound to port 11434.
- **VRAM OOM or slow generation**
    - Switch from `gemma4:e4b` to `gemma4:e2b` via `ollama launch openclaw --model gemma4:e2b` or temporarily use `ollama run` with E2B only.[^4][^3]
    - Reduce context length in prompts and avoid huge inputs.


### OpenClaw / Mission Control issues

- **`openclaw` not found**
    - Re-run `ollama launch openclaw` and follow prompts.[^6][^5]
    - Verify Node.js (e.g. `node -v`) meets supported version.
- **Gateway refuses connections / status shows stopped**
    - Run `openclaw gateway start` manually and watch output.
    - Check logs via Mission Control or CLI.
    - If you created a systemd service, inspect `journalctl -u openclaw-gateway`.
- **Mission Control not reachable on port 18789**
    - Confirm gateway is running.
    - Check firewalld rules (`firewall-cmd --list-ports`) and add port 18789 if missing.[^7]
    - Try `curl -I http://localhost:18789` locally to distinguish local vs remote network issues.


### Telegram issues

- **Bot doesn’t respond**
    - Confirm BotFather token is correct and not leaked/invalidated.[^8][^9]
    - Re-run `openclaw configure --section channels` and re-enter Token and Chat ID.[^6][^5]
    - Use `getUpdates` URL to confirm Telegram sees your messages and that the `chat.id` matches what you configured.[^9]
- **Messages go to wrong chat**
    - Double-check `chat.id` and consider creating a dedicated group or channel, then re-fetching the ID via `getUpdates`.[^8][^9]

***

## Security and maintenance

This is a single-node homelab deployment, but you still want basic hygiene.

### 1. OS and package updates

Schedule regular updates:

```bash
sudo dnf update -y
```

- Consider enabling automatic security updates if this box is internet-exposed.


### 2. Firewall and network exposure

- Restrict public access to:
    - Ollama port `11434` – ideally only reachable from localhost or trusted admin hosts.
    - Mission Control port `18789` – restricted to your LAN or VPN.
- Use `firewall-cmd` to limit access, or put the host behind a reverse proxy/VPN.


### 3. User separation

- Run Ollama as its dedicated `ollama` user (installer usually configures this).[^10][^3]
- Run OpenClaw gateway as a non-root user (`tetraserv` or a dedicated `openclaw` account).
- Keep `/data` permissions tight (`750` or `700` where appropriate).


### 4. Secrets management

- Never commit BotFather tokens or chat IDs to public repos.
- Store secrets in:
    - OpenClaw’s configuration mechanisms.
    - Environment files with permissions `600`.
- Rotate tokens if you suspect leaks.


### 5. Backup and restore strategy

Use your HDD for backups:

1. **Configs and definitions**
    - Periodically archive:
        - `/data/openclaw/config`
        - `/data/mission-control`
        - Any custom agent definitions.
2. **Ollama models**
    - You can always re-pull models with `ollama pull`, so you usually don’t need to back them up.
    - If bandwidth is constrained, a compressed backup of `/data/ollama` to your HDD is reasonable.

Example simple backup script (run daily via `cron`):

```bash
#!/usr/bin/env bash
set -e
BACKUP_DIR=/data/backups/$(date +%Y%m%d-%H%M%S)
mkdir -p "$BACKUP_DIR"
tar czf "$BACKUP_DIR/openclaw-config.tgz" /data/openclaw/config
tar czf "$BACKUP_DIR/mission-control.tgz" /data/mission-control
```

- Keep a rotation policy (e.g. delete backups older than 30 days).


### 6. Updating Ollama and OpenClaw

- **Ollama**: Re-run the official install script or follow the update instructions in the Ollama README (check for breaking changes and new configuration options).[^11][^3]
- **OpenClaw**:
    - Use the upgrade path described in its docs (often `npm`-based).
    - After updates, re-run `openclaw configure` sections if configs changed format.

After updates:

- Re-test `nvidia-smi`, `ollama run`, Mission Control, and Telegram flows.

***

## Next improvements

Once the base Pied-Piper HQ stack is live and stable, you can:

1. **Add more local models**
    - Pull additional models via `ollama pull` for specialized tasks (e.g. smaller fast models for quick replies, coding-specialized variants, etc.).[^11][^3]
    - Use OpenClaw’s configuration to route certain tasks to different models (e.g. use E2B for quick Telegram responses and E4B for deep research).
2. **Enhance Mission Control dashboards**
    - Create separate panes for:
        - **Coding queue** (Richard \& Dinesh).
        - **Infra queue** (Gilfoyle \& Jared).
        - **Docs \& onboarding** (Big Head).
        - **Research \& strategy** (Monica).
    - Add approval steps (Jared) before executing any infra-changing action suggested by Gilfoyle or Dinesh.
3. **Integrate more channels**
    - Use `openclaw configure --section channels` to add Slack, Discord, or other messaging apps if you later want multi-channel access.[^5][^6]
4. **Automate backups and health checks**
    - Add cron jobs or systemd timers for backups, and simple health check scripts that alert you (via Telegram) if `nvidia-smi`, `ollama`, or the gateway are failing.

With this setup, Pied-Piper HQ becomes a durable, GPU‑accelerated, mostly local agent system you can grow over time while keeping control of your data and infrastructure.
<span style="display:none">[^14][^15][^16][^17][^18][^19][^20][^21][^22][^23][^24][^25][^26][^27][^28][^29][^30]</span>

<div align="center">⁂</div>

[^1]: https://docs.rockylinux.org/9/desktop/display/installing_nvidia_gpu_drivers/

[^2]: https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/rocky-linux.html

[^3]: https://ai.google.dev/gemma/docs/integrations/ollama

[^4]: https://ai.google.dev/gemma/docs/core

[^5]: https://docs.ollama.com/integrations/openclaw

[^6]: https://ollama.com/blog/openclaw-tutorial

[^7]: https://www.nimopc.com/blogs/our-blog/2026-guide-how-to-install-openclaw-ai-on-windows

[^8]: https://www.piwebsolution.com/create-a-telegram-bot-using-botfather-and-get-the-api-token/

[^9]: https://www.mikemurphy.co/telegram/

[^10]: https://github.com/ollama/ollama/issues/3516

[^11]: https://github.com/ollama/ollama/blob/main/README.md

[^12]: https://www.reddit.com/r/ollama/comments/1jakaup/ollama_running_on_ubuntu_server_systemd_service/

[^13]: https://dev.to/geek_/gemma-4-vram-requirements-the-hardware-guide-i-wish-i-had-3plo

[^14]: https://tutorialforlinux.com/2025/10/30/how-to-install-ollama-on-rocky-linux-9-step-by-step/

[^15]: https://huggingface.co/SanctumAI/gemma-2-9b-it-GGUF

[^16]: https://x.com/BentoBoiNFT/status/2028957770687427011

[^17]: https://theserverside.tistory.com/3310

[^18]: https://huggingface.co/google/gemma-2-9b-it/discussions/39

[^19]: https://www.youtube.com/watch?v=-YQZ05q4Nps

[^20]: https://gemma-4.org

[^21]: https://ollama.com/library/gemma2:9b

[^22]: https://ollama.com/blog/gemma2

[^23]: https://ollama.com/VladimirGav/gemma4-26b-16GB-VRAM

[^24]: https://ollama.com/library/gemma2:9b-instruct-q4_K_M/blobs/109037bec39c

[^25]: https://unsloth.ai/docs/models/gemma-4

[^26]: https://ollama.com/library/gemma

[^27]: https://www.reddit.com/r/LocalLLaMA/comments/1drxhlh/gemma_2_9b_appreciation_post/

[^28]: https://localllm.in/blog/ollama-vram-requirements-for-local-llms

[^29]: https://ollama.com/mannix/gemma2-9b

[^30]: https://dev.to/purpledoubled/how-to-run-googles-gemma-4-locally-with-ollama-all-4-model-sizes-compared-2pbh

