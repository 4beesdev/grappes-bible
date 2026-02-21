# Security Plan

> Last updated: 2026-02-21

## 1. Network Security

### 1.1 Exposed Ports (Production)

Only the following ports should be reachable from the public internet:

| Port | Service | Protocol | Access |
|------|---------|----------|--------|
| 22   | SSH     | TCP      | Admin only (key-based auth) |
| 80   | HTTP    | TCP      | Public → redirects to HTTPS |
| 443  | HTTPS   | TCP      | Public → nginx → app |

### 1.2 Internal-Only Services (NOT exposed to internet)

| Port | Service    | Access |
|------|------------|--------|
| 5432 | PostgreSQL | Docker internal network only (`postgres:5432`) |
| 6379 | Redis      | Docker internal network only (`redis:6379`) |
| 8080 | Backend    | Docker internal network only (nginx proxies `/api/` → `backend:8080`) |
| 3000 | Frontend   | Docker internal network only (nginx proxies `/` → `frontend:3000`) |

**Implementation:** `docker-compose.yml` does NOT map ports for postgres/redis. Backend and frontend are accessed through nginx reverse proxy only.

### 1.3 DigitalOcean Cloud Firewall (TODO)

Create a Cloud Firewall with these inbound rules:

```
Type     Protocol  Port Range  Sources
─────────────────────────────────────────────
SSH      TCP       22          All (or specific IPs)
HTTP     TCP       80          All
HTTPS    TCP       443         All
```

All other inbound traffic is denied by default.

**Why Cloud Firewall over UFW:** Docker bypasses UFW by modifying iptables directly. DigitalOcean Cloud Firewall operates at the network level before packets reach the droplet, so Docker cannot bypass it.

**How to set up:**
1. DigitalOcean Console → **Networking** → **Firewalls** → **Create Firewall**
2. Name: `grappes-production`
3. Add inbound rules: SSH (22), HTTP (80), HTTPS (443)
4. Apply to droplet: `grappes`
5. Save

Or via CLI:
```bash
doctl auth init  # enter API token
doctl compute firewall create \
  --name grappes-production \
  --droplet-ids $(doctl compute droplet list --format ID --no-header) \
  --inbound-rules "protocol:tcp,ports:22,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:80,address:0.0.0.0/0,address:::/0 protocol:tcp,ports:443,address:0.0.0.0/0,address:::/0" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0,address:::/0 protocol:udp,ports:all,address:0.0.0.0/0,address:::/0 protocol:icmp,address:0.0.0.0/0,address:::/0"
```

## 2. Authentication & Authorization

### 2.1 User Authentication

| Method | Implementation |
|--------|---------------|
| Email/Password | bcrypt hashed (BCryptPasswordEncoder), stored in PostgreSQL |
| Google OAuth | Server-side ID token verification via Google API |
| JWT Access Token | Short-lived, signed with HMAC-SHA256, stored in localStorage |
| JWT Refresh Token | Long-lived, used to obtain new access tokens |

### 2.2 Token Flow

```
Login → JWT access token + refresh token
  ↓
API request with Authorization: Bearer <token>
  ↓
401 → auto-refresh with refresh token
  ↓
Refresh fails → clear tokens, redirect to /login (SPA event, no hard reload)
```

### 2.3 Backend Security

- **Spring Security** with `SecurityWebFilterChain`
- Public endpoints: `/health`, `/api/auth/**`, `/api/plans`
- All other endpoints require valid JWT
- CORS configured per environment (`CORS_ALLOWED_ORIGINS`)
- Rate limiting via Redis (per-user, per-endpoint)

## 3. Data Security

### 3.1 Database

- PostgreSQL with R2DBC (reactive, non-blocking)
- Flyway migrations for schema versioning
- Passwords: bcrypt (never stored in plain text)
- Connection: internal Docker network, no external access

### 3.2 Redis

- Used for: rate limiting, health checks, caching
- No authentication configured (acceptable since network-isolated)
- **Recommendation:** Add `requirepass` in Redis command for defense-in-depth:
  ```yaml
  redis:
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru --requirepass ${REDIS_PASSWORD}
  ```

### 3.3 API Keys (AI Providers)

All API keys stored in `.env` file on server, injected as environment variables:
- `CLAUDE_API_KEY`
- `OPENAI_API_KEY`
- `GEMINI_API_KEY`
- `GROK_API_KEY`
- `RUNWAY_API_KEY`
- `KLING_API_KEY`

**Never committed to git.** `.gitignore` excludes `.env`.

### 3.4 Storage

- S3-compatible (DigitalOcean Spaces, AMS3)
- Used for: data retention archive, file exports
- Access via `STORAGE_ACCESS_KEY` / `STORAGE_SECRET_KEY` in `.env`

## 4. SSL/TLS

- Let's Encrypt certificate for `grappes.4bees.io`
- Nginx terminates TLS (TLSv1.2 + TLSv1.3)
- HTTP → HTTPS redirect enforced
- Certificate path: `/etc/letsencrypt/live/grappes.4bees.io/`
- **TODO:** Set up auto-renewal cron: `certbot renew --quiet`

## 5. Incident History

| Date | Issue | Impact | Resolution |
|------|-------|--------|------------|
| 2026-02-20 | Redis port 6379 exposed to internet (no auth) | Low — no data in Redis, no external connections detected | Removed port mapping from docker-compose.yml |
| 2026-02-20 | PostgreSQL port 5432 exposed to internet | Low — password-protected, no external connections detected | Removed port mapping from docker-compose.yml |
| 2026-02-20 | Backend Docker healthcheck hitting non-existent `/actuator/health` | None (cosmetic — container marked unhealthy) | Changed to `/health` in Dockerfile |

## 6. Security Checklist

- [x] Redis/Postgres not exposed to internet
- [ ] DigitalOcean Cloud Firewall configured
- [ ] Redis password set (`requirepass`)
- [ ] SSH key-only auth (disable password login)
- [ ] Let's Encrypt auto-renewal cron
- [ ] Rate limiting tested under load
- [ ] Dependency vulnerability scan (Snyk / Dependabot)
- [ ] API key rotation schedule defined
- [ ] Backup strategy for PostgreSQL
