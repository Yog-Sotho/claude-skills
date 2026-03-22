---
name: api-design
description: REST API design patterns including resource naming, HTTP methods, status codes, pagination, filtering, error responses, versioning, rate limiting, idempotency, and security. Always activate when the user is designing or reviewing API endpoints, adding pagination or filtering, implementing error handling, planning versioning, building public or partner-facing APIs, or asks how to structure a route, response, or status code — even if they don't use the word "API design".
---

# API Design Patterns

Conventions and best practices for designing consistent, secure, developer-friendly REST APIs.

## Workflow

When this skill activates:

1. **Identify the scope** — new endpoint design, review of existing API, adding pagination/filtering, versioning strategy, or security hardening.
2. **Apply the checklist** at the bottom before marking any endpoint complete.
3. **Adapt the implementation examples** to the user's stack — TypeScript, Python, and Go patterns are all provided.
4. **Flag violations proactively** if spotted in user-provided code — wrong status codes, unhandled JSON parse errors, tokens in query params, and missing auth checks are the most common.
5. **For public or partner APIs**: always address versioning, deprecation timeline, rate limiting, and OpenAPI documentation.

---

## Resource Design

### URL Structure

```
# Resources are nouns — plural, lowercase, kebab-case
GET    /api/v1/users
GET    /api/v1/users/:id
POST   /api/v1/users
PUT    /api/v1/users/:id        # full replacement
PATCH  /api/v1/users/:id        # partial update
DELETE /api/v1/users/:id

# Sub-resources for ownership relationships
GET    /api/v1/users/:id/orders
POST   /api/v1/users/:id/orders

# Actions that don't map to CRUD — use verbs sparingly, always POST
POST   /api/v1/orders/:id/cancel
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
```

### Naming Rules

```
# GOOD
/api/v1/team-members          # kebab-case for multi-word
/api/v1/orders?status=active  # filtering via query params
/api/v1/users/123/orders      # nested for ownership

# BAD
/api/v1/getUsers              # verb in URL
/api/v1/user                  # singular
/api/v1/team_members          # snake_case in URLs
/api/v1/users/123/getOrders   # verb in nested resource
```

---

## HTTP Methods and Status Codes

### Method Semantics

| Method | Idempotent | Safe | Use For |
|--------|-----------|------|---------|
| GET    | Yes | Yes | Retrieve resources |
| POST   | No  | No  | Create resources, trigger actions |
| PUT    | Yes | No  | Full replacement of a resource |
| PATCH  | No* | No  | Partial update |
| DELETE | Yes | No  | Remove a resource |

*PATCH can be made idempotent with a conditional `If-Match` header.

### Status Code Reference

```
# Success
200 OK                    — GET, PUT, PATCH (with response body)
201 Created               — POST (include Location header)
204 No Content            — DELETE, PUT (no response body)

# Client Errors
400 Bad Request           — Malformed JSON, invalid syntax
401 Unauthorized          — Missing or invalid authentication
403 Forbidden             — Authenticated but not authorized
404 Not Found             — Resource doesn't exist
409 Conflict              — Duplicate entry, state conflict
422 Unprocessable Entity  — Valid JSON but semantically invalid data
429 Too Many Requests     — Rate limit exceeded

# Server Errors
500 Internal Server Error — Unexpected failure (never expose internal details)
502 Bad Gateway           — Upstream service failed
503 Service Unavailable   — Overloaded; include Retry-After header
```

### Common Mistakes

```
# BAD: 200 for everything
{ "status": 200, "success": false, "error": "Not found" }

# GOOD: semantic HTTP status codes
HTTP/1.1 404 Not Found
{ "error": { "code": "not_found", "message": "User not found" } }

# BAD: 500 for validation errors → GOOD: 400 or 422 with field-level details
# BAD: 200 for created resources → GOOD: 201 with Location header
```

---

## Response Format

### Single Resource

```json
{
  "data": {
    "id": "abc-123",
    "email": "alice@example.com",
    "name": "Alice",
    "created_at": "2025-01-15T10:30:00Z"
  }
}
```

### Collection (with Pagination)

```json
{
  "data": [
    { "id": "abc-123", "name": "Alice" },
    { "id": "def-456", "name": "Bob" }
  ],
  "meta": {
    "total": 142,
    "page": 1,
    "per_page": 20,
    "total_pages": 8
  },
  "links": {
    "self": "/api/v1/users?page=1&per_page=20",
    "next": "/api/v1/users?page=2&per_page=20",
    "last": "/api/v1/users?page=8&per_page=20"
  }
}
```

### Error Response

```json
{
  "error": {
    "code": "validation_error",
    "message": "Request validation failed",
    "details": [
      { "field": "email", "message": "Must be a valid email address", "code": "invalid_format" },
      { "field": "age",   "message": "Must be between 0 and 150",     "code": "out_of_range" }
    ]
  }
}
```

**Rules:**
- Always return `Content-Type: application/json`
- Never expose stack traces, SQL errors, or internal service names in error responses
- Machine-readable `code` field enables client-side handling without string matching on `message`

