# Real-Time WebSocket Chat — Docker + Nginx + CI/CD

A containerized real-time multi-user chat application built with FastAPI WebSockets, served through an Nginx reverse proxy, deployed on Oracle Cloud, and automated via GitHub Actions CI/CD.

---

## Live Demo

**URL:** [https://demo-chat.duckdns.org](https://demo-chat.duckdns.org)

---

## Repository

[https://github.com/spy1345a/devops](https://github.com/spy1345a/devops)

---

## Architecture

```
                               User Browser
                                    │
                                    ▼
                   https://demo-chat.duckdns.org (port 443)
                                    │
                                    ▼
                    ┌──────────────────────────────────────┐
                    │           Nginx Container            │
                    │           (chat-nginx)               │
                    │                                      │
                    │  :80   →  redirect to HTTPS          │
                    │  :443  →  serve frontend HTML        │
                    │  /ws   →  proxy to backend:8000      │
                    └───────────────┬──────────────────────┘
                                    │   Docker bridge network
                                    ▼
                    ┌──────────────────────────────────────┐
                    │         FastAPI Container            │
                    │         (chat-backend)               │
                    │         0.0.0.0:8000                 │
                    │                                      │
                    │  WebSocket /ws endpoint              │
                    │  ConnectionManager (broadcast)       │
                    └──────────────────────────────────────┘
```

Nginx is the only container exposed to the internet (ports 80 and 443). The backend is internal-only — reachable only within the Docker network via the service name `backend`.

---

## Project Structure

```
devops/
├── app/
│   ├── main.py              # FastAPI app with WebSocket endpoint
│   └── requirements.txt     # Python dependencies
├── frontend/
│   └── index.html           # Chat UI served by Nginx
├── .github/
│   └── workflows/
│       └── deploy.yml       # CI/CD pipeline
├── Dockerfile               # Backend container image
├── docker-compose.yml       # Service orchestration
├── nginx.conf               # Reverse proxy configuration
└── README.md
```

---

## How Docker Containers Are Set Up

### Backend (`Dockerfile`)
Built from `python:3.11-slim`. Installs dependencies from `app/requirements.txt`, copies `app/main.py`, and runs Uvicorn on `0.0.0.0:8000` so the server accepts connections from anywhere on the Docker network — not just localhost.

### Nginx
Uses the official `nginx:alpine` image. Volumes mounted read-only:
- `./frontend` → `/usr/share/nginx/html` — serves the chat UI static files
- `./nginx.conf` → `/etc/nginx/nginx.conf` — applies the reverse proxy configuration
- `/etc/letsencrypt` → `/etc/letsencrypt` — TLS certificates from Certbot

Ports 80 and 443 are published to the host. Port 80 redirects to HTTPS.

Both containers have `restart: always` so they automatically recover on crash or server reboot.

---

## How Docker Networking Works

Docker Compose automatically creates a shared bridge network for all services defined in `docker-compose.yml`. Each service is registered by its **service name** as a DNS hostname on that network.

```
Service name → Internal hostname
backend      → backend (port 8000, internal only)
nginx        → nginx   (ports 80/443, exposed to internet)
```

The backend uses `expose: "8000"` instead of `ports` — this makes port 8000 reachable only within the Docker network, keeping it off the public internet.

Nginx reaches the backend at `http://backend:8000`. No manual network blocks or IP addresses needed.

---

## How Nginx Reverse Proxy Works

Nginx serves two roles simultaneously:

**Static file server** — handles all `GET /` requests by serving `index.html` from the mounted frontend volume over HTTPS.

**WebSocket reverse proxy** — forwards all `/ws` connections through to the FastAPI backend.

```nginx
# Redirect HTTP → HTTPS
server {
    listen 80;
    server_name demo-chat.duckdns.org;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name demo-chat.duckdns.org;

    ssl_certificate /etc/letsencrypt/live/demo-chat.duckdns.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/demo-chat.duckdns.org/privkey.pem;

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /ws {
        proxy_pass http://backend:8000/ws;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

`proxy_read_timeout` and `proxy_send_timeout` are set to 86400s (24 hours) to keep long-lived WebSocket connections open without being dropped.

---

## How WebSocket Works Through Nginx

A WebSocket connection starts as a standard HTTP request then asks the server to upgrade the protocol to a persistent bidirectional TCP tunnel. Nginx must explicitly pass the upgrade headers — without them, it treats the request as plain HTTP and the handshake fails.

```
Browser → HTTPS GET /ws (Upgrade: websocket)
             ↓
          Nginx receives, passes Upgrade headers to backend
             ↓
          Backend accepts, upgrades to WebSocket
             ↓
          Persistent bidirectional connection established
             ↓
          Messages broadcast to all connected clients
```

The FastAPI `ConnectionManager` maintains a list of all active WebSocket connections and broadcasts every chat message to every connected client in real time.

---

## CI/CD Pipeline

The GitHub Actions workflow triggers on every push to `main`.

```
                                    Developer
                                       │
                                       │  git push origin main
                                       ▼
                           ┌─────────────────────────┐
                           │     GitHub              │
                           │     detects push        │
                           └───────────┬─────────────┘
                                       │  triggers workflow
                                       ▼
                           ┌─────────────────────────┐
                           │   GitHub Actions        │
                           │   ubuntu-latest runner  │
                           │                         │
                           │   actions/checkout@v4   │
                           └───────────┬─────────────┘
                                       │  appleboy/ssh-action
                                       │  SSH into VPS
                                       ▼
                           ┌─────────────────────────┐
                           │   Oracle Cloud VPS      │
                           │   (Rocky Linux)         │
                           │                         │
                           │   cd ~/demo/devops      │
                           │         │               │
                           │         ▼               │
                           │   git pull origin main  │
                           │         │               │
                           │         ▼               │
                           │  docker compose down    │
                           │         │               │
                           │         ▼               │
                           │  docker compose up      │
                           │      -d --build         │
                           │         │               │
                           │         ▼               │
                           │  ✓ Containers running   │
                           └─────────────────────────┘
                                       │
                                       ▼
                           https://demo-chat.duckdns.org
                                 live and updated
```

**GitHub repository secrets required:**

| Secret | Value |
|---|---|
| `VPS_HOST` | Oracle Cloud VM public IP |
| `VPS_USER` | SSH username (`rocky`) |
| `VPS_SSH_KEY` | Private key for SSH access |
| `VPS_SSH_PORT` | SSH port (default `22`) |

---

## Bugs Found and Fixed

### Bug 1 — Uvicorn bound to `127.0.0.1` (Dockerfile)

**File:** `Dockerfile`

**Problem:**
```dockerfile
CMD ["uvicorn", "main:app", "--host", "127.0.0.1", "--port", "8000"]
```
`127.0.0.1` is the loopback interface — it only accepts connections from within the same container. The Nginx container on the Docker network could not reach it at all, even though both were on the same compose network.

**Fix:**
```dockerfile
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
`0.0.0.0` tells Uvicorn to listen on all network interfaces, making it reachable from other containers on the Docker bridge network.

---

### Bug 2 — Frontend volume not mounted (docker-compose.yml)

**File:** `docker-compose.yml`

**Problem:** The `frontend/` directory was not mapped into the Nginx container. Nginx had no HTML files to serve and showed the default "Welcome to nginx" page instead of the chat UI.

**Fix:**
```yaml
volumes:
  - ./frontend:/usr/share/nginx/html:ro
  - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

---

### Bug 3 — Wrong backend hostname + WebSocket headers disabled (nginx.conf)

**File:** `nginx.conf`

**Problem 1:** `proxy_pass http://localhost:8000/ws` — `localhost` inside the Nginx container resolves to the Nginx container itself, not the FastAPI backend. The connection was going nowhere.

**Fix:** Changed to `proxy_pass http://backend:8000/ws` — Docker's internal DNS resolves the Compose service name `backend` to the correct container.

**Problem 2:** The two headers required for WebSocket protocol upgrade were commented out:
```nginx
# proxy_set_header Upgrade $http_upgrade;
# proxy_set_header Connection "upgrade";
```
Without these, Nginx does not perform the HTTP → WebSocket upgrade and the handshake always fails.

**Fix:** Uncommented both headers.

---

### Bug 4 — Port 443 not exposed (docker-compose.yml)

**File:** `docker-compose.yml`

**Problem:** After adding HTTPS, only port 80 was published to the host. The Nginx container was listening on 443 internally but the port was never opened to the outside world, causing `ERR_ADDRESS_UNREACHABLE`.

**Fix:**
```yaml
ports:
  - "80:80"
  - "443:443"
```

---

## Deployment Steps

### Prerequisites
- Oracle Cloud VM running Rocky Linux with Docker installed
- Ports 80 and 443 open in OCI Security List AND Rocky Linux firewall (`firewall-cmd`)
- DuckDNS domain pointed at the VM's public IP
- TLS certificate obtained via Certbot

### Get TLS certificate (one time)

```bash
sudo dnf install certbot -y
docker compose down
sudo certbot certonly --standalone -d demo-chat.duckdns.org
docker compose up -d
```

### Run manually

```bash
git clone https://github.com/spy1345a/devops.git
cd devops
docker compose up -d --build
```

Access at `https://demo-chat.duckdns.org`

### Automated via CI/CD

1. Add `VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`, and `VPS_SSH_PORT` to GitHub repository secrets
2. Push any change to the `main` branch
3. GitHub Actions SSHs into the server, pulls the latest code, and restarts containers automatically

---

## Tech Stack

| Component | Technology |
|---|---|
| Backend | FastAPI + Uvicorn (Python 3.11) |
| WebSocket | FastAPI WebSocket |
| Frontend | HTML / JavaScript |
| Reverse Proxy | Nginx Alpine |
| TLS | Let's Encrypt via Certbot |
| Containerization | Docker + Docker Compose v2 |
| CI/CD | GitHub Actions + appleboy/ssh-action |
| Cloud | Oracle Cloud ARM (free tier) |
| OS | Rocky Linux |
| DNS | DuckDNS |


---

## Bonus: Load Balancer Architecture

In a production setup with high traffic, a single Nginx container becomes a bottleneck.
The solution is to run multiple backend containers behind a load balancer.

 ```
                      Users
                        │
                        ▼
             demo-chat.duckdns.org
                        │
                        ▼
           ┌────────────────────────┐
           │     Nginx (LB)         │
           │   upstream backends    │
           └──────────┬─────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ backend1 │ │ backend2 │ │ backend3 │
    │ :8001    │ │ :8002    │ │ :8003    │
    └──────────┘ └──────────┘ └──────────┘
          │            │            │
          └────────────┼────────────┘
                       ▼
              ┌─────────────────┐
              │   Redis         │
              │ (shared state)  │
              │ active sessions │
              └─────────────────┘
```

Nginx would be configured with an `upstream` block:

```nginx
upstream backend_pool {
    least_conn;
    server backend1:8000;
    server backend2:8000;
    server backend3:8000;
}

location /ws {
    proxy_pass http://backend_pool;
    ...
}
```

`least_conn` routes each new connection to whichever backend currently has
the fewest active connections — important for WebSockets since connections
are long-lived, unlike regular HTTP requests.

**Why Redis is needed here:** each backend container has its own
`ConnectionManager` in memory. Without shared state, a message sent to
`backend1` would never reach a user connected to `backend2`. Redis acts as
a shared pub/sub bus — every backend publishes messages to Redis and every
backend subscribes, so all connected users receive all messages regardless
of which container they're on.

---

## Bonus: Auto-Scaling Approach

Auto-scaling adds or removes backend containers automatically based on load.

### With Docker Swarm (simplest)

Docker Swarm is built into Docker and requires no extra tooling:

```bash
docker swarm init
docker stack deploy -c docker-compose.yml chat
```

Then scale manually or automatically:

```bash
# manual scale
docker service scale chat_backend=5

# auto-scale based on CPU (via custom script or swarm-cronjob)
```

### With a Cloud Auto-Scaler (production approach)
```
                CloudWatch / OCI Monitoring
                   (CPU > 70% for 2 min)
                           │
                           ▼
                 Auto-Scaling Group triggers
                           │
                   ┌───────┴────────┐
                   ▼                ▼
              Spin up new      Terminate idle
              VM + container   containers when
              from snapshot    CPU < 30%
```
On Oracle Cloud this would use **Instance Pools** with a scaling policy:

- Scale out: add a VM when average CPU across the pool exceeds 70%
- Scale in: remove a VM when average CPU drops below 30% for 10 minutes
- Minimum instances: 1 (always one backend running)
- Maximum instances: 5 (cost cap)

New VMs pull the latest Docker image on startup via a cloud-init script,
register themselves with the load balancer automatically, and are ready
to serve traffic within ~60 seconds of the scale-out trigger.