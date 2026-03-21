# Observability — Logging, Metrics, and Smoke Tests

Structured logging, metrics export, alerting patterns, and smoke test implementation.

---

## Structured Logging

Every log line must be machine-parseable JSON. Plain-text logs cannot be queried at scale.

### Node.js (pino — recommended)

```typescript
import pino from 'pino'

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  // Pretty-print in development; JSON in production
  transport: process.env.NODE_ENV === 'development'
    ? { target: 'pino-pretty' }
    : undefined,
  base: {
    service: process.env.SERVICE_NAME ?? 'app',
    version: process.env.APP_VERSION ?? 'unknown',
    env: process.env.NODE_ENV,
  },
  redact: {
    paths: ['req.headers.authorization', 'body.password', 'body.token'],
    censor: '[REDACTED]',
  },
})

// ✅ Structured fields — searchable in Datadog/Loki/CloudWatch
logger.info({ userId, marketId, action: 'purchase' }, 'Order placed')
logger.error({ err, requestId, userId }, 'Payment failed')

// ❌ Plain string — unsearchable
logger.info(`User ${userId} placed order for market ${marketId}`)
```

### Request Logging Middleware (Express)

```typescript
import { randomUUID } from 'crypto'

app.use((req, res, next) => {
  const requestId = randomUUID()
  const start = performance.now()

  req.log = logger.child({ requestId })   // attach to request for downstream use

  res.on('finish', () => {
    req.log.info({
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration_ms: Math.round(performance.now() - start),
    }, 'request completed')
  })

  next()
})
```

### Python (structlog — recommended)

```python
import structlog

log = structlog.get_logger()
structlog.configure(
    processors=[
        structlog.contextvars.merge_contextvars,
        structlog.processors.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer(),
    ]
)

# Usage
log.info("order.placed", user_id=user_id, market_id=market_id, amount=amount)
log.error("payment.failed", exc_info=True, user_id=user_id, request_id=request_id)
```

### Rules

- **Never log PII** — no email addresses, phone numbers, full names, or payment details in logs
- **Always include `requestId`** — makes tracing a user journey across services possible
- **Log at the boundary** — requests in/out, third-party calls, background job start/end
- **Log errors with the original exception** — not just the message
- **Use log levels correctly:**
  - `debug` — high-volume detail for development only; disabled in production
  - `info` — normal operations, business events
  - `warn` — unexpected but handled; investigate if frequent
  - `error` — requires investigation; should trigger an alert

---

## Metrics Export

### Key Metrics to Export

Every production service should export at minimum:

```
http_requests_total          # counter: by method, path, status
http_request_duration_seconds # histogram: by method, path, status
http_requests_in_flight      # gauge: concurrent requests
db_query_duration_seconds    # histogram: by query type
cache_hits_total / cache_misses_total  # counter
error_rate                   # derived: 5xx / total requests
```

### Node.js (prom-client)

```typescript
import { Counter, Histogram, Registry, collectDefaultMetrics } from 'prom-client'

const registry = new Registry()
collectDefaultMetrics({ register: registry })   // process CPU, memory, event loop

export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'],
  registers: [registry],
})

export const httpDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
  registers: [registry],
})

// Metrics endpoint — scrape with Prometheus or Datadog agent
app.get('/metrics', async (_req, res) => {
  res.set('Content-Type', registry.contentType)
  res.send(await registry.metrics())
})

// Middleware to record metrics
app.use((req, res, next) => {
  const end = httpDuration.startTimer()
  res.on('finish', () => {
    const route = req.route?.path ?? 'unknown'
    const labels = { method: req.method, route, status: res.statusCode }
    httpRequestsTotal.inc(labels)
    end(labels)
  })
  next()
})
```

---

## Alerting Thresholds

Configure these as a baseline — tune based on your baseline metrics:

