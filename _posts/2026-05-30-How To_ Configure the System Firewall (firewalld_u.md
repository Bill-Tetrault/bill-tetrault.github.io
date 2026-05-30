---
layout: post
title: "How To: Configure the System Firewall (firewalld/ufw) with Common Troubleshooting Commands"
author: "Bill Tetrault"
date: 2026-05-29
description: "Quick-reference guide to configure host firewalls on Rocky Linux (firewalld) and Ubuntu (ufw) with practical troubleshooting commands for homelab use."
tags: [Linux, Rocky, Docker, Tutorial, Guide, DevOps]
categories: [guides, tutorials, linux]
---
<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# How To: Configure the System Firewall (firewalld/ufw) with Common Troubleshooting Commands

**Purpose:**
Quick-reference guide to configure host firewalls on Rocky Linux (firewalld) and Ubuntu (ufw) with practical troubleshooting commands for homelab use.

**Applies To:**

- Rocky Linux 8/9 (RHEL-family, `firewalld`)[^1]
- Ubuntu 20.04/22.04+ (`ufw`, Uncomplicated Firewall)[^2][^3]

**Last Updated:**

- `<fill-in-date>`

**Difficulty:**

- Intermediate (assumes Linux, TCP/IP, and basic security knowledge)

***

## Overview

For Rocky, the native firewall service is `firewalld` (zones, services, rich rules).[^1]
For Ubuntu, the typical host firewall is `ufw`, a front end to `iptables`/`nftables` that exposes simple “allow/deny” syntax.[^3]

This guide focuses on:

- Enabling and hardening the system firewall
- Allowing common services (SSH, HTTP, etc.)
- Inspecting rules and live connections
- Using standard troubleshooting commands (ping, nc, curl, ss, tcpdump) to debug connectivity[^4][^5]

***

## Prerequisites

1. Root or sudo access on the target host.
2. SSH or console access (ideally via out-of-band if you are modifying SSH rules).
3. Package updates applied (`dnf update` / `apt upgrade`) to ensure current firewall components.[^3]

> ⚠️ WARNING: When changing firewall rules on remote systems, always ensure you have a persistent session and that SSH is explicitly allowed before applying restrictive policies, or you risk locking yourself out.[^2][^3]

***

## Step-by-Step Instructions

### 1. Confirm which firewall you are using

On Rocky Linux:

```bash
sudo firewall-cmd --state
sudo systemctl status firewalld
```

On Ubuntu:

```bash
sudo ufw status verbose
sudo systemctl status ufw
```

If `firewalld` or `ufw` are inactive, you will see them reported as `not running` or `inactive`.[^1]

***

### 2. Enable and secure the firewall

#### Rocky Linux: firewalld basics

1. Enable and start `firewalld`:
```bash
sudo systemctl enable --now firewalld
sudo firewall-cmd --state
```

2. Check default zone and active interfaces:
```bash
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --list-all
```

Zones group interfaces/services by trust level (e.g., `public`, `internal`, `dmz`).[^3][^1]

> 💡 TIP: In a homelab, map management networks to a more trusted zone (e.g., `internal`) and WAN-facing or guest networks to restrictive zones like `public`.[^6][^7]

#### Ubuntu: ufw basics

1. Enable `ufw`:
```bash
sudo ufw enable
```

2. Lock in a default deny inbound posture:
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

3. Check status:
```bash
sudo ufw status verbose
```

`ufw` outputs the default policy plus per-rule state.[^2][^3]

***

### 3. Allow essential services (SSH, HTTP, HTTPS)

#### Rocky Linux (firewalld)

Allow SSH, HTTP, HTTPS in the default zone:

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
sudo firewall-cmd --list-services
```

To allow a specific TCP port (e.g., 8080):

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

Services correspond to XML definitions under `/usr/lib/firewalld/services/` and include standard port/protocol mapping.[^1]

#### Ubuntu (ufw)

Allow SSH, HTTP, HTTPS:

```bash
sudo ufw allow ssh            # typically port 22/tcp
sudo ufw allow http           # port 80/tcp
sudo ufw allow https          # port 443/tcp
sudo ufw status numbered
```

To allow a specific TCP port (e.g., 8080):

```bash
sudo ufw allow 8080/tcp
```

To restrict a rule to a source network:

```bash
sudo ufw allow from 192.168.10.0/24 to any port 22 proto tcp
```

This is handy for homelab management networks.[^3]

***

### 4. Add more advanced/firewall-specific rules

#### firewalld: zones and rich rules

Assign an interface to a specific zone (e.g., `eth1` to `internal`):

```bash
sudo firewall-cmd --permanent --zone=internal --change-interface=eth1
sudo firewall-cmd --reload
sudo firewall-cmd --zone=internal --list-all
```

Add a rich rule (e.g., allow SSH only from 10.0.0.0/24):

```bash
sudo firewall-cmd --permanent --zone=public \
  --add-rich-rule='rule family="ipv4" source address="10.0.0.0/24" service name="ssh" accept'
sudo firewall-cmd --reload
```

Rich rules allow more granular matching (source, destination, logging, etc.).[^1]

#### ufw: application profiles and rules

List known application profiles:

```bash
sudo ufw app list
```

Allow a profile (e.g., OpenSSH):

```bash
sudo ufw allow OpenSSH
```

Delete a misconfigured rule by its number:

```bash
sudo ufw status numbered
sudo ufw delete <rule-number>
```

This prevents rule sprawl and keeps the rule set manageable.[^3]

***

### 5. Logging and visibility

#### firewalld logging

Enable logging of denied packets (RHEL-family):

```bash
sudo firewall-cmd --set-log-denied=all
```

Logs typically appear in:

```bash
sudo journalctl -u firewalld
sudo journalctl -k | grep -i "REJECT"
```

This is useful when you suspect the firewall is dropping traffic.[^5]

#### ufw logging

Enable ufw logging:

```bash
sudo ufw logging on
```

Inspect logs:

```bash
sudo grep UFW /var/log/syslog
# or on some systems:
sudo grep UFW /var/log/kern.log
```

Log entries show whether packets were allowed or denied, and from which source/destination.[^5]

***

## Verification

### 1. Basic connectivity checks

From a remote host towards your Linux box:

```bash
ping <target-ip>
traceroute <target-ip>         # or tracepath <target-ip>
```

On the target itself, verify listening services and bound ports:

```bash
sudo ss -tulpen
```

`ss` shows listening ports, associated processes, and which address families are enabled.[^4]

> 💡 TIP: When debugging “service down” vs “firewall blocking,” start with `ss` or `netstat` to confirm the service is actually listening, then test connectivity (ping, nc, curl), then inspect firewall logs.[^4][^5]

### 2. Port testing with netcat and curl

From a remote client:

```bash
nc -vz <target-ip> 22
nc -vz <target-ip> 80
```

For HTTP/S:

```bash
curl -v http://<target-ip>/
curl -vk https://<target-ip>/
```

If `nc`/`curl` time out or are refused, check firewall rules and host-based service status.[^5][^4]

### 3. Packet-level verification (tcpdump)

On the firewall host:

```bash
sudo tcpdump -ni eth0 port 22
```

If packets arrive on the interface but the session never establishes, the firewall or local service is likely the culprit.[^5]

***

## Troubleshooting

### 1. Standard triage steps

Work through these in order:

1. Confirm interface IPs and routes:
```bash
ip addr show
ip route show
```

2. Check firewall status and rules:
```bash
# Rocky
sudo firewall-cmd --state
sudo firewall-cmd --list-all

# Ubuntu
sudo ufw status verbose
```

3. Confirm service is listening:
```bash
sudo systemctl status sshd
sudo ss -tulpen | grep :22
```

4. Review logs for denies:
```bash
# firewalld/Kernel
sudo journalctl -u firewalld
sudo journalctl -k | grep -i "REJECT"

# ufw
sudo grep UFW /var/log/syslog
```

These steps align with typical firewall troubleshooting guidance: log inspection, debugging tools, SNMP/CPU checks, and connectivity tests.[^8][^5]

### 2. Temporarily relax rules (with caution)

If you strongly suspect the firewall is the issue, and you have out-of-band access:

On firewalld:

```bash
sudo firewall-cmd --set-default-zone=trusted
# test connectivity, then revert:
sudo firewall-cmd --set-default-zone=public
```

On ufw:

```bash
sudo ufw disable
# test, then re-enable and fix rules:
sudo ufw enable
```

> ⚠️ WARNING: Never leave a host with a fully disabled firewall on untrusted networks. Use temporary relaxations only for controlled tests and revert immediately after.[^2][^3]

### 3. SELinux considerations (Rocky)

If a service is listening and allowed in firewalld but traffic is still blocked, check SELinux:

```bash
getenforce
sudo ausearch -m AVC,USER_AVC -ts recent
```

Relabel or adjust SELinux policy using `semanage` and `restorecon` rather than disabling SELinux, to retain proper security posture.[^1]

***

## Security Considerations

1. Principle of least privilege: Only open ports/services that are strictly needed, and restrict by source networks where possible.[^7][^6]
2. Default deny: Keep default inbound policy as deny and add explicit allow rules (typical best practice on both network and host firewalls).[^6][^3]
3. Logging and review: Enable firewall logging for drops and periodically review logs to detect misconfigurations and suspicious activity.[^8][^5]
4. Change control: Back up configurations and document rule changes so you can roll back after testing.[^6][^2]

> 💡 TIP: For a homelab, treat “lab-only” networks as lower trust than management networks, and still adhere to strict rules; it keeps habits aligned with production-grade practices.[^7][^6]

***

## Notes/Tips

- Pair host firewalls with upstream controls (Fortigate, etc.) to catch misconfigurations earlier in the path.[^9][^8]
- For complex multi-hop issues, capture on both ends and in the middle (e.g., hypervisor bridge, router) to see exactly where traffic stops.[^5]
- Periodically audit firewall rules and prune unused entries to avoid rule bloat and unintended access.[^10]

To download this as a .md file, copy the raw Markdown above into a file or use the copy button.

What environment do you want to start with for hands-on examples in your homelab—Rocky (firewalld) hosts, Ubuntu (ufw) hosts, or both?
<span style="display:none">[^11][^12][^13][^14][^15]</span>

<div align="center">⁂</div>

[^1]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld_firewall-packet-filters

[^2]: https://www.securitymetrics.com/blog/how-configure-firewall-5-steps

[^3]: https://www.paloaltonetworks.com/cyberpedia/what-is-firewall-configuration

[^4]: https://www.site24x7.com/blog/common-firewall-issues-in-the-datacenter-and-how-to-resolve-them-1489611660726

[^5]: https://www.tufin.com/blog/firewall-troubleshooting-scenarios-unraveling-common-issues-solutions

[^6]: https://www.fortinet.com/resources/cyberglossary/firewall-configuration

[^7]: https://www.checkpoint.com/cyber-hub/network-security/what-is-firewall/firewall-settings-and-how-to-configure-them/

[^8]: https://www.paloaltonetworks.com/cyberpedia/how-to-troubleshoot-common-firewall-issues

[^9]: https://docs.fortinet.com/document/fortigate/8.0.0/cli-troubleshooting-cheat-sheet/420966/cli-troubleshooting-cheat-sheet

[^10]: https://www.firewalls.com/blog/what-are-the-best-practices-for-configuring-a-network-firewall/

[^11]: https://www.cisco.com/site/us/en/learn/topics/small-business/how-to-setup-a-firewall.html

[^12]: https://www.youtube.com/watch?v=gklzTf2suEE

[^13]: https://support.ucsd.edu/its?id=kb_article_view\&sysparm_article=KB0030091

[^14]: https://itsecworks.com/2011/07/18/fortigate-basic-troubleshooting-commands/

[^15]: https://www.youtube.com/watch?v=Zeps86aFv3s

