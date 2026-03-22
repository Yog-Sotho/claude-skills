---
name: docker-patterns
description: Docker and Docker Compose best practices for local development, container security, networking, volume strategies, and multi-service orchestration. Always activate when the user is writing a Dockerfile, setting up Docker Compose, troubleshooting container issues, designing multi-container architectures, reviewing images for security or size, managing secrets in containers, or migrating to a containerized workflow — even if they don't explicitly ask for "best practices".
---

# Docker Patterns

Docker and Docker Compose best practices for containerized development and production.

## Workflow

When this skill activates:

1. **Identify the user's stack** from context — language (Node, Python, Go, etc.), services (Postgres, Redis, etc.), and target environment (local dev, staging, production).
2. **Adapt the examples** to their stack — the patterns below use Node.js as the default, but the Python section covers the key differences.
3. **Apply security hardening proactively** — non-root user, pinned tags, and no secrets in layers are defaults, not options.
4. **Flag anti-patterns immediately** if spotted in user-provided Dockerfiles or compose files — don't wait to be asked.
5. **Suggest the `.dockerignore`** whenever a Dockerfile is written — it's easy to forget and causes bloated images.

---

## Docker Compose for Local Development

### Standard Web App Stack

```yaml
# docker-compose.yml
services:
  app:
    build:
      context: .
      target: dev                     # Use dev stage of multi-stage Dockerfile
    ports:
      - "3000:3000"
    volumes:
      - .:/app                        # Bind mount for hot reload
      - /app/node_modules             # Anonymous volume -- preserves container deps
    environment:
      - DATABASE_URL=postgres://postgres:postgres@db:5432/app_dev
      - REDIS_URL=redis://redis:6379/0
      - NODE_ENV=development
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy    # Use healthy, not service_started
    command: npm run dev

  db:
    image: postgres:16-alpine
    ports:
      - "127.0.0.1:5432:5432"        # Bind to localhost only -- not 0.0.0.0
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: app_dev
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "127.0.0.1:6379:6379"        # Bind to localhost only
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  mailpit:                            # Local email testing
    image: axllent/mailpit
    ports:
      - "8025:8025"                   # Web UI
      - "1025:1025"                   # SMTP

volumes:
  pgdata:
  redisdata:
```

### Override Files

Keep environment-specific config out of the base file:

```yaml
# docker-compose.override.yml — auto-loaded in development
services:
  app:
    environment:
      - DEBUG=app:*
      - LOG_LEVEL=debug
    ports:
      - "9229:9229"                   # Node.js debugger port
```

```yaml
# docker-compose.prod.yml — explicit opt-in for production
services:
  app:
    build:
      target: production
    restart: always
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
```

```bash
# Development (auto-loads override)
docker compose up

# Production (explicit file merge)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## Multi-Stage Dockerfiles

### Node.js

```dockerfile
# Stage: dependencies (cached layer -- only invalidated when lock file changes)
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Stage: dev (hot reload, full source)
FROM node:22-alpine AS dev
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

# Stage: build
FROM node:22-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm prune --omit=dev

# Stage: production (minimal, hardened)
FROM node:22-alpine AS production
WORKDIR /app

# Non-root user: create group first, then assign user to it
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

USER appuser

COPY --from=build --chown=appuser:appgroup /app/dist ./dist
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=build --chown=appuser:appgroup /app/package.json ./

ENV NODE_ENV=production
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
```

### Python

```dockerfile
# Stage: dependencies
FROM python:3.12-slim AS deps
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

# Stage: dev
FROM python:3.12-slim AS dev
WORKDIR /app
COPY --from=deps /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY . .
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

# Stage: production (hardened)
FROM python:3.12-slim AS production
WORKDIR /app

RUN groupadd -g 1001 appgroup && \
    useradd -u 1001 -g appgroup -s /sbin/nologin -M appuser

COPY --from=deps /usr/local/lib/python3.12 /usr/local/lib/python3.12
COPY --chown=appuser:appgroup . .

USER appuser

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')" || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Networking

### Service Discovery

Services in the same Compose network resolve by service name automatically:

```
# From the "app" container:
postgres://postgres:postgres@db:5432/app_dev   # "db" resolves to the db container
redis://redis:6379/0                            # "redis" resolves to the redis container
```

### Network Isolation

Segment services by trust level — databases should never be reachable from the frontend:

```yaml
services:
  frontend:
    networks:
      - frontend-net

  api:
    networks:
      - frontend-net
      - backend-net         # Bridge between tiers

  db:
    networks:
      - backend-net         # Only reachable from api, invisible to frontend

networks:
  frontend-net:
  backend-net:
```

### Port Binding

Only expose ports that need to be reachable from outside the Docker network:

