# Task 5 — Reverse Proxy Setup with Nginx

**Induction 2026 | Web Server Deployment and Secure Configuration**

---

## Overview

This task involves setting up two backend applications and securely exposing them via an Nginx reverse proxy. Direct access to backend ports is blocked using UFW (Uncomplicated Firewall), ensuring all traffic flows exclusively through Nginx on port 80.

### Architecture

```
Internet → Nginx (port 80)
                ↓
        /server1  →  App1 (port 8008)
        /sslopen  →  App2 (port 3000)
```

Backend ports `8008` and `3000` are firewalled. Only Nginx (running on the same host) can reach them via `localhost`.

---

## Step 1 — Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

- `apt update` — Refreshes the local package index to fetch the latest package metadata from configured repositories.
- `apt install nginx -y` — Installs the Nginx web server. The `-y` flag auto-confirms the installation prompt.
- `systemctl enable nginx` — Configures Nginx to start automatically on every system boot.
- `systemctl start nginx` — Starts the Nginx service immediately without requiring a reboot.

**Verify Nginx is running:**

```bash
sudo systemctl status nginx
```

This shows the current status of the Nginx service, including whether it's active (running), any recent log lines, and the PID. You should see `Active: active (running)` in green.

---

## Step 2 — Create a Non-Privileged App User

```bash
sudo adduser appuser
sudo mkdir -p /opt/apps
sudo chown appuser:appuser /opt/apps
sudo su - appuser
cd /opt/apps
```

- `adduser appuser` — Creates a new system user called `appuser`. Running applications under a dedicated non-privileged user limits the damage if an app is compromised — it won't have root or sudo access.
- `mkdir -p /opt/apps` — Creates the `/opt/apps` directory (and any missing parent directories) as the working directory for both applications.
- `chown appuser:appuser /opt/apps` — Transfers ownership of `/opt/apps` to `appuser`, so the user can read/write files there without needing elevated privileges.
- `su - appuser` — Switches the current shell session to `appuser`. The `-` flag loads the user's full environment (home directory, shell profile, etc.) as if they had logged in directly.
- `cd /opt/apps` — Navigates into the application directory.

> **Security Note:** Applications must run from a non-privileged user. This is a core principle of least privilege — services should only have the permissions they absolutely need.

---

## Step 3 — Setup App1

### Download App1 and Its Signature

```bash
wget https://do.edvinbasil.com/ssl/app
wget https://do.edvinbasil.com/ssl/app.sha256.sig
```

- `wget <url>` — Downloads a file from the given URL to the current directory. The first command downloads the `app` binary; the second downloads its SHA-256 checksum signature file, which is used to verify the binary's integrity.

### Make App1 Executable

```bash
chmod +x app
```

- `chmod +x app` — Adds the executable (`x`) permission to the `app` file. Binaries downloaded from the internet are not executable by default — this step is required before you can run the file.

### Run App1 (Foreground Test)

```bash
./app
```

- `./app` — Runs the `app` binary from the current directory. The `./` prefix explicitly tells the shell to look in the current directory rather than searching `$PATH`. App1 starts an HTTP server on port `8008` and echoes back the path from each incoming request.

### Test App1

```bash
curl localhost:8008/test
```

- `curl localhost:8008/test` — Sends an HTTP GET request to App1 at `localhost:8008/test`. If the app is running correctly, it should respond with something like `SSL Onboarding. path: /test`. This confirms App1 is alive and responding before you configure Nginx.

### Run App1 in the Background

```bash
nohup ./app > app.log 2>&1 &
```

- `nohup` — "No Hang Up" — tells the process to keep running even after the terminal session is closed or the user logs out. Without `nohup`, the app would be killed when you exit.
- `./app` — The binary to run.
- `> app.log` — Redirects standard output (stdout) to `app.log`, so you can review logs later.
- `2>&1` — Redirects standard error (stderr) to the same location as stdout, so both normal output and error messages go into `app.log`.
- `&` — Runs the command in the background, returning control of the terminal immediately.

