# Task 2 — Enhanced SSH Security

## How SSH Login Works

When connecting to the VM:

```bash
ssh -i mykey.pem azureuser@VM_IP
```

The `.pem` file is your **private key**. The VM stores the matching **public key** in `~/.ssh/authorized_keys`. SSH verifies the pair mathematically — no password needed.

## Why Default SSH Is Unsafe

| Problem | Risk | Fix |
|---------|------|-----|
| Root login enabled | Full system takeover if compromised | Disable root login |
| Password auth enabled | Vulnerable to brute-force attacks | Use keys only |
| Default port 22 | Scanned constantly by internet bots | Change to custom port |
| No brute-force protection | Unlimited login attempts allowed | Install Fail2ban |

> Bots scan millions of IPs daily trying common username/password combinations — this is called a **brute-force attack** and is normal internet background traffic. A fresh VM gets hit within minutes of creation.

## 2.1 SSH Configuration

Edited `/etc/ssh/sshd_config`:

```
PermitRootLogin no           # Disables root login
PasswordAuthentication no    # Forces key-based auth only
Port 2222                    # Moves SSH off the default scanned port
```

After changing the port, an **Azure NSG inbound rule** was added to allow TCP 2222 — without this, Azure's firewall blocks the connection before it reaches the VM.

Verified SSH is listening on the new port:

```bash
sudo ss -tulnp | grep ssh
```

## 2.2 Fail2ban Setup

Fail2ban monitors auth logs and bans IPs after repeated failed login attempts.

Created `/etc/fail2ban/jail.local`:

```ini
[sshd]
enabled  = true
port     = 2222
logpath  = /var/log/auth.log
maxretry = 5       # Failed attempts before ban
findtime = 600     # 10-minute window for counting failures
bantime  = 3600    # Ban duration: 1 hour
```

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd    # Verify it's actively monitoring
```

## Security Flow After Hardening

```
Internet
   ↓
Password disabled  →  rejected
Keys required      →  verified
Failed attempts    →  banned by Fail2ban
   ↓
Only authorized access granted
```
