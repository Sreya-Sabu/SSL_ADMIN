How does a VPN work? No jargon, we promise!

While you’re online, your device constantly sends and receives data, such as your IP address, browsing data, login credentials, payment details, search queries, and much more.

While some of this data is protected by various measures even if you don't use a VPN (for example, by HTTPS encryption for websites), a VPN adds a much-needed extra layer of security and privacy.

When you turn on a VPN, it begins feeding all of that network data through a secret tunnel of sorts. During this time, it wraps your data in a layer of protective code (encryption) so that it appears to be a string of random symbols that can’t be understood.

The VPN tunnel and encryption of your data are two important aspects of how a VPN protects your privacy. Let’s take a closer look at both.

# WireGuard VPN Server Setup

A step-by-step guide to installing, configuring, and running a WireGuard VPN server on Ubuntu with multiple clients.

---

## Overview

This project sets up a WireGuard VPN server on a Linux VM. It generates cryptographic key pairs for the server and each client, configures IP routing and firewall rules, and produces importable `.conf` profiles for each VPN user.

**Network layout:**
```
Internet
   │
   ▼
VM / WireGuard server — 10.0.0.1
   ├── User 1 — 10.0.0.2
   └── User 2 — 10.0.0.3
```

---

## Configuration Summary

| Setting | Value |
|---|---|
| VPN protocol | WireGuard (UDP) |
| Listen port | 51820 |
| VPN subnet | 10.0.0.0/24 |
| Server IP | 10.0.0.1 |
| Key type | Curve25519 |
| Auto-start | Enabled (systemd) |

---

## Server Setup

### Step 1 — Install WireGuard

Updates the package list and installs WireGuard — a modern, lightweight VPN protocol that creates an encrypted tunnel between devices.

```bash
sudo apt update && sudo apt install wireguard -y
```

---

### Step 2 — Generate server key pair

Creates a private key (the server's secret identity) and derives a public key from it. The public key is shared with clients; the private key never leaves the server.

```bash
sudo mkdir -p /etc/wireguard && cd /etc/wireguard
sudo wg genkey | sudo tee server_private.key | sudo wg pubkey | sudo tee server_public.key
```

---

### Step 3 — Lock down the private key

Sets file permissions to `600` — readable and writable by root only. This ensures no other system user can ever read the server's private key.

```bash
sudo chmod 600 server_private.key
```

---

### Step 4 — Create the VPN config file

Defines the VPN interface. The server sits at `10.0.0.1` and listens on UDP port `51820`. The `PostUp`/`PostDown` rules use `iptables MASQUERADE` so VPN clients can reach the internet through the server.

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address    = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <SERVER_PRIVATE_KEY>

PostUp   = ufw route allow in on wg0 out on eth0
PostUp   = iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = ufw route delete allow in on wg0 out on eth0
PostDown = iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

---

### Step 5 — Enable IP forwarding

By default Linux drops packets not destined for itself. Uncommenting `net.ipv4.ip_forward=1` tells the kernel to forward packets between interfaces — required for routing VPN traffic to the internet.

```bash
sudo nano /etc/sysctl.conf   # uncomment: net.ipv4.ip_forward=1
sudo sysctl -p               # apply immediately
```

---

### Step 6 — Open the VPN port in the firewall

Allows inbound UDP traffic on port `51820` through UFW so clients can reach the VPN server. All other ports remain blocked.

```bash
sudo ufw allow 51820/udp
```

---

### Step 7 — Start the VPN interface

Brings up the `wg0` network interface, loads the config, applies the firewall rules, and begins listening for connections. Run `sudo wg` afterwards to confirm it is active.

```bash
sudo wg-quick up wg0
sudo wg          # verify interface is live
```

---

### Step 8 — Enable auto-start on reboot

Registers the VPN as a systemd service so it restarts automatically whenever the server reboots — essential for a production environment.

```bash
sudo systemctl enable wg-quick@wg0
```

---

## Adding VPN Clients

### Step 9 — Generate a key pair for each user

Each VPN client needs its own private/public key pair. Repeat this for every user — the private key goes in their device config, the public key is registered on the server.

```bash
sudo wg genkey | sudo tee user1_private.key | sudo wg pubkey | sudo tee user1_public.key
```

---

### Step 10 — Register the user on the server

Adds a `[Peer]` block to the server config with the user's public key and their assigned VPN IP (`/32` means exactly that one address). Restart WireGuard to apply changes.

```ini
[Peer]
PublicKey  = <USER1_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32
```

```bash
# reload config
sudo wg-quick down wg0 && sudo wg-quick up wg0
```

---

### Step 11 — Create the client config file

Produces a `.conf` file the user imports into the WireGuard app. `AllowedIPs = 0.0.0.0/0` routes all traffic through the VPN. `PersistentKeepalive` keeps the tunnel alive behind NAT routers.

```ini
[Interface]
PrivateKey = <USER1_PRIVATE_KEY>
Address    = 10.0.0.2/24
DNS        = 8.8.8.8

[Peer]
PublicKey           = <SERVER_PUBLIC_KEY>
Endpoint            = <VM_IP>:51820
AllowedIPs          = 0.0.0.0/0
PersistentKeepalive = 25
```

---

### Step 12 — Add a second user

Repeat steps 9–11 for each additional user. Assign the next available VPN IP — user 2 gets `10.0.0.3`, user 3 gets `10.0.0.4`, and so on.

```ini
# Server peer block
AllowedIPs = 10.0.0.3/32

# Client config
Address = 10.0.0.3/24
```

---

### Step 13 — Verify connected peers

Shows all registered peers, their public keys, last handshake time, and bytes transferred. A recent handshake confirms the client successfully connected.

```bash
sudo wg
```