---

## Step 4 — Secure App1 (Block Direct External Access)

### Check That App1 Is Listening

```bash
ss -tulnp | grep 8008
```

- `ss` — Socket Statistics tool; shows current network connections and listening ports (modern replacement for `netstat`).
- `-t` — Show TCP sockets only.
- `-u` — Show UDP sockets only.
- `-l` — Show only listening sockets (servers waiting for connections).
- `-n` — Show numeric addresses and port numbers instead of resolving hostnames/service names.
- `-p` — Show the process (name and PID) associated with each socket.
- `| grep 8008` — Pipes the output into `grep` to filter and show only lines containing `8008`, confirming App1 is actively listening on that port.

### Block Port 8008 from External Access

```bash
sudo ufw deny 8008/tcp
```

- `ufw deny 8008/tcp` — Adds a UFW firewall rule that drops all incoming TCP traffic to port `8008`. This prevents the internet from reaching App1 directly — all requests must go through Nginx instead.

### Verify the Firewall Rule

```bash
sudo ufw status
```

- `ufw status` — Displays the current UFW firewall rules and whether UFW is active. You should see `8008/tcp` listed as `DENY`.

---

## Step 5 — Configure Nginx as Reverse Proxy for App1

### Edit the Nginx Default Site Configuration

```bash
sudo nano /etc/nginx/sites-available/default
```

- `nano /etc/nginx/sites-available/default` — Opens the default Nginx site configuration file in the `nano` text editor. This is the main config file for the default virtual host.

### Add the Reverse Proxy Location Block

Inside the `server { }` block, add:

```nginx
location /server1/ {
    proxy_pass http://localhost:8008/;
}
```

- `location /server1/` — Defines a routing rule: any incoming HTTP request whose path starts with `/server1/` will be handled by this block.
- `proxy_pass http://localhost:8008/;` — Forwards the matched request to App1 running locally on port `8008`. The trailing `/` in both `location` and `proxy_pass` is important — it tells Nginx to strip the `/server1/` prefix before forwarding, so App1 receives `/` or `/test` instead of `/server1/test`.

### Test the Nginx Configuration

```bash
sudo nginx -t
```

- `nginx -t` — Tests the Nginx configuration files for syntax errors without actually reloading the server. Always run this before reloading — if there's a typo, Nginx will show you exactly where it is. A successful test prints: `syntax is ok` and `test is successful`.

### Reload Nginx

```bash
sudo systemctl reload nginx
```

- `systemctl reload nginx` — Gracefully reloads Nginx with the updated configuration. Unlike `restart`, `reload` does not drop existing connections — it signals Nginx to re-read its config and apply changes without downtime.

### Test the Reverse Proxy for App1

```bash
curl localhost/server1/test
```

- This sends a request to Nginx at `localhost/server1/test`. Nginx matches the `/server1/` prefix, strips it, and forwards `/test` to App1 on port `8008`. A successful response confirms the reverse proxy is working correctly.

---

## Step 6 — Setup App2 (IsSSLOpen)

### Clone the Repository

```bash
cd /opt/apps
git clone https://gitlab.com/tellmeY/issslopen.git
cd issslopen
```

- `git clone <url>` — Downloads the entire repository from GitLab, including all source files and history, into a new directory called `issslopen`.
- `cd issslopen` — Navigates into the cloned repository directory.

### Create the Environment File

```bash
cp .env.example .env
```

- `cp .env.example .env` — Copies the example environment configuration file to `.env`. The `.env` file holds environment-specific settings (like secrets and config values) that the application reads at runtime. You can edit `.env` to customize the configuration (e.g., set an edit token for IsSSLOpen).

### Install Docker and Docker Compose

```bash
sudo apt install docker.io docker-compose -y
```

- `docker.io` — Installs the Docker container runtime, which allows you to build and run containers.
- `docker-compose` — Installs Docker Compose, a tool for defining and running multi-container Docker applications using a `docker-compose.yml` file.

