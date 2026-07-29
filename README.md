# Real-Time WebSocket Chat — Dockerized Deployment (Docker + Nginx + CI/CD)

## 1. Project Overview

This project is a real-time WebSocket chat application deployed using Docker, Docker Compose, and an Nginx reverse proxy, automated via a GitHub Actions CI/CD pipeline, and hosted on an AWS EC2 instance.

The backend is a FastAPI + Uvicorn WebSocket server. The frontend is static HTML/JS served by Nginx, which also reverse-proxies WebSocket traffic (`/ws`) to the backend container. The original repository was provided with intentionally broken deployment configuration; this document explains the issues found and how each was fixed.

- **Live Public IP:** `http://51.20.193.114`
- **Repository:** (link to your fork)

> **Note on HTTPS:** This deployment serves plain HTTP only (no domain name / SSL certificate is attached to a bare IP). Some browsers auto-upgrade addresses to HTTPS by default. If the page fails to load, open the link in an Incognito/Private window and type `http://51.20.193.114` explicitly, or disable "Always use secure connections" in your browser's security settings.

---

## 2. Architecture Diagram

```
                    ┌─────────────────────┐
                    │   User's Browser     │
                    │ (multiple tabs/users)│
                    └──────────┬───────────┘
                               │ HTTP / WS
                               │ Port 80
                               ▼
        ┌───────────────────────────────────────────┐
        │        AWS EC2 Instance (Ubuntu)           │
        │        Public IP: 51.20.193.114            │
        │                                             │
        │   ┌───────────────────────────────────┐     │
        │   │   Docker Network: chat-net         │     │
        │   │                                     │     │
        │   │  ┌─────────────┐   ┌─────────────┐ │     │
        │   │  │ chat-nginx  │──▶│ chat-backend│ │     │
        │   │  │ (nginx:     │   │ (FastAPI +  │ │     │
        │   │  │  alpine)    │   │  Uvicorn)   │ │     │
        │   │  │ Port 80     │   │ Port 8000   │ │     │
        │   │  │ (published) │   │ (internal   │ │     │
        │   │  │             │   │  only)      │ │     │
        │   │  └─────────────┘   └─────────────┘ │     │
        │   └───────────────────────────────────┘     │
        └───────────────────────────────────────────┘
                               ▲
                               │ SSH deploy
                               │
                    ┌─────────────────────┐
                    │   GitHub Actions     │
                    │   CI/CD Pipeline     │
                    │  (on push to main)   │
                    └─────────────────────┘
```

- Nginx is the **only** publicly exposed service (port 80).
- The backend container only `expose`s port 8000 internally on the Docker network — it is never reachable directly from the internet.
- Nginx serves static frontend files and reverse-proxies `/ws` requests to the backend over the internal Docker network using the service name `backend`.

---

## 3. How Docker Containers Are Set Up

Two services are defined in `docker-compose.yml`:

| Service | Image | Role | Exposure |
|---|---|---|---|
| `backend` | built from local `Dockerfile` (Python 3.11-slim + FastAPI/Uvicorn) | WebSocket + chat logic | Internal only (`expose: 8000`) |
| `nginx` | `nginx:alpine` | Reverse proxy + static file server | Public (`ports: 80:80`) |

Both containers are set to `restart: always`, so they automatically restart if they crash or if the EC2 instance reboots.

## 4. How Docker Networking Works

Both services are attached to a custom bridge network (`chat-net`) declared in `docker-compose.yml`. Docker Compose provides built-in DNS resolution on this network, so containers can reach each other using their **service name** as a hostname (e.g., `http://backend:8000`) instead of an IP address. This is what allows Nginx to forward WebSocket traffic to the backend container reliably, even though container IPs can change on restart.

The backend container is **not** published to the host (`ports`), only `expose`d — meaning it is reachable from other containers on `chat-net`, but not from outside the Docker host. This follows the principle of least exposure: only Nginx needs to be public.

## 5. How the Nginx Reverse Proxy Works

Nginx listens on port 80 and handles two types of requests:

- **`location /`** — serves static frontend files from `/usr/share/nginx/html` (mounted from the `frontend/` folder), with `try_files` falling back to `index.html` for client-side routing.
- **`location /ws`** — proxies WebSocket connections to `http://backend:8000/ws` using the internal Docker network.

## 6. How WebSocket Works Through Nginx

By default, Nginx treats all requests as standard HTTP and does not know how to handle a WebSocket "upgrade" handshake. To support WebSockets, the `/ws` location block includes:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

- `proxy_http_version 1.1` — WebSocket upgrades require HTTP/1.1 (Nginx defaults to 1.0 for proxying).
- `Upgrade` / `Connection: upgrade` headers — tell Nginx to switch the connection from plain HTTP to a persistent WebSocket connection and forward that upgrade to the backend.
- `proxy_read_timeout` / `proxy_send_timeout` are set to 86400s (24h) so idle WebSocket connections aren't dropped prematurely.