```yaml
services:
  db:
    ports:
      - "127.0.0.1:5432:5432"   # Dev: localhost only, not exposed to LAN
    # In production: omit ports entirely -- use internal network only
```

---

## Volume Strategies

```yaml
services:
  app:
    volumes:
      - .:/app                   # Bind mount: host source → container (hot reload)
      - /app/node_modules        # Anonymous volume: shields container deps from host
      - /app/.next               # Anonymous volume: shields build cache from host

  db:
    volumes:
      - pgdata:/var/lib/postgresql/data         # Named volume: survives container restarts
      - ./scripts/init.sql:/docker-entrypoint-initdb.d/init.sql  # Seed scripts

volumes:
  pgdata:                        # Named volumes are managed by Docker, not tied to host path
```

**Rule of thumb:**
- **Named volumes** for persistent data (databases, uploads)
- **Bind mounts** for source code in development
- **Anonymous volumes** to protect container-generated paths from being overwritten by bind mounts

---

## Container Security

### Dockerfile Hardening

```dockerfile
# ✅ Pin to a specific digest or full version tag — never :latest
FROM node:22.12-alpine3.20

# ✅ Create a group, then assign the user to it explicitly
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# ✅ Drop to non-root before any COPY or CMD
USER appuser

# ✅ Never set secrets as ENV — they persist in image layers and are visible in docker inspect
# ❌ ENV API_KEY=sk-proj-xxxxx
```

### Compose Security Options

```yaml
services:
  app:
    security_opt:
      - no-new-privileges:true   # Prevents privilege escalation via setuid binaries
    read_only: true              # Root filesystem is read-only
    tmpfs:
      - /tmp                     # Writable scratch space in memory
      - /app/.cache
    cap_drop:
      - ALL                      # Drop all Linux capabilities...
    cap_add:
      - NET_BIND_SERVICE         # ...add back only what's needed (ports < 1024)
```

### Secret Management

```yaml
# ✅ .env file injected at runtime (never committed to git)
services:
  app:
    env_file:
      - .env
    environment:
      - API_KEY                  # Inherits value from host environment

# ✅ Docker secrets (Swarm / production)
secrets:
  db_password:
    file: ./secrets/db_password.txt

services:
  db:
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
```

---

## .dockerignore

Always include — without it, bind mounts and large directories get sent to the build context:

```
node_modules
.git
.env
.env.*
dist
build
coverage
*.log
.next
.cache
__pycache__
*.pyc
.pytest_cache
.venv
docker-compose*.yml
Dockerfile*
README.md
tests/
.github/
```

---

## Debugging

### Common Commands

```bash
# Logs
docker compose logs -f app           # Follow app logs
docker compose logs --tail=50 db     # Last 50 lines from db

# Shell access
docker compose exec app sh           # Shell into app container
docker compose exec db psql -U postgres  # Connect to Postgres

# Inspect
docker compose ps                    # Running services + health status
docker compose top                   # Processes inside each container
docker stats                         # Live CPU/memory per container

# Rebuild
docker compose up --build            # Rebuild changed images
docker compose build --no-cache app  # Force full rebuild (bypasses layer cache)

# Clean up
docker compose down                  # Stop and remove containers (volumes preserved)
docker compose down -v               # ⚠️ Also removes volumes -- data loss
docker system prune -f               # Remove all unused images, containers, networks
```

### Debugging Network Issues

```bash
# Verify DNS resolution from inside a container
docker compose exec app nslookup db

# Test connectivity to another service
docker compose exec app wget -qO- http://api:3000/health

# Inspect the default network
docker network ls
docker network inspect <project>_default
```

### Debugging Volume Issues

```bash
# List volumes and their mount points
docker volume ls
docker volume inspect <project>_pgdata

# Confirm what's mounted in a running container
docker compose exec app mount | grep /app
docker compose exec app ls -la /app/node_modules
```

---

## Anti-Patterns

| ❌ Anti-pattern | ✅ Instead |
|---|---|
| `FROM node:latest` | Pin to `node:22.12-alpine3.20` |
| Running as root | `addgroup` + `adduser`, then `USER appuser` |
| `ENV API_KEY=secret` in Dockerfile | Inject via `env_file` or Docker secrets at runtime |
| Secrets in `docker-compose.yml` | Use `.env` (gitignored) or Docker secrets |
| No `.dockerignore` | Always include — prevents `node_modules` / `.git` in build context |
| Storing data in container filesystem | Named volumes for all persistent data |
| One giant container with all services | One process per container |
| `docker compose` in production without orchestration | Use Kubernetes, ECS, or Docker Swarm |
| `depends_on: service_started` for databases | Use `condition: service_healthy` with a healthcheck |
| Exposing ports as `0.0.0.0:5432:5432` | Bind to `127.0.0.1:5432:5432` in dev; omit in prod |
