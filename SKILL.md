---
name: security-review
description: Security review checklist and implementation patterns for web applications — secrets management, input validation, authentication, authorization, SQL injection, XSS, CSRF, rate limiting, security headers, and sensitive data handling. Always activate when the user is implementing authentication or authorization, handling user input or file uploads, creating API endpoints, working with secrets or credentials, implementing payment features, storing or transmitting sensitive data, or integrating third-party APIs. Also activate proactively when spotting hardcoded secrets, missing auth checks, or unsafe data handling in user code.
---

# Security Review

Security patterns and checklists for production web applications. One missed check can compromise everything — apply this skill completely, not selectively.

## Workflow

When this skill activates:

1. **Identify the threat surface** — what is being built or reviewed? Authentication, API endpoint, file upload, data storage, payment flow?
2. **Apply the relevant sections** — navigate to the sections that apply. Don't skip sections that seem unlikely to matter.
3. **Flag issues proactively** — if spotted in user code, call them out immediately with the corrected pattern.
4. **Run the pre-deployment checklist** before any production release.
5. **For advanced topics** — file upload magic bytes validation, CSRF implementation, timing attacks, password hashing, JWT verification, SSRF prevention, LLM prompt injection, and Python patterns — see `references/advanced.md`.

---

## 1. Secrets Management

```typescript
// ❌ NEVER: Hardcoded secrets — exposed in git history forever
const apiKey = "sk-proj-xxxxx"
const dbPassword = "password123"

// ✅ ALWAYS: Environment variables with startup validation
import { z } from 'zod'

const env = z.object({
  OPENAI_API_KEY: z.string().min(1),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32, 'JWT_SECRET must be at least 32 characters'),
}).parse(process.env)   // throws at startup if misconfigured — correct behavior
```

**Checklist:**
- [ ] No hardcoded API keys, tokens, or passwords anywhere in the codebase
- [ ] All secrets in environment variables; validated at startup
- [ ] `.env`, `.env.local`, `.env.*` in `.gitignore` — added before first commit
- [ ] Secrets not in git history (check with `git log -S "sk-"`)
- [ ] Production secrets in platform secrets manager (Vercel, Railway, AWS Secrets Manager)
- [ ] Secrets never logged, even at debug level

---

## 2. Input Validation

Validate every input at the boundary — before it touches your database, filesystem, or business logic. Use allow-lists, not block-lists.

```typescript
import { z } from 'zod'

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100).trim(),
  age: z.number().int().min(0).max(150),
})

export async function POST(request: NextRequest) {
  let body: unknown
  try {
    body = await request.json()
  } catch {
    return NextResponse.json({ error: 'Invalid JSON' }, { status: 400 })
  }

  const result = CreateUserSchema.safeParse(body)
  if (!result.success) {
    return NextResponse.json(
      { error: 'Validation failed', issues: result.error.issues },
      { status: 422 },
    )
  }

  // result.data is fully typed and validated
  return NextResponse.json({ data: await createUser(result.data) })
}
```

**Checklist:**
- [ ] All user inputs validated with a schema library (Zod, Pydantic, Joi)
- [ ] `request.json()` wrapped in try/catch — malformed JSON throws before validation
- [ ] Validation uses allowlist (permitted values) not blocklist (forbidden values)
- [ ] Error messages describe the problem without exposing internal structure

---

## 3. SQL Injection Prevention

Parameterized queries are non-negotiable. String interpolation in SQL is always wrong, regardless of what the input is.

```typescript
// ❌ DANGEROUS — SQL injection: user can inject arbitrary SQL
const query = `SELECT * FROM users WHERE email = '${userEmail}'`
await db.query(query)

// ✅ SAFE — parameterized query, user input never interpreted as SQL
await db.query('SELECT * FROM users WHERE email = $1', [userEmail])

// ✅ SAFE — ORM/query builder handles parameterization automatically
const user = await db.users.findUnique({ where: { email: userEmail } })

// ✅ SAFE — Supabase
const { data } = await supabase.from('users').select('*').eq('email', userEmail)
```

**Checklist:**
- [ ] No string concatenation or interpolation in SQL queries — ever
- [ ] All raw queries use `$1` / `?` placeholders with separate parameter arrays
- [ ] ORM or query builder used for all database access where possible
- [ ] `LIMIT` applied to all queries that could return unbounded rows

---

## 4. Authentication & Authorization

### Token Storage

