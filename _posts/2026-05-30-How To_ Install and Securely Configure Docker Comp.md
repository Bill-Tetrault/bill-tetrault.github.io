---
layout: post
title: "How To: Install and Securely Configure Docker Compose with `/data/docker`, Global Networks, and a Default Management Stack"
author: "Bill Tetrault"
date: 2026-05-29
description: "Install Docker and Docker Compose, move Docker’s data root to `/data/docker`, define global Docker networks (`SERVERS`, `DMZ`, `INTERNAL`), and deploy a default non-root stack with Nginx (reverse proxy), Portainer, and Homarr."
tags: [Linux, Rocky, Docker, Tutorial, Guide, DevOps]
categories: [guides, tutorials, linux]
---
<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# How To: Install and Securely Configure Docker Compose with `/data/docker`, Global Networks, and a Default Management Stack

**Purpose:**
Install Docker and Docker Compose, move Docker’s data root to `/data/docker`, define global Docker networks (`SERVERS`, `DMZ`, `INTERNAL`), and deploy a default non-root stack with Nginx (reverse proxy), Portainer, and Homarr.

**Applies To:**
Rocky Linux 8/9, Ubuntu 22.04/24.04 (amd64)

**Last Updated:**
YYYY-MM-DD

**Difficulty:**
Intermediate–Advanced

***

## Overview

This extended guide adds:

- Global, named Docker bridge networks for segmentation: `SERVERS`, `DMZ`, `INTERNAL`
- How to make those networks reusable across multiple Compose projects using `external: true`[^1]
- A baseline non-root stack with:
    - Nginx as reverse proxy (in `DMZ` and `INTERNAL`)
    - Portainer for managing future stacks
    - Homarr for monitoring and dashboarding

Docker’s user-defined bridge networks isolate traffic by default; only containers on the same network can see each other, and the traffic stays on the host unless ports are published.[^2][^3]

> 💡 TIP: Think of each Docker network as a dedicated Layer 2 domain internal to the host. Use networks to express trust boundaries: DMZ (internet-facing), SERVERS (backend), INTERNAL (sidecar/tooling).[^4][^2]

***

## Prerequisites

All from the previous guide still apply:

- Docker Engine and `docker compose` installed
- Data root set to `/data/docker`
- Non-root per-service users and volumes under `/data/docker/volumes`

Additionally:

- Decide which services belong to:
    - `DMZ`: public-facing (Nginx reverse proxy, any internet-exposed service)
    - `SERVERS`: internal app services (databases, internal APIs)
    - `INTERNAL`: glue/monitoring/management (Portainer UI, Homarr, etc.)

***

## Step-by-Step Instructions

### 1. Create Global Docker Networks

Create user-defined bridge networks once at the host level.[^5][^2]

```bash
# DMZ: public-facing frontends, reverse proxy
docker network create DMZ

# SERVERS: backend services only
docker network create SERVERS

# INTERNAL: monitoring, admin, glue between others
docker network create INTERNAL
```

Verify:

```bash
docker network ls
```

You should see `DMZ`, `SERVERS`, `INTERNAL` with driver `bridge`.

> 💡 TIP: User-defined bridge networks provide built-in name-based service discovery and container-to-container isolation, unlike the default `bridge` network.[^2][^1]

***

### 2. Reference Global Networks from Compose Stacks

Compose can attach services to pre-existing networks by marking them as `external`.[^1]

Example top-level networks section (reusable pattern):

```yaml
networks:
  DMZ:
    external: true
  SERVERS:
    external: true
  INTERNAL:
    external: true
```

Any stack that includes the above `networks` section can attach services to these same shared networks, allowing cross-stack communication without recreating networks.[^1]

> ⚠️ WARNING: Do not redefine these networks with `driver: bridge` inside other Compose files; use `external: true` so they all point to the same global network objects.[^1]

***

### 3. Default Stack Layout and Users

Assume:

- `nginx` reverse proxy:
    - Networks: `DMZ`, `INTERNAL`
    - Non-root user ID: `10100`
- `portainer`:
    - Networks: `INTERNAL`
    - Non-root user ID: `10101`
- `homarr`:
    - Networks: `INTERNAL`
    - Non-root user ID: `10102`

Create host users:

```bash
sudo useradd -r -u 10100 -s /usr/sbin/nologin nginxrp
sudo useradd -r -u 10101 -s /usr/sbin/nologin portainer
sudo useradd -r -u 10102 -s /usr/sbin/nologin homarr
```