This allows multiple browser tabs/users to open persistent WebSocket connections through Nginx to the same backend, enabling real-time multi-user chat.

## 7. How the CI/CD Pipeline Works

A GitHub Actions workflow (`.github/workflows/main.yml`) runs automatically on every push to the `main` branch:

1. GitHub Actions spins up a runner.
2. Using the `appleboy/ssh-action`, it connects to the EC2 server over SSH using credentials stored as encrypted GitHub Secrets (`SERVER_HOST`, `SERVER_USER`, `SERVER_SSH_KEY`).
3. On the server, it runs:
   ```bash
   cd ~/devops-assignment
   git pull origin main
   docker compose down
   docker compose up -d --build
   ```
4. This pulls the latest code and rebuilds/restarts the containers with zero manual intervention.

**Secrets used (never committed to the repo):**
- `SERVER_HOST` — EC2 public/Elastic IP
- `SERVER_USER` — `ubuntu`
- `SERVER_SSH_KEY` — private key of an SSH keypair authorized on the server (`~/.ssh/authorized_keys`)

## 8. Issues Found and How They Were Fixed

| # | File | Issue | Fix |
|---|---|---|---|
| 1 | `Dockerfile` | Uvicorn was bound to `--host 127.0.0.1`, making the backend unreachable from any other container (including Nginx) since `127.0.0.1` inside a container only refers to itself. | Changed to `--host 0.0.0.0` so the backend accepts connections from other containers on the Docker network. |
![image alt](https://github.com/ops86199/devops-assignment/blob/a03216d4de0a6a6808a185e1e19da5f84a071dad/dockerfile%20of%20changes.png)
| 2 | `docker-compose.yml` | The frontend volume mount was commented out, so Nginx had no static files to serve. | Uncommented `./frontend:/usr/share/nginx/html:ro`. |
![image alt](https://github.com/ops86199/devops-assignment/blob/8b32a76783f3796afeae42b25fc419f852b7ab90/compose%20fix.png)

| 3 | `docker-compose.yml` | No explicit custom network defined between services. | Added a custom bridge network (`chat-net`) for clearer, more reliable service discovery. |
| 4 | `nginx.conf` | `proxy_pass` pointed to `http://localhost:8000/ws`, which inside the Nginx container refers to the Nginx container itself, not the backend — causing `502 Bad Gateway` / `Connection refused`. | Changed to `http://backend:8000/ws`, using the Docker Compose service name resolved via internal DNS. |
![image alt](https://github.com/ops86199/devops-assignment/blob/97b8d9867d6d4d4d032273629ef5ff33091f0191/change%20url%20local%20to%20backend.png)

| 5 | `nginx.conf` | The `Upgrade` and `Connection` headers required for WebSocket handshakes were commented out, so the WebSocket upgrade failed. | Uncommented both headers. |
| 6 | Deployment / AWS | EC2 instance had no Elastic IP, so the public IP changed every time the instance stopped/started, breaking the previously shared link. | Allocated and associated an AWS Elastic IP so the public IP is now fixed permanently. |

## 9. Steps to Deploy This Project

### Prerequisites
- An AWS EC2 instance (Ubuntu 22.04, free-tier `t2.micro` or similar) with Docker and Docker Compose installed
- Security Group allowing inbound HTTP (port 80) and SSH (port 22) from `0.0.0.0/0`
- An Elastic IP associated with the instance (recommended, prevents IP changes on restart)

### Steps
```bash
# 1. Clone the repository
git clone <your-fork-url>
cd devops-assignment

# 2. Build and run
docker compose up -d --build

# 3. Verify containers are running
docker compose ps

# 4. Access the app
# Open http://<your-ec2-public-ip> in a browser
```

### CI/CD Setup (one-time)
1. Generate an SSH keypair on the server and authorize the public key:
   ```bash
   ssh-keygen -t ed25519 -f ~/gh-actions-key -N ""
   cat ~/gh-actions-key.pub >> ~/.ssh/authorized_keys
   ```
2. Add `SERVER_HOST`, `SERVER_USER`, and `SERVER_SSH_KEY` (the private key) as GitHub repository secrets under **Settings → Secrets and variables → Actions**.
3. Push to `main` — the `.github/workflows/main.yml` workflow will automatically deploy on every push.

---
## dash bord 
![image alt](https://github.com/ops86199/devops-assignment/blob/20963792b80e844b9b7647595b2b973689a334ba/dash%20bord%20work%20sucessfully%20.png)

## 10. Tech Stack

- **Backend:** Python 3.11, FastAPI, Uvicorn (WebSocket support)
- **Reverse Proxy:** Nginx (Alpine)
- **Containerization:** Docker, Docker Compose
- **CI/CD:** GitHub Actions + SSH deployment
- **Cloud Hosting:** AWS EC2 (Ubuntu 22.04, Elastic IP)