### TypeScript Envelope Types

```typescript
interface ApiResponse<T> {
  data: T;
  meta?: PaginationMeta;
  links?: PaginationLinks;
}

interface ApiError {
  error: {
    code: string;
    message: string;
    details?: Array<{ field: string; message: string; code: string }>;
  };
}
```

---

## Pagination

### Offset-Based (Simple)

```
GET /api/v1/users?page=2&per_page=20

SELECT * FROM users ORDER BY created_at DESC LIMIT 20 OFFSET 20;
```

**Pros:** Easy to implement, supports "jump to page N"
**Cons:** Slow on large offsets (`OFFSET 100000`), inconsistent with concurrent inserts

### Cursor-Based (Scalable)

```
GET /api/v1/users?cursor=eyJpZCI6MTIzfQ&limit=20
```

The cursor is an opaque base64-encoded JSON blob. Always encode/decode explicitly:

```typescript
// Encode cursor
const cursor = Buffer.from(JSON.stringify({ id: lastItem.id })).toString("base64url");

// Decode cursor in next request
const { id: cursorId } = JSON.parse(Buffer.from(cursor, "base64url").toString());

// Query: fetch one extra item to determine has_next
const rows = await db.query(
  "SELECT * FROM users WHERE id > $1 ORDER BY id ASC LIMIT $2",
  [cursorId, limit + 1],
);
const hasNext = rows.length > limit;
const data = hasNext ? rows.slice(0, limit) : rows;
```

```json
{
  "data": [...],
  "meta": {
    "has_next": true,
    "next_cursor": "eyJpZCI6MTQzfQ"
  }
}
```

**Pros:** Consistent performance at any depth, stable with concurrent inserts
**Cons:** Cannot jump to arbitrary page; cursor is opaque to clients

### When to Use Which

| Use Case | Type |
|----------|------|
| Admin dashboards, small datasets (<10K rows) | Offset |
| Infinite scroll, feeds, large datasets | Cursor |
| Public APIs (default) | Cursor |
| Search results (users expect page numbers) | Offset |

---

## Filtering, Sorting, and Search

```
# Simple equality
GET /api/v1/orders?status=active&customer_id=abc-123

# Comparison operators — bracket notation
GET /api/v1/products?price[gte]=10&price[lte]=100
GET /api/v1/orders?created_at[after]=2025-01-01

# Multiple values — comma-separated
GET /api/v1/products?category=electronics,clothing

# Sorting — prefix - for descending, comma-separated for multiple fields
GET /api/v1/products?sort=-created_at
GET /api/v1/products?sort=-featured,price,-created_at

# Full-text search
GET /api/v1/products?q=wireless+headphones

# Sparse fieldsets — reduce payload
GET /api/v1/users?fields=id,name,email
```

---

## Authentication and Authorization

```
# Bearer token — always in Authorization header, never in query params
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

# API key — server-to-server
X-API-Key: sk_live_abc123

# ⚠️ Never in query string — shows up in logs, browser history, referrer headers
# BAD: GET /api/v1/data?token=sk_live_abc123
```

```typescript
// Resource-level: check ownership before returning
app.get("/api/v1/orders/:id", async (req, res) => {
  const order = await Order.findById(req.params.id);
  if (!order) return res.status(404).json({ error: { code: "not_found" } });
  if (order.userId !== req.user.id) return res.status(403).json({ error: { code: "forbidden" } });
  return res.json({ data: order });
});

// Role-based: guard with middleware
app.delete("/api/v1/users/:id", requireRole("admin"), async (req, res) => {
  await User.delete(req.params.id);
  return res.status(204).send();
});
```

---

## Idempotency Keys

For non-idempotent POST operations (payments, email sends, order creation), accept an `Idempotency-Key` header. If the same key is seen again within a TTL window, return the stored response instead of re-executing.

```
POST /api/v1/payments
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000

# On retry with same key → return original response, do not charge twice
```

```typescript
app.post("/api/v1/payments", async (req, res) => {
  const idempotencyKey = req.headers["idempotency-key"];
  if (idempotencyKey) {
    const cached = await cache.get(`idempotency:${idempotencyKey}`);
    if (cached) return res.status(cached.status).json(cached.body);
  }

  const result = await processPayment(req.body);
  const response = { data: result };

  if (idempotencyKey) {
    await cache.set(`idempotency:${idempotencyKey}`, { status: 201, body: response }, { ttl: 86400 });
  }

  return res.status(201).json(response);
});
```

---

## Rate Limiting

### Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1735689600    # Unix epoch seconds

HTTP/1.1 429 Too Many Requests
Retry-After: 60
{ "error": { "code": "rate_limit_exceeded", "message": "Retry in 60 seconds." } }
```

### Tiers

| Tier | Limit | Window | Use Case |
|------|-------|--------|----------|
| Anonymous | 30/min | Per IP | Public endpoints |
| Authenticated | 100/min | Per user | Standard access |
| Premium | 1000/min | Per API key | Paid plans |
| Internal | 10000/min | Per service | Service-to-service |

---

## Security

### CORS

```typescript
// Allow only known origins — never wildcard on authenticated APIs
const allowedOrigins = ["https://app.example.com", "https://admin.example.com"];