Create data directories:

```bash
sudo mkdir -p /data/docker/volumes/nginx/conf
sudo mkdir -p /data/docker/volumes/nginx/html
sudo mkdir -p /data/docker/volumes/portainer/data
sudo mkdir -p /data/docker/volumes/homarr/config

sudo chown -R 10100:10100 /data/docker/volumes/nginx
sudo chown -R 10101:10101 /data/docker/volumes/portainer
sudo chown -R 10102:10102 /data/docker/volumes/homarr
sudo chmod -R 750 /data/docker/volumes/nginx \
                  /data/docker/volumes/portainer \
                  /data/docker/volumes/homarr
```

On Rocky with SELinux enforcing:

```bash
sudo semanage fcontext -a -t container_file_t "/data/docker/volumes(/.*)?"
sudo restorecon -Rv /data/docker/volumes
```


***

### 4. Example Default Stack `docker-compose.yml`

Create a directory, for example `/data/docker/stacks/core`:

```bash
sudo mkdir -p /data/docker/stacks/core
cd /data/docker/stacks/core
```

Create `docker-compose.yml`:

```yaml
version: "3.9"

services:
  reverse-proxy:
    image: nginx:alpine
    container_name: reverse-proxy
    user: "10100:10100"
    networks:
      - DMZ
      - INTERNAL
    ports:
      - "80:8080"    # Nginx listens on 8080 in container, 80 on host
      - "443:8443"   # 8443 in container, 443 on host
    volumes:
      - /data/docker/volumes/nginx/conf:/etc/nginx/conf.d:ro
      - /data/docker/volumes/nginx/html:/usr/share/nginx/html:ro
    restart: unless-stopped

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    user: "10101:10101"
    networks:
      - INTERNAL
    # Bind UI only to LAN IP or localhost as desired
    ports:
      - "127.0.0.1:9443:9443"
    volumes:
      - /data/docker/volumes/portainer/data:/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: unless-stopped

  homarr:
    image: ghcr.io/ajnart/homarr:latest
    container_name: homarr
    user: "10102:10102"
    networks:
      - INTERNAL
    volumes:
      - /data/docker/volumes/homarr/config:/app/data
    restart: unless-stopped

networks:
  DMZ:
    external: true
  SERVERS:
    external: true
  INTERNAL:
    external: true
```

Key points:

- Nginx spans `DMZ` and `INTERNAL`, so it can receive public traffic and proxy to internal services across stacks.[^3][^4]
- Portainer and Homarr are on `INTERNAL` only and reachable via reverse proxy or LAN-restricted port bindings.
- All services run non-root via the `user:` directive; volumes are owned by the corresponding UIDs.[^6]

> ⚠️ WARNING: Mounting the Docker socket (`/var/run/docker.sock`) is powerful; keep Portainer on a restricted network and control who can reach the UI. Consider a docker-socket-proxy pattern if you want finer-grained control.[^4]

***

### 5. Wiring Other Stacks to the Global Networks

Any future stack can use the same global networks. For example, an “app” stack:

```yaml
version: "3.9"

services:
  myapp:
    image: myorg/myapp:latest
    container_name: myapp
    user: "10010:10010"
    networks:
      - SERVERS
      - INTERNAL
    volumes:
      - /data/docker/volumes/myapp:/app/data
    restart: unless-stopped

  mydb:
    image: postgres:16-alpine
    container_name: mydb
    user: "10011:10011"
    networks:
      - SERVERS
    volumes:
      - /data/docker/volumes/mydb:/var/lib/postgresql/data
    restart: unless-stopped

networks:
  DMZ:
    external: true
  SERVERS:
    external: true
  INTERNAL:
    external: true
```

Effects:

- `reverse-proxy` (core stack) and `myapp` (app stack) can communicate using service names on the shared `INTERNAL` network.[^1]
- `mydb` is only on `SERVERS`, so only containers on `SERVERS` (e.g., `myapp`) can reach it; nothing in `DMZ` can reach it directly.[^4]

Inside the Nginx config you can then use `myapp:port` as upstream, leveraging Docker’s internal DNS on the shared network.[^2][^1]

***

## Verification

1. **Check networks exist and are shared**

```bash
docker network ls
```

You should see `DMZ`, `SERVERS`, `INTERNAL` as `bridge` networks.
2. **Bring up the core stack**

```bash
cd /data/docker/stacks/core
docker compose up -d
docker compose ps
```