```typescript
// ❌ WRONG: localStorage is accessible to any JavaScript on the page
// One XSS vulnerability = full token theft
localStorage.setItem('token', jwt)

// ✅ CORRECT: httpOnly cookies — inaccessible to JavaScript entirely
res.setHeader('Set-Cookie', [
  `session=${jwt}; HttpOnly; Secure; SameSite=Strict; Max-Age=3600; Path=/`,
])
```

### Authorization Checks

Always verify the user exists **and** has the required permission before proceeding:

```typescript
export async function deleteUser(userId: string, requesterId: string) {
  const requester = await db.users.findUnique({ where: { id: requesterId } })

  // Check existence before accessing properties
  if (!requester) {
    return NextResponse.json({ error: 'Requester not found' }, { status: 401 })
  }

  if (requester.role !== 'admin') {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }

  // Also verify the target user exists — prevents enumeration through timing
  const target = await db.users.findUnique({ where: { id: userId } })
  if (!target) {
    return NextResponse.json({ error: 'Not found' }, { status: 404 })
  }

  await db.users.delete({ where: { id: userId } })
  return NextResponse.json({ success: true })
}
```

### Row Level Security (Supabase)

Enable RLS on every table — default-deny, then grant what's needed:

```sql
-- Enable RLS on all tables (run on every new table)
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Default deny is automatic once RLS is enabled
-- Add explicit policies for what IS allowed:

CREATE POLICY "users_select_own"
  ON users FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "users_update_own"
  ON users FOR UPDATE
  USING (auth.uid() = id)
  WITH CHECK (auth.uid() = id);   -- WITH CHECK prevents escalating own role

-- Admins can do anything:
CREATE POLICY "admins_all"
  ON users FOR ALL
  USING (EXISTS (
    SELECT 1 FROM users WHERE id = auth.uid() AND role = 'admin'
  ));
```

**Checklist:**
- [ ] Tokens in httpOnly cookies — not localStorage or sessionStorage
- [ ] Auth check: existence AND permission verified before every sensitive operation
- [ ] Null guard on user lookup before accessing `.role` or any property
- [ ] RLS enabled on all Supabase tables — verify with `SELECT tablename FROM pg_tables WHERE schemaname = 'public'`
- [ ] `WITH CHECK` on UPDATE policies to prevent self-escalation
- [ ] Session expiry enforced — tokens have a short `Max-Age`

---

## 5. XSS Prevention

React escapes variables in JSX by default — but `dangerouslySetInnerHTML` bypasses this entirely.

```typescript
import DOMPurify from 'isomorphic-dompurify'

// ❌ DANGEROUS: raw HTML from user input
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// ✅ SAFE: sanitize before rendering, with strict allowlist
function SafeUserContent({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br'],
    ALLOWED_ATTR: [],   // No attributes — prevents href="javascript:", onload=, etc.
  })
  return <div dangerouslySetInnerHTML={{ __html: clean }} />
}

// ✅ SAFEST: render as text, not HTML — use this whenever HTML is not required
<p>{userInput}</p>
```

### Content Security Policy

A properly configured CSP is a critical XSS defense layer. `'unsafe-inline'` and `'unsafe-eval'` in `script-src` **completely defeat** the protection:

```typescript
// next.config.ts
const cspHeader = [
  "default-src 'self'",
  "script-src 'self'",          // NO 'unsafe-inline', NO 'unsafe-eval'
  "style-src 'self' 'unsafe-inline'",  // unsafe-inline for styles is lower risk
  "img-src 'self' data: https:",
  "font-src 'self'",
  "connect-src 'self' https://api.example.com",
  "frame-ancestors 'none'",     // prevents clickjacking
  "base-uri 'self'",            // prevents base tag injection
  "form-action 'self'",
].join('; ')

// For apps with Next.js inline scripts — use nonces instead of unsafe-inline:
// https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy
```

**Checklist:**
- [ ] `dangerouslySetInnerHTML` never used with unsanitized input
- [ ] DOMPurify with strict `ALLOWED_TAGS` / empty `ALLOWED_ATTR` for any HTML rendering
- [ ] CSP header configured — no `unsafe-eval` or `unsafe-inline` in `script-src`
- [ ] React's default JSX escaping relied upon for all non-HTML content

---

## 6. Rate Limiting

### Next.js App Router (Upstash)

`express-rate-limit` is Express middleware and does not work in Next.js App Router. Use `@upstash/ratelimit` with Redis:

