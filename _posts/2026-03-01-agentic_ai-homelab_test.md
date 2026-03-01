# Agentic AI Home Lab on Proxmox

> A step‑by‑step guide (from zero to working AI agents in Docker) written from the perspective of a recent computer science graduate.

---

## 1. What You Will Build

By the end of this guide you will have:

- A Proxmox VE host running on your homelab hardware.
- An Ubuntu Server VM dedicated to containers (Docker + Docker Compose).
- A basic “agentic AI” stack using a modern agent framework (for example LangGraph, CrewAI, or AutoGen) running inside Docker.
- A development workflow to:
  - Edit code on your laptop.
  - Build images in Docker.
  - Deploy and test agents in your home lab.
- Optional: Portainer for container management via web UI.

---

## 2. Prerequisites

You do **not** need prior experience with Proxmox, Docker, or AI agents. You should have:

- A physical machine that will become your Proxmox host:
  - 4+ cores, 16 GB+ RAM recommended.
  - At least 256 GB SSD or NVMe.
- A second device (laptop/desktop) with:
  - SSH client (Windows: PowerShell, macOS/Linux: Terminal).
  - Web browser.
- Network:
  - Home router handing out DHCP addresses.
  - Ability to access your Proxmox host via local IP.

Accounts / software:

- Modern browser (Chrome, Edge, Firefox, etc.).
- GitHub account (optional but recommended).
- An OpenAI / compatible LLM API key (or local model later).

---

## 3. Proxmox VE Installation

### 3.1 Downloading Proxmox

1. Go to the Proxmox VE download page.
2. Download the latest Proxmox VE ISO.
3. Use a tool such as Rufus (Windows) or `dd` (Linux/macOS) to create a bootable USB.

### 3.2 Installing Proxmox on Bare Metal

1. Boot your server from the USB.
2. Choose “Install Proxmox VE”.
3. Follow the wizard:
   - Target disk: your main SSD/NVMe.
   - Country, time zone, keyboard: configure as appropriate.
   - Password: choose a strong root password and record it.
   - Management network: typically your main NIC with DHCP.

4. After installation, the console will show a URL, for example:

   - `https://192.168.1.50:8006`

5. On your laptop, open that URL and accept the browser’s TLS warning.

---

## 4. First Steps in Proxmox

### 4.1 Logging In

- Username: `root`
- Realm: `pam`
- Password: the one you set during install.

You will land on the Proxmox web UI.

### 4.2 Basic Proxmox Concepts

- **Node**: your physical Proxmox server.
- **VM**: full virtual machine (virtual hardware, runs its own OS).
- **Container (LXC)**: lightweight OS-level virtualization.

For this guide, we will:

- Use a **VM** for Docker (simpler, clean separation).
- Optionally later use LXC if you prefer.

---

## 5. Create the Ubuntu Docker VM

### 5.1 Download an Ubuntu Server ISO

1. Download Ubuntu Server LTS ISO.
2. In the Proxmox UI:
   - Select your node → “local” storage → “ISO Images” → “Upload”.
   - Upload the Ubuntu ISO.

### 5.2 Create the VM

1. Click “Create VM”.
2. General:
   - Node: your Proxmox node.
   - VM ID: automatic or pick one (e.g., 100).
   - Name: `ubuntu-docker`.
3. OS:
   - ISO Image: select your Ubuntu Server ISO.
   - Type: Linux.
4. System:
   - Leave default for a first build or enable QEMU/UEFI if you prefer.
5. Disks:
   - Bus/Device: `scsi`.
   - Disk size: 64–128 GB or more depending on your usage.
6. CPU:
   - 2–4 cores.
7. Memory:
   - 4–8 GB (more if you will run many containers).
8. Network:
   - Bridge: `vmbr0` (default bridge to your LAN).
9. Finish and start the VM.

### 5.3 Install Ubuntu in the VM

1. Open the VM console in Proxmox.
2. Follow the Ubuntu installer:
   - Language, keyboard.
   - Install Ubuntu Server.
   - Disk: use entire virtual disk.
   - Create a user, for example:
     - Username: `dev`.
   - Enable OpenSSH server.
3. Reboot into the installed system.

---

## 6. SSH Access and Basic Setup

