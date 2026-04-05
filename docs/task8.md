# Docker Portfolio Website Deployment

A step-by-step guide to installing Docker, containerizing a static portfolio website, and exposing it through an Nginx reverse proxy.

---

## Overview

This project installs Docker on Ubuntu, builds a custom Nginx container image to serve a static HTML portfolio, and configures an Nginx reverse proxy so the site is accessible via a URL path rather than a raw port number.

**Traffic flow:**
```
User request
     │
     ▼
Nginx (port 80) → /portfolio/
     │
     ▼
Docker container (port 8080)
     │
     ▼
Nginx inside container (port 80)
     │
     ▼
index.html
```

---

## Summary

| Setting | Value |
|---|---|
| Container image | nginx:latest |
| Container port | 80 |
| Host port | 8080 |
| Reverse proxy path | /portfolio/ |
| Restart policy | unless-stopped |

---

## Part 1 — Docker Installation

### Step 1 — Verify Docker is installed

Checks whether Docker is installed correctly by printing the installed version number.

```bash
docker --version
```

---

### Step 2 — Start the Docker service

Starts the Docker daemon — the background service responsible for building and managing containers.

```bash
sudo systemctl start docker
```

---

### Step 3 — Enable Docker at boot

Ensures the Docker service starts automatically every time the system reboots.

```bash
sudo systemctl enable docker
```

---

### Step 4 — Check service status

Confirms Docker is running properly before proceeding.

```bash
sudo systemctl status docker
```

---

### Step 5 — Allow non-root Docker usage

Adds the current user to the `docker` group so Docker commands can be run without `sudo`. `newgrp` applies the group change immediately without requiring a logout.

```bash
sudo usermod -aG docker <your-username>
newgrp docker
```

---

### Step 6 — Test Docker installation

Pulls a lightweight test image, runs it as a container, and prints a confirmation message. A successful output means Docker is working correctly.

```bash
docker run hello-world
```

---

## Part 2 — Portfolio Website Deployment

### Step 7 — Create the project directory

Creates a dedicated folder to hold the website files and Docker configuration.

```bash
mkdir ~/portfolio && cd ~/portfolio
```

---

### Step 8 — Create the HTML file

Creates a basic static webpage that will be served inside the container.

```bash
nano index.html
```

```html
<!DOCTYPE html>
<html>
  <head>
    <title>My Portfolio</title>
  </head>
  <body>
    <h1>My Portfolio</h1>
    <p>Hello, this is my Docker hosted website.</p>
  </body>
</html>
```

---

### Step 9 — Create the Dockerfile

Defines the container image. It starts from the official Nginx image and copies the HTML file into the directory Nginx serves by default.

```bash
nano Dockerfile
```

```dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
```

---

### Step 10 — Build the Docker image

Builds a container image from the Dockerfile. `-t` assigns the image a name; `.` tells Docker to use the current directory as the build context.

```bash
docker build -t portfolio-site .
```

---

### Step 11 — Run the container

Deploys the container in the background (`-d`), maps VM port `8080` to container port `80`, and names it `portfolio`.

```bash
docker run -d -p 8080:80 --name portfolio portfolio-site
```

---

### Step 12 — Verify the container is running

Lists all active containers to confirm the deployment succeeded.

```bash
docker ps
```

---

### Step 13 — Configure auto-restart

Removes the existing container and re-runs it with the `--restart unless-stopped` policy. This means the container will automatically start on boot and recover from crashes — stopping only if manually stopped.

```bash
docker stop portfolio && docker rm portfolio

docker run -d -p 8080:80 --restart unless-stopped --name portfolio portfolio-site
```

---

## Part 3 — Nginx Reverse Proxy

### Step 14 — Create the Nginx site config

Creates a reverse proxy rule so requests to `/portfolio/` are forwarded to the Docker container on port `8080`. Users access the site via a path instead of specifying a port.

```bash
sudo nano /etc/nginx/sites-available/portfolio
```

```nginx
server {
    listen 80;

    location /portfolio/ {
        proxy_pass http://localhost:8080/;
    }
}
```

---

### Step 15 — Enable the site

Creates a symbolic link from `sites-available` to `sites-enabled`, which is how Nginx activates a site configuration.

```bash
sudo ln -s /etc/nginx/sites-available/portfolio /etc/nginx/sites-enabled/
```

---

### Step 16 — Test the Nginx config

Validates the Nginx configuration syntax before applying it. Always run this before reloading to catch errors early.

```bash
sudo nginx -t
```

---

### Step 17 — Reload Nginx

Applies the new reverse proxy configuration without interrupting the running Nginx service.

```bash
sudo systemctl reload nginx
```

---

## Verification

Test the container directly on port 8080:

```bash
curl localhost:8080
```

Test via the reverse proxy path:

```bash
curl localhost/portfolio
```

Both should return the HTML content of the portfolio page.