```typescript
// lib/rate-limit.ts
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

export const rateLimiter = new Ratelimit({
  redis: Redis.fromEnv(),   // UPSTASH_REDIS_REST_URL + UPSTASH_REDIS_REST_TOKEN
  limiter: Ratelimit.slidingWindow(100, '15 m'),
  analytics: true,
})

export const searchLimiter = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '1 m'),
})

// Usage in API route
export async function GET(request: NextRequest) {
  const ip = request.headers.get('x-forwarded-for') ?? '127.0.0.1'
  const { success, limit, remaining, reset } = await rateLimiter.limit(ip)

  if (!success) {
    return NextResponse.json(
      { error: 'Too many requests' },
      {
        status: 429,
        headers: {
          'X-RateLimit-Limit': String(limit),
          'X-RateLimit-Remaining': String(remaining),
          'X-RateLimit-Reset': new Date(reset).toISOString(),
          'Retry-After': String(Math.ceil((reset - Date.now()) / 1000)),
        },
      },
    )
  }
  // proceed
}
```

**Checklist:**
- [ ] Rate limiting on all public API endpoints
- [ ] Stricter limits on expensive or sensitive operations (auth, search, email)
- [ ] Rate limit by authenticated user ID when user is known — not just IP
- [ ] `Retry-After` header returned on 429 responses
- [ ] Middleware or Redis-backed (not in-memory — resets on server restart)

---

## 7. Security Headers

```typescript
// next.config.ts
const securityHeaders = [
  { key: 'X-DNS-Prefetch-Control',   value: 'on' },
  { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
  { key: 'X-Frame-Options',           value: 'DENY' },
  { key: 'X-Content-Type-Options',    value: 'nosniff' },
  { key: 'Referrer-Policy',           value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy',        value: 'camera=(), microphone=(), geolocation=()' },
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self'",
      "style-src 'self' 'unsafe-inline'",
      "img-src 'self' data: https:",
      "font-src 'self'",
      "connect-src 'self' https://api.example.com",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'",
    ].join('; '),
  },
]

export default {
  async headers() {
    return [{ source: '/(.*)', headers: securityHeaders }]
  },
}
```

**Checklist:**
- [ ] HSTS: `max-age=63072000; includeSubDomains; preload`
- [ ] X-Frame-Options: `DENY` (or use `frame-ancestors 'none'` in CSP)
- [ ] X-Content-Type-Options: `nosniff`
- [ ] CSP configured without `unsafe-eval` or `unsafe-inline` in `script-src`
- [ ] Referrer-Policy restricts outgoing referrer headers
- [ ] Permissions-Policy disables APIs the app doesn't use

---

## 8. CORS

```typescript
// middleware.ts — Next.js
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

const ALLOWED_ORIGINS = [
  'https://app.example.com',
  process.env.NODE_ENV === 'development' ? 'http://localhost:3000' : null,
].filter(Boolean) as string[]

export function middleware(request: NextRequest) {
  const origin = request.headers.get('origin') ?? ''
  const isAllowed = ALLOWED_ORIGINS.includes(origin)

  const response = NextResponse.next()
  if (isAllowed) {
    response.headers.set('Access-Control-Allow-Origin', origin)
    response.headers.set('Access-Control-Allow-Credentials', 'true')
    response.headers.set('Access-Control-Allow-Methods', 'GET, POST, PUT, PATCH, DELETE, OPTIONS')
    response.headers.set('Access-Control-Allow-Headers', 'Content-Type, Authorization, X-CSRF-Token')
  }

  if (request.method === 'OPTIONS') {
    return new NextResponse(null, { status: 204, headers: response.headers })
  }

  return response
}
```

**Checklist:**
- [ ] CORS never set to `*` on authenticated APIs
- [ ] Allowed origins explicitly listed — not derived from `Origin` header without validation
- [ ] Preflight OPTIONS responses handled
- [ ] `Access-Control-Allow-Credentials: true` only set when cookies are needed

---

## 9. Sensitive Data Exposure

```typescript
// ❌ WRONG: leaking internal details and sensitive values
catch (error) {
  console.log('Login attempt:', { email, password })   // password in logs!
  return NextResponse.json({ error: error.message, stack: error.stack }, { status: 500 })
}

// ✅ CORRECT: redact sensitive fields, generic user-facing errors
catch (error) {
  logger.error({ err: error, userId, action: 'login' }, 'Login failed')  // no password
  return NextResponse.json(
    { error: 'An error occurred. Please try again.' },
    { status: 500 },
  )
}

// ✅ Redact sensitive fields in logs
logger.info({
  userId: user.id,
  email: user.email,          // email OK for user identification
  // password: NEVER log
  // cardNumber: NEVER log — use last4 only
  payment: { last4: card.last4, brand: card.brand },
})
```