### 6.1 Find VM IP Address

In the VM console:

```bash
ip a
```

Look for an `inet` address on `ens18` or similar, such as `192.168.1.101/24`.

### 6.2 SSH from Your Laptop

From your laptop/desktop:

```bash
ssh dev@192.168.1.101
```

Accept the host key and log in with your password.

### 6.3 System Updates

```bash
sudo apt update
sudo apt upgrade -y
sudo reboot
```

Reconnect via SSH after reboot.

---

## 7. Install Docker and Docker Compose

### 7.1 Install Docker Engine

On Ubuntu VM:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Add your user to the `docker` group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Verify:

```bash
docker run hello-world
```

### 7.2 Install Docker Compose (v2 CLI)

Docker on Ubuntu now includes Docker Compose as `docker compose`. Test:

```bash
docker compose version
```

---

## 8. Optional: Install Portainer

Portainer is a web UI to manage Docker containers.

```bash
docker volume create portainer_data

docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Access Portainer at:

- `https://<ubuntu-vm-ip>:9443`

---

## 9. Agentic AI Concepts (High Level)

Before deployment, understand key ideas:

- **LLM (Large Language Model)**: A model like GPT that can generate and understand text.
- **Tool use**: Agents can call tools (APIs, scripts) to interact with the outside world.
- **Agent**: A process that uses an LLM plus tools, memory, and a planning loop to take actions.
- **Multi-agent system**: Several agents collaborating, often with roles (planner, researcher, executor).

We will start with:

- A **single agent** that can:
  - Receive a task description.
  - Call a web API or run a local script.
  - Return a result.

Then you can expand to multi-agent workflows.

---

## 10. Choose an Agent Framework

You can pick any of the popular frameworks. Three common choices:

- **LangGraph**
- **CrewAI**
- **AutoGen**

For a first build, pick one and stay consistent through this guide. The steps below use a generic “Python agent service” pattern that works for all three with small adjustments.

---

## 11. Create a Project Structure

On your Ubuntu VM (or cloned from GitHub), create a directory:

```bash
mkdir -p ~/agent-lab
cd ~/agent-lab
```

Example structure:

```text
agent-lab/
  docker-compose.yml
  agent-service/
    Dockerfile
    requirements.txt
    app.py
```

---

## 12. Write a Minimal Agent Service (Python)

### 12.1 `requirements.txt`

Example for a generic agent with OpenAI-compatible client:

```text
fastapi
uvicorn[standard]
openai
langchain
```

Replace or extend with your chosen framework, for example `langgraph` or `crewai`.

### 12.2 `app.py` (Simple HTTP Agent)

```python
from fastapi import FastAPI
from pydantic import BaseModel
from openai import OpenAI
import os

app = FastAPI()

client = OpenAI(api_key=os.getenv("OPENAI_API_KEY"))

class Task(BaseModel):
    prompt: str

@app.post("/agent")
async def run_agent(task: Task):
    # Simple single-call agent (no tools) as a starting point
    response = client.chat.completions.create(
        model="gpt-4.1-mini",
        messages=[
            {"role": "system", "content": "You are a helpful coding assistant in a homelab."},
            {"role": "user", "content": task.prompt},
        ],
    )
    return {"result": response.choices[0].message.content}
```

This is intentionally minimal. Later you can:

- Add tools.
- Maintain state between calls.
- Use an agent framework abstraction instead of directly calling the API.

---

## 13. Dockerfile for the Agent

`agent-service/Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

ENV PYTHONUNBUFFERED=1

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8000

CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## 14. Docker Compose Configuration

`docker-compose.yml` in `agent-lab`:

```yaml
version: "3.9"

services:
  agent-service:
    build: ./agent-service
    container_name: agent-service
    ports:
      - "8000:8000"
    environment:
      OPENAI_API_KEY: "${OPENAI_API_KEY}"
    restart: unless-stopped