### Enable and Start Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

- `systemctl enable docker` — Configures Docker to start automatically on boot.
- `systemctl start docker` — Starts Docker immediately.

### Add `appuser` to the Docker Group

```bash
sudo usermod -aG docker appuser
sudo su - appuser
```

- `usermod -aG docker appuser` — Adds `appuser` to the `docker` group. By default, only root can run Docker commands. Adding the user to the `docker` group lets them run Docker without `sudo`.
- `-aG` — The `-a` (append) flag ensures the user is added to the group without being removed from existing groups; `-G` specifies the group name.
- `su - appuser` — Re-logs in as `appuser` so that the new group membership takes effect in the current session (group changes don't apply until the next login).

### Start App2 with Docker Compose

```bash
docker-compose up -d
```

- `docker-compose up` — Reads `docker-compose.yml` and starts all defined services (containers).
- `-d` — "Detached" mode — runs all containers in the background, returning the terminal prompt immediately.

### Verify App2 Is Running

```bash
ss -tulnp | grep 3000
```

Checks whether App2's container has bound to port `3000` on the host (see Step 4 for a full explanation of `ss` flags).

### Test App2 Directly

```bash
curl localhost:3000
```

Sends an HTTP request directly to App2 on port `3000` to confirm it's responding before Nginx is configured for it.

---

## Step 7 — Secure App2 (Block Direct External Access)

```bash
sudo ufw deny 3000/tcp
sudo ufw status
```

- `ufw deny 3000/tcp` — Blocks all external incoming connections to port `3000`, just as was done for App1 on port `8008`. App2 is now only reachable through Nginx.
- `ufw status` — Confirms the rule is in place.

---

## Step 8 — Configure Nginx as Reverse Proxy for App2

### Edit the Nginx Configuration

```bash
sudo nano /etc/nginx/sites-available/default
```

### Add the Location Block for App2

```nginx
location /sslopen/ {
    proxy_pass http://localhost:3000/;
}
```

- `location /sslopen/` — Routes all requests starting with `/sslopen/` to App2.
- `proxy_pass http://localhost:3000/;` — Forwards those requests to App2 running on port `3000`. The trailing slashes strip the `/sslopen/` prefix before forwarding.

### Test and Reload Nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Same as Step 5 — validate config syntax, then apply changes gracefully.

### Test the Reverse Proxy for App2

```bash
curl localhost/sslopen/
```

Sends a request through Nginx to verify it correctly proxies to App2 on port `3000`.

---

## Final Verification

### Test Both Applications Through Nginx

```bash
curl localhost/server1/test
curl localhost/sslopen/
```

Both should return valid responses, routed entirely through Nginx — confirming the reverse proxy is fully functional.

### Verify Firewall Status

```bash
sudo ufw status
```

**Expected open ports:**

| Port | Protocol | Status | Purpose |
|------|----------|--------|---------|
| 80 | TCP | ALLOW | HTTP (Nginx) |
| 443 | TCP | ALLOW | HTTPS (Nginx) |
| 2222 | TCP | ALLOW | SSH (non-default port) |

**Expected blocked ports:**

| Port | Protocol | Status | Reason |
|------|----------|--------|--------|
| 8008 | TCP | DENY | App1 — internal only |
| 3000 | TCP | DENY | App2 — internal only |

---

## Summary

| Component | Technology | Port | Access |
|-----------|------------|------|--------|
| Reverse Proxy | Nginx | 80, 443 | Public |
| App1 | Native binary | 8008 | Localhost only |
| App2 (IsSSLOpen) | Docker / Node.js | 3000 | Localhost only |

By firewalling backend ports and routing all public traffic through Nginx, this setup ensures:

- **Security** — Backend services are not exposed to the internet directly.
- **Flexibility** — Nginx can handle SSL termination, load balancing, and CSP headers centrally.
- **Least Privilege** — Apps run under a non-privileged `appuser` account, limiting the blast radius of any potential compromise.