app.use((req, res, next) => {
  const origin = req.headers.origin;
  if (origin && allowedOrigins.includes(origin)) {
    res.setHeader("Access-Control-Allow-Origin", origin);
  }
  res.setHeader("Access-Control-Allow-Methods", "GET, POST, PATCH, DELETE");
  res.setHeader("Access-Control-Allow-Headers", "Authorization, Content-Type, Idempotency-Key");
  next();
});
```

### Request Hardening

```typescript
import express from "express";

app.use(express.json({ limit: "1mb" }));       // cap request body size
app.use(helmet());                              // security headers (X-Frame-Options, etc.)

// Always enforce Content-Type on mutation endpoints
app.use((req, res, next) => {
  if (["POST", "PUT", "PATCH"].includes(req.method)) {
    if (!req.is("application/json")) {
      return res.status(415).json({ error: { code: "unsupported_media_type" } });
    }
  }
  next();
});
```

### Never Leak Internals

```typescript
// ❌ Exposes DB schema, query, and stack trace
catch (err) {
  res.status(500).json({ error: err.message });
}

// ✅ Log internally, return safe message
catch (err) {
  logger.error({ err, requestId: req.id }, "Unhandled error");
  res.status(500).json({ error: { code: "internal_error", message: "An unexpected error occurred" } });
}
```

---

## Health Endpoints

Every production API needs at minimum a liveness check:

```
GET /health       → 200 { "status": "ok" }              (liveness — is the process up?)
GET /health/ready → 200 { "status": "ok", "db": "ok" }  (readiness — can it serve traffic?)
               or → 503 { "status": "degraded", "db": "timeout" }
```

```typescript
app.get("/health", (req, res) => {
  res.json({ status: "ok", version: process.env.APP_VERSION });
});

app.get("/health/ready", async (req, res) => {
  const dbOk = await db.ping().then(() => true).catch(() => false);
  const status = dbOk ? 200 : 503;
  res.status(status).json({ status: dbOk ? "ok" : "degraded", db: dbOk ? "ok" : "unreachable" });
});
```

---

## Versioning

### URL Path (Recommended)

```
/api/v1/users
/api/v2/users
```

**Pros:** Explicit, easy to route, cacheable, visible in logs
**Cons:** URL changes between versions

### Header Versioning

```
GET /api/users
Accept: application/vnd.myapp.v2+json
```

**Pros:** Clean URLs. **Cons:** Easy to forget, harder to test, not visible in logs.

### Deprecation Strategy

```
1. Start with /api/v1/ — don't version until you need a breaking change
2. Maintain at most 2 active versions (current + previous)
3. Deprecation timeline (public APIs):
   - Announce deprecation with 6 months notice
   - Add: Deprecation: true and Sunset: Sat, 01 Jan 2026 00:00:00 GMT headers
   - Return 410 Gone after sunset date
4. Non-breaking changes (no new version needed):
   - Adding new fields to responses
   - Adding new optional query parameters
   - Adding new endpoints
5. Breaking changes (require new version):
   - Removing or renaming fields
   - Changing field types
   - Changing URL structure
   - Changing authentication method
```

---

## Implementation Examples

Read `references/implementations.md` for complete working route handlers in:
- **TypeScript (Next.js)** — POST handler with JSON parse guard, Zod validation, CORS middleware, idempotency key pattern, cursor pagination
- **Python (FastAPI)** — POST handler with Pydantic, global exception handler, cursor pagination with SQLAlchemy
- **Go (net/http)** — POST handler with error mapping, `writeJSON`/`writeError` helpers, health endpoints

---

## API Design Checklist

Before shipping any endpoint:

- [ ] URL is plural, kebab-case, no verbs — actions use POST
- [ ] Correct HTTP method (GET for reads, POST for creates, etc.)
- [ ] Appropriate status code (not 200 for everything; 201 + Location for creates)
- [ ] Request body JSON parse error returns 400, not 500
- [ ] Input validated with schema (Zod, Pydantic, etc.); validation errors return 422
- [ ] Error response follows standard format (`code`, `message`, optional `details`)
- [ ] Internal errors never exposed (no stack traces, SQL, service names in responses)
- [ ] Pagination implemented for list endpoints (cursor or offset)
- [ ] Auth required — or explicitly documented as public
- [ ] Resource-level authorization checked (user can only see their own data)
- [ ] Rate limiting configured; `X-RateLimit-*` headers present
- [ ] CORS origin whitelist configured (no `*` on authenticated APIs)
- [ ] Request body size capped; `Content-Type: application/json` enforced
- [ ] Tokens and keys only in headers — never in query string
- [ ] POST operations with side effects protected by idempotency key
- [ ] Naming consistent with existing endpoints (casing, field names)
- [ ] `GET /health` implemented; `/health/ready` if DB/dependencies are involved
- [ ] OpenAPI/Swagger spec updated with new endpoint and schema
