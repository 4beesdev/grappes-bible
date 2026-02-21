# Infrastructure & Deployment

> Last updated: 2026-02-21

## 1. Server

| Property | Value |
|----------|-------|
| Provider | DigitalOcean |
| Region | AMS3 (Amsterdam) |
| Droplet name | `grappes` |
| IP | 164.92.248.164 |
| OS | Ubuntu (Docker pre-installed) |
| SSH | Root access, key-based |

## 2. Domain & SSL

| Property | Value |
|----------|-------|
| Domain | grappes.4bees.io |
| DNS | Managed via domain registrar → A record → 164.92.248.164 |
| SSL | Let's Encrypt (auto via certbot) |
| Certificate path | `/etc/letsencrypt/live/grappes.4bees.io/` |
| Protocols | TLSv1.2, TLSv1.3 |

## 3. Docker Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Docker Compose                        │
│                                                         │
│  ┌─────────┐    ┌──────────┐    ┌──────────┐           │
│  │  nginx   │───►│ frontend │    │ backend  │           │
│  │  :80/443 │    │  :3000   │    │  :8080   │           │
│  └────┬─────┘    └──────────┘    └────┬─────┘           │
│       │                               │                  │
│       └───────────────────────────────┤                  │
│                    /api/ ─────────────┘                  │
│                                                         │
│  ┌────────────┐    ┌─────────┐                          │
│  │ PostgreSQL │    │  Redis  │                          │
│  │   :5432    │    │  :6379  │                          │
│  └────────────┘    └─────────┘                          │
│   (internal only)   (internal only)                     │
└─────────────────────────────────────────────────────────┘
         │
    Only 80/443 exposed to internet
```

## 4. Containers

| Container | Image | Exposed | Healthcheck |
|-----------|-------|---------|-------------|
| `polyai-nginx-1` | nginx:alpine | 80, 443 (public) | `wget https://localhost/health` |
| `polyai-frontend-1` | polyai-frontend | 3000 (internal) | — |
| `polyai-backend-1` | polyai-backend | 8080 (internal) | `wget http://localhost:8080/health` |
| `polyai-postgres-1` | postgres:16-alpine | 5432 (internal) | `pg_isready` |
| `polyai-redis-1` | redis:7-alpine | 6379 (internal) | `redis-cli ping` |

## 5. Nginx Routing

```
https://grappes.4bees.io/api/*  →  backend:8080  (proxy_pass, SSE-friendly: no buffering)
https://grappes.4bees.io/health →  backend:8080  (health endpoint)
https://grappes.4bees.io/*      →  frontend:3000 (Next.js, WebSocket upgrade for HMR)
http://grappes.4bees.io/*       →  301 redirect to https
```

## 6. Compose Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Base config (used on server) |
| `docker-compose.prod.yml` | Production overrides (memory limits, restart policy, nginx) |

**Server uses:** `docker compose` (reads `docker-compose.yml` by default)

## 7. Deployment Process

### Manual deploy (current)

```bash
# From local machine
rsync -avz --exclude 'node_modules' --exclude '.next' --exclude 'target' --exclude '.git' \
  /path/to/Poly.ai/ root@164.92.248.164:/opt/polyai/

# On server
cd /opt/polyai
docker compose build --no-cache backend    # rebuild backend
docker compose build frontend              # rebuild frontend
docker compose up -d backend frontend      # restart services
```

### Verify deployment

```bash
docker compose ps -a                       # all containers running?
docker logs polyai-backend-1 --tail 20     # backend started OK?
curl -s http://localhost:8080/health        # backend healthy?
curl -sk https://localhost/health           # nginx → backend healthy?
```

## 8. Environment Variables

All secrets are in `/opt/polyai/.env` (not committed to git).

| Variable | Description |
|----------|-------------|
| `DB_NAME`, `DB_USER`, `DB_PASSWORD` | PostgreSQL credentials |
| `DB_HOST`, `DB_PORT` | Set to `postgres` / `5432` in compose |
| `REDIS_HOST`, `REDIS_PORT` | Set to `redis` / `6379` in compose |
| `JWT_SECRET` | HMAC key for JWT signing |
| `CLAUDE_API_KEY` | Anthropic API key |
| `OPENAI_API_KEY` | OpenAI API key |
| `GEMINI_API_KEY` | Google AI API key |
| `GROK_API_KEY` | xAI API key |
| `RUNWAY_API_KEY` | Runway ML API key |
| `KLING_API_KEY` | Kling AI API key |
| `STORAGE_ACCESS_KEY`, `STORAGE_SECRET_KEY` | S3/Spaces credentials |
| `STORAGE_ENDPOINT`, `STORAGE_BUCKET`, `STORAGE_REGION` | S3 config |
| `CORS_ALLOWED_ORIGINS` | Allowed origins for CORS |
| `NEXT_PUBLIC_API_URL` | Frontend → backend API URL |
| `NEXT_PUBLIC_GOOGLE_CLIENT_ID` | Google OAuth client ID |
| `NEXT_PUBLIC_PADDLE_CLIENT_TOKEN` | Paddle.net client token |

## 9. Backups (TODO)

- [ ] Automated PostgreSQL backups (pg_dump daily → S3)
- [ ] Retention: keep 7 daily + 4 weekly
- [ ] Test restore procedure

## 10. Monitoring (TODO)

- [ ] Uptime monitoring (e.g. UptimeRobot on https://grappes.4bees.io/health)
- [ ] Docker container restart alerts
- [ ] Disk space monitoring
- [ ] Log aggregation
