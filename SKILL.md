---
name: production-deployment
description: Deployment workflows, CI/CD pipelines, Docker containerization, health checks, rollback strategies, database migrations, and production readiness for web applications. Always activate when the user is setting up CI/CD, Dockerizing an application, planning a deployment strategy, implementing health checks, preparing a production release, configuring environments, or asking how to deploy, roll back, or migrate a database safely. Also activate proactively when spotting hardcoded secrets, missing health checks, or root-running containers.
---

# Production Deployment

Production deployment workflows, CI/CD patterns, and operational readiness.

## Workflow

When this skill activates:

1. **Identify the task** — new pipeline, Dockerfile, deployment strategy, health checks, database migration, or rollback plan.
2. **Navigate to the relevant section** — don't apply every pattern to every situation.
3. **Apply the production readiness checklist** before any release goes out.
4. **For structured logging, metrics, smoke tests, and observability implementation**, see `references/observability.md`.
5. **Flag violations proactively** — root-running containers, `:latest` tags, missing health check timeouts, and hardcoded secrets are the most common.

---

## Deployment Strategies

### Rolling (Default)

Replace instances gradually — old and new run simultaneously during rollout.

```
Instance 1: v1 → v2   ← updated first, traffic continues
Instance 2: v1         ← still v1
Instance 3: v1         ← still v1

Instance 1: v2
Instance 2: v1 → v2   ← updated second
Instance 3: v1

Instance 1: v2
Instance 2: v2
Instance 3: v1 → v2   ← updated last
```

**Pros:** Zero downtime, gradual rollout, no extra infrastructure
**Cons:** Two versions run simultaneously — API must be backward-compatible
**Use when:** Standard deployments, backward-compatible changes

### Blue-Green

Two identical environments. Switch traffic atomically.

```
Blue  (v1) ← all traffic
Green (v2)   idle, new version deployed and verified

# After verification — atomic cutover:
Blue  (v1)   standby (instant rollback target)
Green (v2) ← all traffic
```

**Pros:** Instant rollback (flip back to blue), clean cutover, no mixed versions
**Cons:** Requires 2× infrastructure during deployment
**Use when:** Critical services, zero tolerance for partial failures

### Canary

Route a small percentage of traffic to the new version first.

```
v1: 95% of traffic
v2:  5% of traffic  ← canary watches error rate, latency, business metrics

# If metrics are healthy after N minutes:
v1: 0%   →   v2: 100%

# If metrics degrade:
v2: 0%   →   automatic rollback to v1
```

**Pros:** Real traffic validation before full rollout, automatic abort criteria
**Cons:** Requires traffic-splitting infrastructure and automated monitoring
**Use when:** High-traffic services, risky changes, when you need real traffic signal

---

## Docker

### Multi-Stage Dockerfile (Node.js)

```dockerfile
# Stage 1: Install all dependencies (including dev)
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Stage 2: Build
FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm prune --omit=dev   # --omit=dev replaces deprecated --production

# Stage 3: Production image — minimal, hardened
FROM node:22-alpine AS runner
WORKDIR /app

# Create group first, then assign user to it — both needed for --chown to work
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

USER appuser

COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

ENV NODE_ENV=production
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --start-period=15s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/server.js"]
```

### Multi-Stage Dockerfile (Go)

```dockerfile
FROM golang:1.22-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
# Static binary — no libc dependency
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o /server ./cmd/server

FROM alpine:3.20 AS runner
RUN apk --no-cache add ca-certificates tzdata
RUN addgroup -g 1001 -S appgroup && adduser -S appuser -u 1001 -G appgroup
USER appuser

# --chown ensures appuser can execute the binary
COPY --from=builder --chown=appuser:appgroup /server /server

EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD wget -qO- http://localhost:8080/health || exit 1
CMD ["/server"]
```

### Multi-Stage Dockerfile (Python)

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
RUN pip install --no-cache-dir uv
COPY requirements.txt .
RUN uv pip install --system --no-cache -r requirements.txt

FROM python:3.12-slim AS runner
WORKDIR /app

RUN groupadd -g 1001 appgroup && \
    useradd -r -u 1001 -g appgroup appuser

# Copy installed packages and app source — set ownership before USER switch
COPY --from=builder --chown=appuser:appgroup /usr/local/lib/python3.12/site-packages \
     /usr/local/lib/python3.12/site-packages
COPY --from=builder --chown=appuser:appgroup /usr/local/bin /usr/local/bin
COPY --chown=appuser:appgroup . .

