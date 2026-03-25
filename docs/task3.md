# Task 3 — Firewall and Network Security

## Goal

Configure UFW to implement a **default deny** security model — block all incoming traffic by default and only allow what is necessary.

| Port | Service | Allowed |
|------|---------|---------|
| 2222 | SSH | ✅ |
| 80 | HTTP | ✅ |
| 443 | HTTPS | ✅ |
| Everything else | — | ❌ Blocked |

> **Warning:** Always allow SSH **before** enabling the firewall. Enabling UFW without allowing SSH first will immediately lock you out of the VM.

## Concept: Default Deny Security Model

```
Before UFW:                After UFW:
All ports open      →      DENY all incoming (default)
                           ALLOW 2222/tcp  (SSH)
                           ALLOW 80/tcp    (HTTP)
                           ALLOW 443/tcp   (HTTPS)
```

Block first, allow only what is needed — standard practice on production servers.

## Configuration

```bash
# Set default policies
sudo ufw default deny incoming       # Block everything unless explicitly allowed
sudo ufw default allow outgoing      # VM can reach the internet normally

# Allow required ports (do this BEFORE enabling)
sudo ufw allow 2222/tcp              # SSH
sudo ufw allow 80/tcp                # HTTP
sudo ufw allow 443/tcp               # HTTPS

# Enable the firewall
sudo ufw enable                      # Type 'y' when prompted

# Verify
sudo ufw status verbose
```

Expected output:

```
Status: active

To          Action      From
--          ------      ----
2222/tcp    ALLOW IN    Anywhere
80/tcp      ALLOW IN    Anywhere
443/tcp     ALLOW IN    Anywhere
```

## Logging

Enabled UFW logging at medium level to monitor traffic:

```bash
sudo ufw logging medium
```

Logs verified at `/var/log/ufw.log` — showed allowed outbound connections to Azure infrastructure, confirming firewall monitoring is active.

## What This Protects Against

- Port scanning by automated bots
- Exploit attempts on unused open ports
- Worm propagation through unnecessary services
- Unauthorized connections to internal services
