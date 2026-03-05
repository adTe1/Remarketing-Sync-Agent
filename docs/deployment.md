# Deployment — Remarketing Spy Agent

## Deployment Strategy

**Principle**: Build and test locally first, deploy to VPS later. The codebase is identical — only environment configuration changes.

---

## 1. Local Development

### Prerequisites

- Python 3.11+
- Node.js 20+
- Git

### Quick Start

```bash
# Clone
git clone https://github.com/YOUR_USER/Remarketing-Sync-Agent.git
cd Remarketing-Sync-Agent

# Backend
python -m venv .venv
source .venv/bin/activate        # Linux/Mac
# .venv\Scripts\activate         # Windows
pip install -r requirements.txt
pip install -r requirements-dev.txt
playwright install chromium

# Copy env template and fill in your keys
cp .env.example .env
# Edit .env with your API keys

# Run backend
uvicorn agent.main:app --reload --port 8000

# Frontend (new terminal)
cd frontend
npm install
npm run dev
# Dashboard at http://localhost:5173
```

### Running Tests

```bash
# All tests
pytest tests/ -v

# Unit only (fast, no network)
pytest tests/unit/ -v

# Integration (needs running backend)
pytest tests/integration/ -v

# Coverage report
pytest tests/ --cov=agent --cov-report=html
open htmlcov/index.html
```

---

## 2. Docker (Local + VPS)

### Dockerfile

```dockerfile
# -- Backend --
FROM python:3.11-slim AS backend

WORKDIR /app

RUN apt-get update && apt-get install -y \
    wget gnupg \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
RUN playwright install chromium --with-deps

COPY agent/ ./agent/

EXPOSE 8000
CMD ["uvicorn", "agent.main:app", "--host", "0.0.0.0", "--port", "8000"]

# -- Frontend --
FROM node:20-alpine AS frontend-build

WORKDIR /app
COPY frontend/package*.json ./
RUN npm ci
COPY frontend/ ./
RUN npm run build

# -- Final (serves both) --
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    nginx wget gnupg \
    && rm -rf /var/lib/apt/lists/*

COPY --from=backend /app /app
COPY --from=backend /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=backend /usr/local/bin/uvicorn /usr/local/bin/uvicorn
COPY --from=backend /root/.cache/ms-playwright /root/.cache/ms-playwright

COPY --from=frontend-build /app/dist /app/static

COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80 443
CMD ["sh", "-c", "nginx && uvicorn agent.main:app --host 0.0.0.0 --port 8000"]
```

### docker-compose.yml

```yaml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./data:/app/data          # SQLite DB persistence
      - ./.env:/app/.env:ro       # Env vars (read-only)
    restart: unless-stopped
    environment:
      - DATABASE_URL=sqlite:///./data/spyagent.db
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/api/v1/health"]
      interval: 60s
      timeout: 10s
      retries: 3
```

### Run with Docker

```bash
# Build and start
docker compose up -d --build

# View logs
docker compose logs -f

# Stop
docker compose down
```

---

## 3. VPS Deployment

### Recommended VPS Specs

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| CPU | 1 vCPU | 2 vCPU |
| RAM | 2 GB | 4 GB (Playwright uses ~500MB) |
| Storage | 10 GB | 20 GB |
| OS | Ubuntu 22.04+ | Ubuntu 24.04 |
| Network | Public IP + domain | Same |

### Step-by-Step VPS Setup

**1. Initial server setup:**
```bash
# SSH into VPS
ssh root@YOUR_VPS_IP

# Update
apt update && apt upgrade -y

# Create non-root user
adduser spyagent
usermod -aG sudo spyagent
su - spyagent
```

**2. Install Docker:**
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# Log out and back in
```

**3. Clone and configure:**
```bash
git clone https://github.com/YOUR_USER/Remarketing-Sync-Agent.git
cd Remarketing-Sync-Agent