USER appuser

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1
EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=20s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health/')" || exit 1

CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000", "--workers", "4"]
```

### .dockerignore

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
__pycache__
*.pyc
.pytest_cache
.venv
```

---

## Health Checks

Liveness and readiness must use **separate endpoints**. This is the most important health check rule:

- **`/health`** (liveness) — is the process alive? Returns 200 if the app is running, even if dependencies are down. A liveness failure causes Kubernetes to restart the pod.
- **`/health/ready`** (readiness) — can the pod serve traffic? Checks dependencies. A readiness failure removes the pod from the load balancer without restarting it.

**Why this matters:** If `/health` checks the database and the database goes down, Kubernetes restarts every pod in a crash loop — which doesn't fix the database and makes things worse. The pod is alive; it just can't serve traffic yet.

```typescript
// Liveness — simple, never checks dependencies
app.get('/health', (_req, res) => {
  res.json({ status: 'ok', uptime: process.uptime(), version: process.env.APP_VERSION })
})

// Readiness — checks dependencies with individual timeouts
app.get('/health/ready', async (_req, res) => {
  const timeout = (ms: number) =>
    new Promise<never>((_, reject) =>
      setTimeout(() => reject(new Error('timeout')), ms),
    )

  const check = async (name: string, fn: () => Promise<void>): Promise<HealthResult> => {
    const start = performance.now()
    try {
      await Promise.race([fn(), timeout(2000)])   // 2s per dependency
      return { status: 'ok', latency_ms: Math.round(performance.now() - start) }
    } catch (err) {
      return {
        status: 'error',
        latency_ms: Math.round(performance.now() - start),
        message: err instanceof Error ? err.message : 'unknown',
      }
    }
  }

  const checks = {
    database: await check('database', () => db.query('SELECT 1').then(() => undefined)),
    redis:    await check('redis',    () => redis.ping().then(() => undefined)),
  }

  const allOk = Object.values(checks).every(c => c.status === 'ok')
  res.status(allOk ? 200 : 503).json({
    status: allOk ? 'ok' : 'degraded',
    timestamp: new Date().toISOString(),
    checks,
  })
})
```

### Kubernetes Probes (Separate Endpoints)

```yaml
livenessProbe:
  httpGet:
    path: /health        # simple — never checks dependencies
    port: 3000
  initialDelaySeconds: 10
  periodSeconds: 30
  failureThreshold: 3    # 3 × 30s = 90s before pod restart

readinessProbe:
  httpGet:
    path: /health/ready  # dependency check — removes from LB on failure, no restart
    port: 3000
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 2

startupProbe:
  httpGet:
    path: /health
    port: 3000
  initialDelaySeconds: 0
  periodSeconds: 5
  failureThreshold: 30   # 30 × 5s = 150s allowed for initial startup
  # startupProbe disables liveness/readiness until it passes — safe for slow starts
```

---

## CI/CD Pipeline

### GitHub Actions — Complete Pipeline

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm test -- --coverage
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: coverage-${{ github.sha }}
          path: coverage/

  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write          # required to push to GHCR
    outputs:
      image: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=,format=long
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          provenance: true       # SLSA attestation
          sbom: true             # software bill of materials

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to staging
        run: |
          # kubectl set image deployment/app app=ghcr.io/${{ github.repository }}:${{ github.sha }}
          # railway up / vercel --prod --env staging
          echo "Deploy ${{ github.sha }} → staging"

      - name: Smoke test staging
        run: |
          sleep 30   # allow time for rollout
          curl --fail --retry 5 --retry-delay 5 \
            https://staging.example.com/health || exit 1

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production      # requires manual approval gate in GitHub
    steps:
      - name: Deploy to production
        run: |
          echo "Deploy ${{ github.sha }} → production"
```

### Pipeline Stages

```
PR opened:
  lint → typecheck → unit tests → integration tests → preview deploy

Merged to main:
  lint → typecheck → unit tests → build image (SBOM + provenance)
  → deploy staging → smoke tests → [manual approval] → deploy production
```

### Secrets Management in CI/CD

Never put secrets in workflow files, repository variables, or image layers:

```yaml
# ✅ Use GitHub Environments with environment-scoped secrets
# Settings → Environments → production → Add secret

steps:
  - name: Deploy
    env:
      DATABASE_URL: ${{ secrets.DATABASE_URL }}      # environment secret
      API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}      # environment secret
    run: ./deploy.sh

# ✅ For cloud providers — use OIDC, not long-lived keys
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789:role/GitHubActions
    aws-region: us-east-1
    # No AWS_ACCESS_KEY_ID or AWS_SECRET_ACCESS_KEY needed
