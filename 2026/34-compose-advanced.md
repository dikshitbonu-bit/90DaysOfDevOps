# Day 34 – Docker Compose: Real-World Multi-Container Apps

**ScoreVault-app** — A Live Leaderboard API. A production-like 3-service stack:

| Service | Tech | Role |
|---------|------|------|
| `app` | Node.js + Express | API server — handles requests |
| `db` | PostgreSQL 16 | Permanent score history |
| `cache` | Redis 7 | Live real-time rankings |


---

## Final Files

### `app/Dockerfile`

```dockerfile
# ── Stage 1: Install dependencies ─────────────────────────────────────────────
FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files first — layer cache optimization
# Only re-runs npm install if package.json changes, not on every code change
COPY package*.json ./
RUN npm install --omit=dev        # --omit=dev: skip dev tools (jest, nodemon etc) in prod

COPY . .

# ── Stage 2: Lean runtime image ────────────────────────────────────────────────
FROM node:20-alpine

WORKDIR /app

# Create non-root user BEFORE copying files — cache efficient
# If code changes, this layer is already cached and won't re-run
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copy only what the app needs to run — not the entire builder stage
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/index.js    ./

# chown must come AFTER copy — needs the files to exist first
RUN chown -R appuser:appgroup /app

# Switch to non-root for security — if compromised, attacker has no root access
USER appuser

EXPOSE 3000

CMD ["node", "index.js"]
```

---

### `docker-compose.yml`

```yaml
version: '3.9'

# ── Named Volumes ──────────────────────────────────────────────────────────────
# Data persists across docker compose down/up
# docker compose down -v to explicitly delete
volumes:
  db_data:
    driver: local
  redis_data:
    driver: local

# ── Named Networks ─────────────────────────────────────────────────────────────
# Services can only talk to each other if on the same network
# db and cache are NOT exposed to the outside world — only app is
networks:
  backend:
    driver: bridge
    name: scorevault_network

# ── Services ───────────────────────────────────────────────────────────────────
services:

  # ── Web App ──────────────────────────────────────────────────────────────────
  app:
    build: .                          # build from Dockerfile in current directory
    restart: on-failure               # restart if it crashes, not if manually stopped
    ports:
      - "3000:3000"                   # only the app is exposed to the outside world
    depends_on:
      db:
        condition: service_healthy    # wait for DB healthcheck to pass, not just start
      cache:
        condition: service_healthy    # wait for Redis healthcheck to pass
    env_file:
      - .env                          # secrets (passwords) live here, not in compose
    environment:
      - PORT=3000
      - POSTGRES_HOST=db              # service name = hostname inside Docker network
      - REDIS_HOST=cache              # service name = hostname inside Docker network
    networks:
      - backend

  # ── PostgreSQL ───────────────────────────────────────────────────────────────
  db:
    image: postgres:16-alpine
    restart: always                   # always restart — database must never stay down
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB} -h localhost"]
      interval: 10s       # check every 10 seconds
      timeout: 5s         # fail if no response in 5s
      retries: 5          # mark unhealthy after 5 failures
    networks:
      - backend

  # ── Redis ─────────────────────────────────────────────────────────────────────
  cache:
    image: redis:7-alpine
    restart: on-failure
    command: redis-server --appendonly yes --maxmemory 128mb --maxmemory-policy allkeys-lru
    # --appendonly yes      → persist data to disk, survive restarts
    # --maxmemory 128mb     → hard cap, don't eat the whole server
    # --maxmemory-policy    → when full, evict least recently used keys
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend
```

---

### `.env`

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=supersecret123
POSTGRES_DB=scorevault
```

Secrets live here — never commit this file to Git. Add `.env` to your `.gitignore`.

---

## Task 2: depends_on & Healthchecks

### The Problem Without Healthchecks

Without `condition: service_healthy`, Docker just waits for Postgres to *start* — not for it to be *ready to accept connections*. The app crashes on startup because Postgres needs a few seconds to initialize after the container starts.

### How the Healthcheck Works

```yaml
healthcheck:
  test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB} -h localhost"]
  interval: 10s       # check every 10 seconds
  timeout: 5s         # fail if no response in 5s
  retries: 5          # mark unhealthy after 5 consecutive failures
```

`pg_isready` is a built-in Postgres utility — checks if the server is accepting connections. Docker runs this inside the container on a timer.

### The Chain

```
Docker starts db  →  healthcheck runs every 10s
                  →  passes after ~2-3 checks
                  →  db marked as "healthy"
                  →  Docker THEN starts app
                  →  app connects to a fully ready database ✅
```

**Test it:**
```bash
docker compose down
docker compose up
# Watch — app waits for db to be healthy before starting
```

---

## Task 3: Restart Policies

| Policy | When it restarts | Use case |
|--------|-----------------|----------|
| `no` | Never | Dev/testing |
| `always` | Always, even after `docker stop` then daemon restart | Databases, critical infra |
| `on-failure` | Only on non-zero exit code (crash) | App servers, workers |
| `unless-stopped` | Always EXCEPT after manual `docker stop` | General production services |

**Test restart:always on db:**
```bash
docker kill scorevault-db-1
docker ps --filter name=db    # watch it come back automatically
```

**Test restart:on-failure on app:**
```bash
docker kill scorevault-app-1
# Comes back only if it exited with error code
```

---

## Task 4: Rebuild After Code Change

```bash
# Change something in index.js, then:
docker compose up --build

# Rebuild only the app, leave db and cache untouched:
docker compose up --build app -d
```

---

## Task 5: Named Networks & Volumes

### Why Named Volumes

```bash
# Data survives restarts
docker compose down
docker compose up -d
curl http://localhost:3000/leaderboard/tetris   # scores still there ✅

# Explicitly nuke everything including data
docker compose down -v
```

### Why Named Networks

Only services on the same network can talk. `db` and `cache` have no `ports:` mapping — they're invisible to the outside world. Only `app` is reachable from your machine on port 3000. This is security by design.

```bash
# Inspect the network
docker network inspect scorevault_network
```

---

## Task 6: Scaling (Bonus)

```bash
docker compose up --scale app=3
# Error: address already in use :::3000
```

Port mapping is 1-to-1. Three replicas all try to claim port 3000 on your machine — only one can win. The real solution is a load balancer in front:

```
Client → :80 → Nginx
                 ├─▶ app_1:3000
                 ├─▶ app_2:3000
                 └─▶ app_3:3000
```

This is what Kubernetes solves automatically.

---

## Quick Reference Commands

```bash
# Start everything
docker compose up -d

# Start and watch logs
docker compose up

# Rebuild app after code change
docker compose up --build app -d

# View logs
docker compose logs -f app
docker compose logs -f db

# Check health status of all services
docker compose ps

# Inspect network
docker network inspect scorevault_network

# List volumes
docker volume ls

# Tear down (keep data)
docker compose down

# Tear down + delete all data
docker compose down -v
```

---
