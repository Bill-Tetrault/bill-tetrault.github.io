---
layout: post
title: "Docker Homelab Beginner's Guide"
author: "Bill Tetrault"
date: 2025-09-18
description: "Complete Beginner's Guide to Docker Services on Proxmox"
tags: [Linux, Rocky, Docker, Tutorial, Guide, DevOps]
categories: [guides, tutorials, linux]
---
#### Created using Perplexity AI

# Rocky Linux 9 Docker Deployment Guide
### Complete Beginner's Guide to Docker Services on Proxmox

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [VM Hardware Recommendations](#vm-hardware-recommendations)
3. [Rocky Linux 9 VM Setup](#rocky-linux-9-vm-setup)
4. [System Preparation](#system-preparation)
5. [Docker Installation](#docker-installation)
6. [User and Group Configuration](#user-and-group-configuration)
7. [Firewall Configuration](#firewall-configuration)
8. [Directory Structure Setup](#directory-structure-setup)
9. [Docker Compose Configuration](#docker-compose-configuration)
10. [Service Configuration](#service-configuration)
11. [Deployment](#deployment)
12. [Post-Deployment Tasks](#post-deployment-tasks)
13. [Troubleshooting](#troubleshooting)
14. [Tips and Reminders](#tips-and-reminders)

## Prerequisites

Before beginning this deployment, ensure you have:
- **Proxmox VE 7.0+** installed and functional
- **Administrative access** to Proxmox
- **Rocky Linux 9 ISO** downloaded
- **Basic familiarity** with Linux command line
- **Network connectivity** for package downloads
- **Sufficient storage** for VM and container data

## VM Hardware Recommendations

### Minimum Specifications
- **CPU**: 4 vCores (host type recommended for Rocky Linux 9)
- **Memory**: 8GB RAM
- **Storage**: 60GB disk space
- **Network**: 1 NIC with internet access

### Recommended Specifications
- **CPU**: 6-8 vCores (host or haswell+ type)
- **Memory**: 12-16GB RAM
- **Storage**: 100GB+ SSD storage
- **Network**: 1 Gbit NIC

### Storage Breakdown
- **OS**: ~20GB
- **Docker images**: ~15GB
- **Application data** (/data): ~25GB
- **Logs and backups**: ~10GB
- **Free space buffer**: ~30GB

⚠️ **Important**: Rocky Linux 9 requires x86-64-v2 CPU features. In Proxmox, use "host" CPU type or ensure your CPU type supports AVX2 instructions.

## Rocky Linux 9 VM Setup

### Proxmox VM Creation
1. **Create new VM** in Proxmox
2. **OS**: Linux (6.x/2.6 Kernel)
3. **CPU**: Set to "host" type with 4+ cores
4. **Memory**: 8GB minimum
5. **Storage**: 60GB+ on fast storage
6. **Network**: Default bridge with DHCP or static IP

### Rocky Linux 9 Installation
1. **Boot from ISO** and select minimal installation
2. **Configure network** with static IP (recommended)
3. **Create user account** with sudo privileges
4. **Complete installation** and reboot

## System Preparation

### Update System
```bash
# Update all packages
sudo dnf update -y

# Reboot to ensure kernel updates are active
sudo reboot
```

### Install Essential Packages
```bash
# Install required utilities
sudo dnf install -y epel-release
sudo dnf install -y curl wget git nano vim htop tree

# Install development tools (optional but useful)
sudo dnf groupinstall -y "Development Tools"
```

### Set Timezone (Optional)
```bash
# Set your timezone
sudo timedatectl set-timezone America/Chicago
# Verify
timedatectl
```

## Docker Installation

### Remove Conflicting Packages
```bash
# Remove podman and buildah if installed
sudo dnf remove -y podman buildah
```

### Add Docker Repository
```bash
# Install dnf config manager
sudo dnf install -y dnf-utils

# Add Docker repository
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

### Install Docker
```bash
# Install Docker and related packages
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# If you encounter containerd.io conflicts, use:
# sudo dnf install docker-ce --allowerasing -y
```

### Start and Enable Docker
```bash
# Start Docker service
sudo systemctl start docker

# Enable Docker to start on boot
sudo systemctl enable docker

# Verify Docker is running
sudo systemctl status docker
```

### Verify Installation
```bash
# Check Docker version
docker --version

# Check Docker Compose version
docker compose version

# Test Docker installation
sudo docker run hello-world
```

## User and Group Configuration

### Add User to Docker Group
```bash
# Add current user to docker group
sudo usermod -aG docker $USER

# Apply group changes without logout
newgrp docker

# Verify group membership
groups $USER
```

### Create Service User (Optional but Recommended)
```bash
# Create a dedicated user for Docker services
sudo useradd -r -s /bin/false -d /data dockersvc

# Add dockersvc user to docker group
sudo usermod -aG docker dockersvc
```

### Set Proper Permissions
```bash
# Ensure docker.sock has correct permissions
sudo chmod 666 /var/run/docker.sock

# Fix ownership if needed
sudo chown root:docker /var/run/docker.sock
```

## Firewall Configuration

### Basic Firewall Setup
```bash
# Start and enable firewalld
sudo systemctl start firewalld
sudo systemctl enable firewalld
```

### Configure Docker Firewall Integration
```bash
# Add docker0 interface to trusted zone
sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0

# Enable masquerading for Docker networking
sudo firewall-cmd --permanent --zone=public --add-masquerade

# Create a custom zone for Docker services (optional)
sudo firewall-cmd --permanent --new-zone=docker-services
sudo firewall-cmd --permanent --zone=docker-services --set-target=ACCEPT
```

### Open Required Ports
```bash
# Homepage (Port 3000)
sudo firewall-cmd --permanent --zone=public --add-port=3000/tcp

# OpenSpeedTest (Ports 3001-3002)
sudo firewall-cmd --permanent --zone=public --add-port=3001/tcp
sudo firewall-cmd --permanent --zone=public --add-port=3002/tcp

# Portainer (Port 9000)
sudo firewall-cmd --permanent --zone=public --add-port=9000/tcp

# Nginx Proxy Manager (Ports 80, 81, 443)
sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
sudo firewall-cmd --permanent --zone=public --add-port=81/tcp
sudo firewall-cmd --permanent --zone=public --add-port=443/tcp

# Pi-hole (Ports 53, 67, 80-alt)
sudo firewall-cmd --permanent --zone=public --add-port=53/tcp
sudo firewall-cmd --permanent --zone=public --add-port=53/udp
sudo firewall-cmd --permanent --zone=public --add-port=67/udp
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp

# Grafana (Port 3000-alt)
sudo firewall-cmd --permanent --zone=public --add-port=3001/tcp

# Prometheus (Port 9090)
sudo firewall-cmd --permanent --zone=public --add-port=9090/tcp

# GitLab (Ports 8081, 2424)
sudo firewall-cmd --permanent --zone=public --add-port=8081/tcp
sudo firewall-cmd --permanent --zone=public --add-port=2424/tcp

# SSH (should already be open)
sudo firewall-cmd --permanent --zone=public --add-service=ssh

# Apply all firewall changes
sudo firewall-cmd --reload
```

### Verify Firewall Rules
```bash
# Check active zones
sudo firewall-cmd --get-active-zones

# List all rules for public zone
sudo firewall-cmd --zone=public --list-all

# Check if Docker integration is working
sudo firewall-cmd --zone=trusted --list-all
```

## Directory Structure Setup

### Create Main Data Directory
```bash
# Create the main /data directory
sudo mkdir -p /data

# Set ownership
sudo chown $USER:$USER /data

# Set permissions
sudo chmod 755 /data
```

### Create Service Directories
```bash
# Create directories for each service
mkdir -p /data/homepage/config
mkdir -p /data/openspeedtest
mkdir -p /data/portainer
mkdir -p /data/nginx-proxy-manager/data
mkdir -p /data/nginx-proxy-manager/letsencrypt
mkdir -p /data/pihole/config
mkdir -p /data/pihole/dnsmasq
mkdir -p /data/grafana/data
mkdir -p /data/prometheus/config
mkdir -p /data/prometheus/data
mkdir -p /data/gitlab/config
mkdir -p /data/gitlab/logs
mkdir -p /data/gitlab/data

# Create shared directories
mkdir -p /data/logs
mkdir -p /data/backups
mkdir -p /data/compose
```

### Set Directory Permissions
```bash
# Set proper ownership for specific services
sudo chown -R 472:472 /data/grafana/data  # Grafana user
sudo chown -R 65534:65534 /data/prometheus/data  # Prometheus user
sudo chown -R 1000:1000 /data/pihole  # Pi-hole default user

# Ensure main user can access all directories
sudo chown -R $USER:$USER /data/homepage
sudo chown -R $USER:$USER /data/openspeedtest
sudo chown -R $USER:$USER /data/portainer
sudo chown -R $USER:$USER /data/nginx-proxy-manager
sudo chown -R $USER:$USER /data/gitlab
```

## Docker Compose Configuration

### Create Main Docker Compose File
```bash
# Navigate to compose directory
cd /data/compose

# Create the main docker-compose.yml file
nano docker-compose.yml
```

### Complete Docker Compose Configuration
```yaml
version: '3.8'

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  portainer_data:
  prometheus_data:
  grafana_data:
  gitlab_data:
  gitlab_logs:
  gitlab_config:

services:
  # Homepage - Dashboard
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - "3000:3000"
    volumes:
      - /data/homepage/config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - PUID=1000
      - PGID=1000
    networks:
      - frontend

  # OpenSpeedTest - Network Speed Testing
  openspeedtest:
    image: openspeedtest/latest
    container_name: openspeedtest
    restart: unless-stopped
    ports:
      - "3001:3000"  # HTTP
      - "3002:3001"  # HTTPS
    volumes:
      - /data/openspeedtest:/var/log/nginx
    networks:
      - frontend

  # Portainer - Docker Management
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - frontend

  # Nginx Proxy Manager - Reverse Proxy
  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - /data/nginx-proxy-manager/data:/data
      - /data/nginx-proxy-manager/letsencrypt:/etc/letsencrypt
    environment:
      - DB_SQLITE_FILE=/data/database.sqlite
    networks:
      - frontend

  # Pi-hole - DNS Ad Blocker
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "67:67/udp"
      - "8080:80/tcp"
    volumes:
      - /data/pihole/config:/etc/pihole
      - /data/pihole/dnsmasq:/etc/dnsmasq.d
    environment:
      - TZ=America/Chicago
      - WEBPASSWORD=changeme123
      - DNS1=1.1.1.1
      - DNS2=1.0.0.1
    cap_add:
      - NET_ADMIN
    dns:
      - 127.0.0.1
      - 1.1.1.1
    networks:
      - frontend

  # Prometheus - Metrics Collection
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - /data/prometheus/config:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'
    networks:
      - backend

  # Grafana - Metrics Visualization
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3003:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=changeme123
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource
    networks:
      - backend
      - frontend

  # GitLab - Git Repository and CI/CD
  gitlab:
    image: gitlab/gitlab-ce:latest
    container_name: gitlab
    restart: unless-stopped
    hostname: 'gitlab.local'
    ports:
      - "8081:80"
      - "2424:22"
    volumes:
      - gitlab_config:/etc/gitlab
      - gitlab_logs:/var/log/gitlab
      - gitlab_data:/var/opt/gitlab
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.local:8081'
        gitlab_rails['gitlab_shell_ssh_port'] = 2424
        gitlab_rails['initial_root_password'] = 'changeme123'
    shm_size: '256m'
    networks:
      - frontend
```

## Service Configuration

### Prometheus Configuration
```bash
# Create Prometheus configuration
cat > /data/prometheus/config/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['host.docker.internal:9100']

  - job_name: 'docker'
    static_configs:
      - targets: ['host.docker.internal:9323']
EOF
```

### Homepage Configuration
```bash
# Create basic Homepage configuration
cat > /data/homepage/config/settings.yaml << 'EOF'
title: Home Dashboard
headerStyle: clean
statusStyle: dot
layout:
  - Services:
      style: row
      columns: 4
      
providers:
  docker:
    endpoint: unix:///var/run/docker.sock
EOF

# Create services configuration
cat > /data/homepage/config/services.yaml << 'EOF'
- Infrastructure:
    - Portainer:
        href: http://{{HOSTNAME}}:9000
        description: Docker Management
        icon: portainer.png
        server: my-docker
        container: portainer
        
    - Pi-hole:
        href: http://{{HOSTNAME}}:8080/admin
        description: DNS Ad Blocker
        icon: pi-hole.png
        server: my-docker
        container: pihole
        
    - Nginx Proxy Manager:
        href: http://{{HOSTNAME}}:81
        description: Reverse Proxy
        icon: nginx-proxy-manager.png
        server: my-docker
        container: nginx-proxy-manager

- Monitoring:
    - Grafana:
        href: http://{{HOSTNAME}}:3003
        description: Metrics Visualization  
        icon: grafana.png
        server: my-docker
        container: grafana
        
    - Prometheus:
        href: http://{{HOSTNAME}}:9090
        description: Metrics Collection
        icon: prometheus.png
        server: my-docker
        container: prometheus

- Development:
    - GitLab:
        href: http://{{HOSTNAME}}:8081
        description: Git Repository
        icon: gitlab.png
        server: my-docker
        container: gitlab
        
    - Speed Test:
        href: http://{{HOSTNAME}}:3001
        description: Network Speed Test
        icon: speedtest-tracker.png
        server: my-docker
        container: openspeedtest
EOF

# Create Docker configuration
cat > /data/homepage/config/docker.yaml << 'EOF'
my-docker:
  host: unix:///var/run/docker.sock
EOF
```

## Deployment

### Deploy Services
```bash
# Navigate to compose directory
cd /data/compose

# Deploy all services
docker compose up -d

# Check deployment status
docker compose ps

# View logs for any issues
docker compose logs -f
```

### Verify Services
```bash
# Check all containers are running
docker ps

# Check specific service logs
docker compose logs homepage
docker compose logs pihole
docker compose logs grafana
```

## Post-Deployment Tasks

### Configure Individual Services

#### Pi-hole Setup
```bash
# Access Pi-hole admin interface at http://YOUR_IP:8080/admin
# Default password: changeme123
# Change the password:
docker exec -it pihole pihole -a -p
```

#### Nginx Proxy Manager Setup
```bash
# Access at http://YOUR_IP:81
# Default credentials:
# Email: admin@example.com
# Password: changeme
```

#### Grafana Setup
```bash
# Access at http://YOUR_IP:3003
# Default credentials:
# Username: admin
# Password: changeme123

# Add Prometheus data source: http://prometheus:9090
```

#### GitLab Setup
```bash
# GitLab will take several minutes to initialize
# Access at http://YOUR_IP:8081
# Username: root
# Password: changeme123
```

### Security Hardening
```bash
# Change all default passwords immediately after deployment
# Update firewall rules to restrict access as needed
# Consider using Nginx Proxy Manager for SSL termination
# Regularly update container images

# Example: Update all containers
cd /data/compose
docker compose pull
docker compose up -d
```

## Troubleshooting

### Common Issues and Solutions

#### Container Won't Start
```bash
# Check logs
docker compose logs SERVICE_NAME

# Check resource usage
docker stats

# Restart specific service
docker compose restart SERVICE_NAME
```

#### Port Conflicts
```bash
# Check what's using a port
sudo netstat -tulpn | grep :PORT

# Stop conflicting service
sudo systemctl stop SERVICE_NAME
```

#### Permission Issues
```bash
# Fix data directory permissions
sudo chown -R $USER:$USER /data/SERVICE_NAME

# Fix Docker socket permissions
sudo chmod 666 /var/run/docker.sock
```

#### Firewall Issues
```bash
# Check if ports are open
sudo firewall-cmd --zone=public --list-ports

# Add missing port
sudo firewall-cmd --permanent --zone=public --add-port=PORT/tcp
sudo firewall-cmd --reload
```

#### Docker Network Issues
```bash
# Restart Docker service
sudo systemctl restart docker

# Recreate networks
docker compose down
docker compose up -d
```

### Resource Monitoring
```bash
# Monitor system resources
htop

# Monitor Docker resources
docker stats

# Check disk usage
df -h
du -sh /data/*
```

## Tips and Reminders

### Regular Maintenance
- **Update containers monthly**: `docker compose pull && docker compose up -d`
- **Monitor disk usage**: Docker logs and images can consume significant space
- **Backup configurations**: Regular backups of `/data` directory
- **Security updates**: Keep Rocky Linux updated with `dnf update`

### Performance Optimization
- **Use SSD storage** for better performance
- **Monitor memory usage** - increase VM memory if needed
- **Consider resource limits** in docker-compose.yml for production use
- **Use caching** where possible (Redis for caching layer)

### Security Best Practices
- **Change all default passwords** before production use
- **Use strong passwords** for all services
- **Enable SSL/TLS** via Nginx Proxy Manager
- **Restrict firewall rules** to necessary ports only
- **Regular security updates** for both OS and containers
- **Monitor logs** for suspicious activity

### Backup Strategy
```bash
# Create backup script
cat > /data/backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/data/backups/$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup configurations
tar -czf $BACKUP_DIR/configs.tar.gz /data/*/config/
tar -czf $BACKUP_DIR/compose.tar.gz /data/compose/

# Backup Docker volumes
docker compose -f /data/compose/docker-compose.yml stop
tar -czf $BACKUP_DIR/volumes.tar.gz /var/lib/docker/volumes/
docker compose -f /data/compose/docker-compose.yml start

echo "Backup completed: $BACKUP_DIR"
EOF

chmod +x /data/backup.sh
```

### Useful Commands
```bash
# Quick service restart
docker compose restart SERVICE_NAME

# View all logs
docker compose logs -f

# Update specific service
docker compose pull SERVICE_NAME
docker compose up -d SERVICE_NAME

# Clean up unused resources
docker system prune -a

# Monitor resources
watch docker stats
```

This completes your Rocky Linux 9 Docker deployment guide. All services should now be accessible and functional. Remember to change default passwords and implement proper security measures before using in production environments.