```

---

## Database Migrations

Database migrations are the highest-risk part of any deployment. The expand-contract pattern enables zero-downtime changes.

### Expand-Contract (Zero-Downtime) Pattern

Never make a breaking schema change in a single deployment. Split it across three:

```
Step 1 — EXPAND: add the new thing, keep the old
  → Add new column with default or nullable
  → Deploy code that writes to BOTH old and new column
  → Old code still works (reads old column)

Step 2 — MIGRATE: backfill existing data
  → Run migration to populate new column from old
  → Deploy code that reads from new column
  → Keep writing to both during transition

Step 3 — CONTRACT: remove the old thing
  → Remove old column once all traffic reads new column
  → Remove dual-write code
```

**Concrete example — renaming `full_name` to `display_name`:**

```sql
-- Step 1: Add new column (backward-compatible)
ALTER TABLE users ADD COLUMN display_name TEXT;
UPDATE users SET display_name = full_name WHERE display_name IS NULL;

-- Deploy: app writes to both full_name and display_name

-- Step 2: Verify backfill complete, switch reads to display_name
-- Deploy: app reads display_name, writes to both

-- Step 3: Remove old column (after all pods on new version)
ALTER TABLE users DROP COLUMN full_name;
-- Deploy: app only uses display_name
```

### Migration Safety Rules

```
✅ Always safe:
  - Adding a nullable column
  - Adding a column with a default value
  - Adding a new table
  - Adding an index CONCURRENTLY (Postgres)
  - Widening a column (VARCHAR(100) → VARCHAR(200))

❌ Never safe in a single deploy:
  - Dropping a column currently read by live code
  - Renaming a column used by live code
  - Adding a NOT NULL column without a default
  - Changing a column type incompatibly
  - Dropping a table used by live code
```

### Rollback Checklist

- [ ] Previous image/artifact is tagged and available
- [ ] All migrations in this release are reversible — or explicitly planned as one-way
- [ ] If one-way: the rollback plan is "fix forward", documented and tested
- [ ] Feature flags can disable new behavior without a deploy
- [ ] Staging migration tested against a production-sized data snapshot
- [ ] Rollback command verified to work before release window opens

---

## Environment Configuration

```typescript
import { z } from 'zod'

// Validate all required env vars at startup — crash immediately if misconfigured
const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'staging', 'production']),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
  APP_VERSION: z.string().default('unknown'),
})

export const env = envSchema.parse(process.env)
// If this throws, the process exits before serving any traffic — correct behavior
```

---

## Production Readiness Checklist

Run before every production release:

### Application
- [ ] All tests pass (unit, integration, E2E)
- [ ] No hardcoded secrets — secrets manager or environment injection only
- [ ] Error handling covers all edge cases; errors preserve cause chain
- [ ] Logs are structured JSON and contain no PII
- [ ] `/health` (liveness) and `/health/ready` (readiness) implemented separately

### Docker & Infrastructure
- [ ] Image builds from pinned base tags — no `:latest`
- [ ] Container runs as non-root user with correct group ownership
- [ ] `.dockerignore` excludes `.env`, `node_modules`, `.git`, test directories
- [ ] Resource limits set (CPU, memory) — no unbounded containers
- [ ] Horizontal scaling configured with min/max instance bounds
- [ ] SSL/TLS enabled on all external endpoints

### CI/CD & Secrets
- [ ] Secrets stored in environment-scoped CI secrets — not in workflow YAML
- [ ] OIDC used for cloud provider auth — no long-lived access keys in CI
- [ ] Image signed with provenance attestation (SLSA)
- [ ] Smoke test runs against staging before production gate

### Database
- [ ] Migrations use expand-contract — no destructive changes in a single deploy
- [ ] Migration tested against production-sized data snapshot
- [ ] Rollback is either reversible migration or documented fix-forward plan

### Monitoring & Alerting
- [ ] Request rate, error rate, and latency exported as metrics
- [ ] Alerts configured: error rate spike, latency P99 threshold, pod restart count
- [ ] Logs aggregated and searchable (Datadog, Loki, CloudWatch, etc.)
- [ ] Uptime monitoring on `/health` from an external probe

### Operations
- [ ] Rollback command verified in staging before the release window
- [ ] Runbook exists for: rollback, database restore, dependency outage
- [ ] On-call rotation and escalation path defined and tested

---

For structured logging implementation, metrics export, and observability setup, see `references/observability.md`.