**Checklist:**
- [ ] No passwords, tokens, secrets, or CVV/card numbers in any log output
- [ ] Error responses to clients are generic — no stack traces, SQL errors, or internal paths
- [ ] Detailed errors logged server-side only with structured logging
- [ ] PII handled per applicable regulations (GDPR, HIPAA) — minimized and protected

---

## 10. Dependency Security

```bash
# Audit for known CVEs
npm audit

# Auto-fix safe updates
npm audit fix

# Check for outdated packages
npm outdated

# Always use ci in CI/CD — installs exactly what's in lock file
npm ci
```

**Checklist:**
- [ ] `package-lock.json` / `yarn.lock` committed and up to date
- [ ] `npm ci` used in CI/CD — not `npm install`
- [ ] Dependabot or Renovate enabled for automated security updates
- [ ] `npm audit` runs in CI pipeline — build fails on high-severity CVEs
- [ ] No packages installed from unknown or unofficial registries

---

## Security Testing

```typescript
// Required minimum test coverage for any auth surface:

test('protected route returns 401 without auth', async () => {
  const response = await fetch('/api/protected')
  expect(response.status).toBe(401)
})

test('admin route returns 403 for non-admin user', async () => {
  const response = await fetch('/api/admin', {
    headers: { Authorization: `Bearer ${regularUserToken}` },
  })
  expect(response.status).toBe(403)
})

test('rejects invalid input with 422', async () => {
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email: 'not-an-email' }),
  })
  expect(response.status).toBe(422)
})

test('rejects malformed JSON with 400', async () => {
  const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: 'not json {{{',
  })
  expect(response.status).toBe(400)
})

test('enforces rate limit after threshold', async () => {
  // Run enough requests to exceed the limit
  const requests = Array.from({ length: 20 }, () =>
    fetch('/api/search', { headers: { 'X-Forwarded-For': '1.2.3.4' } }),
  )
  const responses = await Promise.all(requests)
  expect(responses.some(r => r.status === 429)).toBe(true)
})

test('error response does not leak stack trace', async () => {
  // Trigger an internal error
  const response = await fetch('/api/trigger-error')
  const body = await response.json()
  expect(body).not.toHaveProperty('stack')
  expect(body.error).not.toMatch(/at Object|node_modules/)
})
```

---

## Pre-Deployment Security Checklist

Before any production release:

- [ ] **Secrets**: No hardcoded values; startup validation catches missing vars
- [ ] **Input validation**: All inputs validated with schemas; JSON parse guarded
- [ ] **SQL injection**: All queries parameterized; no interpolation
- [ ] **Auth**: Tokens in httpOnly cookies; null guard before `.role` access
- [ ] **Authorization**: Existence + permission checked; RLS enabled in Supabase; `WITH CHECK` on UPDATE policies
- [ ] **XSS**: `dangerouslySetInnerHTML` only with DOMPurify; CSP without `unsafe-inline`/`unsafe-eval` in `script-src`
- [ ] **Rate limiting**: Redis-backed; covers all public endpoints; `Retry-After` header on 429
- [ ] **Security headers**: HSTS, X-Frame-Options, X-Content-Type-Options, CSP, Referrer-Policy
- [ ] **CORS**: Explicit origin allowlist; never `*` on authenticated APIs
- [ ] **Sensitive data**: No PII, passwords, or tokens in logs; generic error messages to clients
- [ ] **HTTPS**: Enforced in production; HSTS preload submitted
- [ ] **Dependencies**: `npm audit` clean; `npm ci` in CI; Dependabot enabled
- [ ] **Security tests**: Auth, authz, validation, and error-leak tests all pass

---

## Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [Next.js Security](https://nextjs.org/docs/app/building-your-application/configuring/content-security-policy)
- [Supabase RLS Guide](https://supabase.com/docs/guides/auth/row-level-security)
- [Web Security Academy](https://portswigger.net/web-security)

For advanced topics — file upload magic bytes, CSRF implementation, timing attacks, JWT verification, password hashing, SSRF, LLM prompt injection, and Python patterns — see `references/advanced.md`.