```

Create a `.env` file in `agent-lab` (never commit secrets):

```bash
cat <<EOF > .env
OPENAI_API_KEY=your-real-api-key-here
EOF
```

---

## 15. Build and Run the Agent Service

From `~/agent-lab`:

```bash
docker compose build
docker compose up -d
```

Check status:

```bash
docker ps
```

You should see `agent-service` running and listening on port 8000.

---

## 16. Test the Agent API

From your laptop (replace IP):

```bash
curl -X POST "http://192.168.1.101:8000/agent" \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Write a short Dockerfile that prints Hello World and explain each line."}'
```

You should receive JSON with a text response.

---

## 17. Evolving Towards Agentic Behavior

The minimal service above is a simple “stateless” chat wrapper. To make it **agentic**, incrementally add:

- **Tools**: Functions that the agent can call for:
  - Shell commands (carefully sandboxed).
  - HTTP APIs.
  - File operations inside a restricted directory.
- **Memory**: Store conversation context or task history.
- **Planning loop**: Let the model decide:
  - What to do next.
  - Which tool to call.
  - When to stop.

At the framework level this often means:

- Defining tools/functions.
- Writing a main loop that:
  - Sends the current state + tool schema to the model.
  - Parses tool calls, executes them, and feeds results back.
- Persisting state in a database or simple file store.

---

## 18. Example: Add a Simple Tool (Conceptual)

Here is a conceptual pattern (pseudo-code style) for adding a tool:

```python
import subprocess
from typing import List

def list_files() -> List[str]:
    files = subprocess.check_output(["ls", "-1"], text=True).splitlines()
    return files
```

Then expose `list_files` to your LLM using your agent framework’s tool mechanism.

> Note: For safety, start with read-only tools and limit directories.

---

## 19. Using the Agent Lab to Test Docker Containers

Your home lab is now ready to:

- Spin up new services as containers.
- Let the agent:
  - Generate or modify Dockerfiles.
  - Build images via CI or scripts.
  - Suggest or automate test sequences.

Example workflow:

1. Clone a containerized app into `~/projects/app1`.
2. Use your agent to:
   - Analyze its `Dockerfile`.
   - Propose improvements.
3. Build and run it with:

   ```bash
   docker compose build
   docker compose up -d
   ```

4. Capture logs:

   ```bash
   docker logs <container-name>
   ```

5. Feed relevant logs back to the agent for debugging help.

---

## 20. Monitoring and Maintenance

### 20.1 Basic Docker Commands

- List containers:

  ```bash
  docker ps
  ```

- View logs:

  ```bash
  docker logs agent-service
  ```

- Restart:

  ```bash
  docker restart agent-service
  ```

- Stop all:

  ```bash
  docker stop $(docker ps -q)
  ```

### 20.2 Backups

- Export your compose project:

  - Keep `agent-lab` in a Git repo.
  - Backup `.env` separately and securely.

- For Proxmox:
  - Use built-in backup jobs to snapshot the Ubuntu VM.

---

## 21. Security Basics

- Never expose Docker daemon (`/var/run/docker.sock`) to the internet.
- Keep Proxmox and Ubuntu updated.
- Use strong passwords and, ideally, SSH keys.
- Limit which services are accessible from outside your LAN.
- Consider:
  - A reverse proxy (e.g., Traefik, Nginx Proxy Manager).
  - Zero-trust access (e.g., Tailscale, Cloudflare Tunnel) if you want remote access.

---

## 22. Where to Go Next

Now that you have the basics:

- Swap the simple `app.py` for:
  - LangGraph, CrewAI, or AutoGen examples from their docs.
  - Multi-agent workflows (planner, researcher, executor).
- Add:
  - Vector database (e.g., Qdrant, Weaviate, Chroma) via Docker for retrieval.
  - Observability tools (Prometheus, Grafana, Loki) to monitor containers.
- Automate:
  - Use GitHub Actions to build images and deploy to your homelab via SSH.

---

## 23. Appendix: Common Commands Cheat Sheet

### Proxmox

- Restart a VM from CLI (on Proxmox host):

  ```bash
  qm reboot <VMID>
  ```

### Ubuntu VM

- Update system:

  ```bash
  sudo apt update && sudo apt upgrade -y
  ```

### Docker

- Build:

  ```bash
  docker compose build
  ```

- Up:

  ```bash
  docker compose up -d
  ```

- Down:

  ```bash
  docker compose down
  ```

- Remove unused:

  ```bash
  docker system prune
  ```

---

*End of Markdown guide (v1).*