| Signal | Alert Condition | Severity |
|---|---|---|
| Error rate | > 1% of requests are 5xx for 5 minutes | 🔴 Critical |
| Latency P99 | > 2s for 10 minutes | 🔴 Critical |
| Pod restarts | > 3 restarts in 10 minutes | 🔴 Critical |
| Error rate | > 0.1% of requests are 5xx for 15 minutes | 🟡 Warning |
| Latency P95 | > 500ms for 15 minutes | 🟡 Warning |
| Disk usage | > 80% on any persistent volume | 🟡 Warning |
| Memory usage | > 85% of limit for 5 minutes | 🟡 Warning |

---

## Smoke Tests

A smoke test is a minimal set of checks run against the deployed environment to verify the deployment succeeded before routing production traffic.

### What to Test

- Health endpoint returns 200
- At least one authenticated API route works end-to-end
- At least one database-backed read works
- Static assets load (if applicable)

### Implementation (shell — runs in CI after deploy)

```bash
#!/usr/bin/env bash
# scripts/smoke-test.sh

set -euo pipefail

BASE_URL="${1:-https://staging.example.com}"
MAX_RETRIES=10
RETRY_DELAY=10

echo "→ Smoke testing: $BASE_URL"

# 1. Liveness check with retry (allows time for rollout to complete)
for i in $(seq 1 $MAX_RETRIES); do
  if curl --silent --fail --max-time 5 "$BASE_URL/health" > /dev/null; then
    echo "✅ /health OK"
    break
  fi
  echo "   Attempt $i/$MAX_RETRIES failed, retrying in ${RETRY_DELAY}s..."
  sleep $RETRY_DELAY
  if [ $i -eq $MAX_RETRIES ]; then
    echo "❌ /health never responded"
    exit 1
  fi
done

# 2. Readiness check — dependencies healthy
STATUS=$(curl --silent --max-time 10 "$BASE_URL/health/ready" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['status'])" 2>/dev/null || echo "error")
if [ "$STATUS" != "ok" ]; then
  echo "❌ /health/ready returned: $STATUS"
  exit 1
fi
echo "✅ /health/ready OK"

# 3. Authenticated API spot check
HTTP_CODE=$(curl --silent --output /dev/null --write-out "%{http_code}" \
  --header "Authorization: Bearer $SMOKE_TEST_TOKEN" \
  --max-time 10 \
  "$BASE_URL/api/v1/users/me")
if [ "$HTTP_CODE" != "200" ]; then
  echo "❌ /api/v1/users/me returned HTTP $HTTP_CODE"
  exit 1
fi
echo "✅ /api/v1/users/me OK"

echo ""
echo "✅ All smoke tests passed — $BASE_URL is healthy"
```

### Integrate in GitHub Actions

```yaml
- name: Smoke test staging
  env:
    SMOKE_TEST_TOKEN: ${{ secrets.SMOKE_TEST_TOKEN }}
  run: |
    chmod +x scripts/smoke-test.sh
    ./scripts/smoke-test.sh https://staging.example.com
```

---

## Runbook Template

Every service should have a runbook covering at minimum:

```markdown
# [Service Name] Runbook

## Rollback
kubectl rollout undo deployment/[name]
# or: railway up --commit <previous-sha>
# Verify: curl https://example.com/health

## Database restore
# 1. Point-in-time restore: [cloud console URL or CLI command]
# 2. Run migrations: npx prisma migrate deploy
# 3. Verify: curl https://example.com/health/ready

## Dependency outage

### Database unreachable
- Check: kubectl exec -it [pod] -- pg_isready -h $DB_HOST
- App will return 503 on /health/ready; liveness (/health) stays green
- Pods will NOT restart (correct behavior)
- Action: investigate DB host; scale down app writes if needed

### Redis unreachable  
- App degrades gracefully (falls back to DB if implemented)
- Alert: cache_miss_rate spike
- Action: restart Redis / failover to replica

## Alert response

### Error rate > 1% (5xx)
1. Check recent deployments: kubectl rollout history deployment/[name]
2. Check error logs: [log query link]
3. If new deploy: kubectl rollout undo deployment/[name]
4. If pre-existing: escalate to on-call lead

### Latency P99 > 2s
1. Check DB query latency: [metrics dashboard link]
2. Check pod resource usage: kubectl top pods
3. Check downstream dependencies: /health/ready
```