# Create env file with production values
cp .env.example .env
nano .env
# Fill in all API keys, set:
#   SESSION_SECRET=<generate random 64 char string>
#   TOKEN_ENCRYPTION_KEY=<generate Fernet key>
#   META_REDIRECT_URI=https://yourdomain.com/api/v1/auth/meta/callback
#   GOOGLE_REDIRECT_URI=https://yourdomain.com/api/v1/auth/google/callback
```

**4. HTTPS with Let's Encrypt:**
```bash
# Install certbot
sudo apt install certbot

# Get certificate (replace yourdomain.com)
sudo certbot certonly --standalone -d yourdomain.com

# Certificates will be at:
# /etc/letsencrypt/live/yourdomain.com/fullchain.pem
# /etc/letsencrypt/live/yourdomain.com/privkey.pem
```

**5. Nginx configuration (production):**

```nginx
# nginx.conf for production
server {
    listen 80;
    server_name yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # Frontend (React build)
    location / {
        root /app/static;
        try_files $uri $uri/ /index.html;
    }

    # Backend API
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # WebSocket for real-time logs
    location /ws/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

**6. Deploy:**
```bash
docker compose up -d --build

# Verify
curl https://yourdomain.com/api/v1/health
# Should return: {"status": "healthy", ...}
```

**7. Auto-renew SSL:**
```bash
# Add to crontab
sudo crontab -e
# Add line:
0 3 * * * certbot renew --quiet && docker compose restart
```

---

## 4. Monitoring in Production

### Health Check Endpoint

```
GET /api/v1/health

Response:
{
  "status": "healthy",    // or "degraded"
  "checks": {
    "api": "ok",
    "database": "ok",
    "meta_token": "ok",          // or "expired" / "not_connected"
    "google_token": "ok",
    "last_scan": "ok",           // or "stale" (> 24h)
    "playwright": "ok",
    "disk": "ok"                 // or "low" (< 100MB)
  },
  "agent_state": "idle",
  "timestamp": "2026-03-03T14:30:00Z"
}
```

### Slack Alerts

The agent sends Slack alerts when:
- Health check transitions from `healthy` to `degraded`
- Scan fails (API error, timeout, blocked)
- Token expires and refresh fails
- Campaign creation fails on Meta
- Disk space below threshold

### Log Files

```
data/
  logs/
    agent.log          # Main agent log (structured JSON)
    scan.log           # Scan-specific logs
    api.log            # REST API access log
```

Log format (structured JSON per line):
```json
{"timestamp": "2026-03-03T14:30:00Z", "module": "meta_ad_library", "level": "info", "domain": "mentorpro.gr", "message": "Found 4 active ads", "details": {"ad_count": 4}}
```

---

## 5. Backup Strategy

### SQLite Database

```bash
# Manual backup
cp data/spyagent.db data/backups/spyagent_$(date +%Y%m%d).db

# Automated daily backup (add to crontab)
0 2 * * * cp /home/spyagent/Remarketing-Sync-Agent/data/spyagent.db \
              /home/spyagent/backups/spyagent_$(date +\%Y\%m\%d).db
```

### Env File

Keep a secure copy of `.env` outside the server (password manager, encrypted backup).

---

## 6. Updating

```bash
cd Remarketing-Sync-Agent

# Pull latest code
git pull origin main

# Rebuild and restart
docker compose up -d --build

# Verify health
curl https://yourdomain.com/api/v1/health
```

Zero-downtime updates can be achieved with Docker rolling updates or blue-green deployment (future enhancement).

---

## Environment Variables Reference

See full list in `api_integrations.md`. Critical production-only vars:

| Variable | Production Value |
|----------|-----------------|
| `SESSION_SECRET` | Random 64+ char string |
| `TOKEN_ENCRYPTION_KEY` | Fernet key (`python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`) |
| `META_REDIRECT_URI` | `https://yourdomain.com/api/v1/auth/meta/callback` |
| `GOOGLE_REDIRECT_URI` | `https://yourdomain.com/api/v1/auth/google/callback` |
| `DATABASE_URL` | `sqlite:///./data/spyagent.db` |