3. **Confirm network membership**

```bash
docker inspect reverse-proxy | grep -A3 '"DMZ"' 
docker inspect reverse-proxy | grep -A3 '"INTERNAL"'
docker inspect portainer | grep -A3 '"INTERNAL"'
```

Each service should show connectivity to the intended networks.[^5][^2]
4. **Test name resolution across stacks**

After starting another stack on `INTERNAL` or `SERVERS`, from inside `reverse-proxy`:

```bash
docker exec -it reverse-proxy sh
ping -c 3 myapp
```

You should see successful resolution and replies (assuming a `myapp` service on a shared network).[^2][^1]
5. **Verify non-root inside containers**

```bash
docker exec -it reverse-proxy id
docker exec -it portainer id
docker exec -it homarr id
```

UIDs must be non-zero and not `root`.[^7][^6]

***

## Troubleshooting

- **Containers cannot reach each other across stacks**
    - Ensure networks are declared as `external: true` and created once via `docker network create`.[^1]
    - Confirm both containers are attached to at least one common network with `docker inspect`.[^5][^2]
- **Reverse proxy cannot reach backend**
    - Check that backend service is on a shared network (`INTERNAL` or `SERVERS` as appropriate).
    - Verify the upstream name in Nginx matches the Compose service name.[^1]
- **Exposure to LAN/Internet too broad**
    - Use host-bound mappings like `127.0.0.1:PORT:PORT` for Portainer/Homarr and only proxy them via Nginx with authentication.[^8][^3]
    - Use your Fortigate/Ubiquiti to restrict inbound TCP 80/443 to desired source networks.
- **Port conflicts**
    - Align container listen ports and host bindings carefully:
        - Non-root inside containers -> use high ports in-container (e.g., 8080/8443)
        - Map to 80/443 on host if needed via `ports:` directive.[^9][^5]

***

## Security Considerations

- **Network segmentation:**
    - DMZ only for ingress/egress; no databases on DMZ.[^3][^4]
    - SERVERS only for internal services; no published ports.
    - INTERNAL for monitoring/admin; restrict access via firewall and reverse proxy.
- **Non-root containers everywhere:**
All services use `user:` or image `USER` to avoid root in containers.[^6][^7]
- **Principle of least privilege:**
    - Only join containers to networks they explicitly need.[^4]
    - Only expose ports required for external communication.[^10][^3]
- **Docker access control:**
Limit `docker` group membership; Portainer access is effectively root-equivalent.[^11]

> ⚠️ WARNING: Avoid `network_mode: host` unless absolutely necessary; it bypasses Docker’s network isolation.[^8][^10]

***

## Notes/Tips

- Keep a simple naming convention across stacks so Nginx upstream definitions stay predictable.
- Consider putting your core stack under version control, including Nginx configs, so changes to networks and UIDs are tracked.
- When you add new stacks, start by deciding which networks each service belongs to, then design ports and reverse proxy rules.

***

To download this as a .md file, copy the raw Markdown above into a file or use the copy button.

How would you like to map your Fortigate/Ubiquiti VLANs to these Docker networks (DMZ/SERVERS/INTERNAL) in your homelab, and do you want Nginx to terminate TLS for everything or only for some services?
<span style="display:none">[^12][^13]</span>

<div align="center">⁂</div>

[^1]: https://docs.docker.com/compose/how-tos/networking/

[^2]: https://docs.docker.com/engine/network/

[^3]: https://www.reddit.com/r/docker/comments/klj6vn/docker_containers_exposing_containers_to_the/

[^4]: https://www.reddit.com/r/selfhosted/comments/1sjivk5/how_to_properly_set_up_containers_networks/

[^5]: https://spacelift.io/blog/docker-networking

[^6]: https://lours.me/posts/compose-tip-014-non-root-users/

[^7]: https://oneuptime.com/blog/post/2026-01-16-docker-run-non-root-user/view

[^8]: https://forums.balena.io/t/reaching-other-container-with-network-mode-host/217325

[^9]: https://www.youtube.com/watch?v=WDQIv-Kd6hk

[^10]: https://developer.toradex.com/torizon/application-development/use-cases/networking-connectivity/how-to-setup-network-between-containers/

[^11]: https://www.reddit.com/r/docker/comments/ynug2i/running_docker_containers_is_equivalent_to_having/

[^12]: https://www.youtube.com/watch?v=itZ_x_nDBxU

[^13]: https://github.com/moby/moby/issues/30053

