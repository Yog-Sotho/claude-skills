# Security Review — Advanced Patterns

File upload validation (magic bytes), CSRF, timing attack prevention, JWT verification, password hashing, SSRF, LLM prompt injection, blockchain security, and Python patterns.

---

## File Upload Security

`file.type` (browser MIME type) is user-controlled and trivially spoofed. Validate actual file content by reading magic bytes.

```typescript
// lib/file-validation.ts

const MAGIC_BYTES: Record<string, { bytes: number[]; offset: number }> = {
  'image/jpeg': { bytes: [0xFF, 0xD8, 0xFF], offset: 0 },
  'image/png':  { bytes: [0x89, 0x50, 0x4E, 0x47], offset: 0 },
  'image/gif':  { bytes: [0x47, 0x49, 0x46, 0x38], offset: 0 },
  'image/webp': { bytes: [0x57, 0x45, 0x42, 0x50], offset: 8 },
  'application/pdf': { bytes: [0x25, 0x50, 0x44, 0x46], offset: 0 },
}

async function validateFileMagicBytes(
  file: File,
  allowedTypes: string[],
): Promise<string> {
  const headerSize = 12
  const buffer = await file.slice(0, headerSize).arrayBuffer()
  const bytes = new Uint8Array(buffer)

  for (const mimeType of allowedTypes) {
    const magic = MAGIC_BYTES[mimeType]
    if (!magic) continue

    const matches = magic.bytes.every(
      (byte, i) => bytes[magic.offset + i] === byte,
    )
    if (matches) return mimeType
  }

  throw new Error(`Invalid file type — expected one of: ${allowedTypes.join(', ')}`)
}

export async function validateImageUpload(file: File): Promise<void> {
  const MAX_SIZE = 5 * 1024 * 1024  // 5MB

  if (file.size > MAX_SIZE) {
    throw new Error('File exceeds 5MB limit')
  }

  // Validate content, not browser-supplied MIME type
  await validateFileMagicBytes(file, ['image/jpeg', 'image/png', 'image/gif', 'image/webp'])

  // Sanitize filename — strip path traversal attempts
  const safeName = file.name.replace(/[^a-zA-Z0-9._-]/g, '_').replace(/\.{2,}/g, '_')
  if (safeName !== file.name) {
    throw new Error('Filename contains invalid characters')
  }
}
```

**Additional upload rules:**
- Store uploaded files outside the web root or in a separate storage bucket — never serve directly from the upload directory
- Generate a new UUID-based filename on the server — never use the user-supplied filename for storage
- Scan with antivirus for non-image uploads (documents, archives) if feasible
- Set `Content-Disposition: attachment` and correct MIME type when serving downloads

---

## CSRF Protection

SameSite cookies provide strong CSRF protection for most cases. Supplement with double-submit tokens for APIs that accept cross-origin requests or non-browser clients.

```typescript
// lib/csrf.ts
import { createHmac, randomBytes, timingSafeEqual } from 'crypto'

const CSRF_SECRET = process.env.CSRF_SECRET!   // 32+ random bytes, env var

export function generateCsrfToken(sessionId: string): string {
  const nonce = randomBytes(16).toString('hex')
  const mac = createHmac('sha256', CSRF_SECRET)
    .update(`${sessionId}:${nonce}`)
    .digest('hex')
  return `${nonce}:${mac}`
}

export function verifyCsrfToken(token: string | null, sessionId: string): boolean {
  if (!token) return false
  const [nonce, mac] = token.split(':')
  if (!nonce || !mac) return false

  const expected = createHmac('sha256', CSRF_SECRET)
    .update(`${sessionId}:${nonce}`)
    .digest('hex')

  // timingSafeEqual prevents timing attacks on the comparison
  return timingSafeEqual(Buffer.from(mac, 'hex'), Buffer.from(expected, 'hex'))
}

// Usage in API route
export async function POST(request: NextRequest) {
  const sessionId = getSessionId(request)
  const csrfToken = request.headers.get('X-CSRF-Token')

  if (!verifyCsrfToken(csrfToken, sessionId)) {
    return NextResponse.json({ error: 'Invalid CSRF token' }, { status: 403 })
  }
  // proceed
}
```

---

## Timing Attack Prevention

When comparing secrets, tokens, or hashes — always use constant-time comparison. Regular string comparison (`===`) leaks information about how many characters match through timing differences.

```typescript
import { timingSafeEqual } from 'crypto'

// ❌ WRONG: timing attack — returns faster when early characters mismatch
function validateToken(provided: string, expected: string): boolean {
  return provided === expected
}

// ✅ CORRECT: constant-time — always takes the same time regardless of match position
function validateTokenSafe(provided: string, expected: string): boolean {
  if (provided.length !== expected.length) {
    // Still compare to avoid length-based timing leak
    timingSafeEqual(Buffer.from(provided), Buffer.from(provided))
    return false
  }
  return timingSafeEqual(Buffer.from(provided), Buffer.from(expected))
}

// Use for: API keys, CSRF tokens, webhook signatures, any secret comparison
```

---

## Password Hashing

Never store plaintext passwords. Never use MD5, SHA-1, or SHA-256 for password hashing — they're designed to be fast, which makes brute-force attacks cheap.

```typescript
// Use bcrypt (well-established) or argon2 (modern recommended)
import bcrypt from 'bcrypt'

const SALT_ROUNDS = 12  // minimum; higher = slower but safer

export async function hashPassword(plain: string): Promise<string> {
  return bcrypt.hash(plain, SALT_ROUNDS)
}

export async function verifyPassword(plain: string, hash: string): Promise<boolean> {
  return bcrypt.compare(plain, hash)  // constant-time internally
}

// argon2 alternative (more resistant to GPU attacks):
import argon2 from '@node-rs/argon2'

export async function hashPasswordArgon2(plain: string): Promise<string> {
  return argon2.hash(plain, {
    type: argon2.Algorithm.Argon2id,  // Argon2id for general use
    memoryCost: 65536,                // 64MB
    timeCost: 3,
    parallelism: 4,
  })
}
```

---

## JWT Verification

Never skip verification. Pin the expected algorithm to prevent algorithm-confusion attacks (`alg: none`, HS256 when RS256 expected).

```typescript
import { jwtVerify, SignJWT } from 'jose'

const JWT_SECRET = new TextEncoder().encode(process.env.JWT_SECRET!)

export async function verifyJwt(token: string) {
  const { payload } = await jwtVerify(token, JWT_SECRET, {
    algorithms: ['HS256'],    // pin algorithm — never accept 'none'
    issuer: 'https://app.example.com',
    audience: 'app-users',
  })

  return payload as { sub: string; role: string; exp: number }
}

export async function signJwt(userId: string, role: string): Promise<string> {
  return new SignJWT({ sub: userId, role })
    .setProtectedHeader({ alg: 'HS256' })
    .setIssuedAt()
    .setExpirationTime('1h')      // short expiry — use refresh tokens for longer sessions
    .setIssuer('https://app.example.com')
    .setAudience('app-users')
    .sign(JWT_SECRET)
}
```

---

## SSRF Prevention

If your server fetches URLs provided by users, attackers can access internal services (`http://169.254.169.254/` for cloud metadata, `http://localhost:6379/` for Redis, etc.).

```typescript
import { URL } from 'url'

const ALLOWED_PROTOCOLS = new Set(['https:'])
const BLOCKED_HOSTNAMES = /^(localhost|127\.|10\.|172\.(1[6-9]|2\d|3[01])\.|192\.168\.|169\.254\.)/

export function validateExternalUrl(rawUrl: string): URL {
  let parsed: URL
  try {
    parsed = new URL(rawUrl)
  } catch {
    throw new Error('Invalid URL')
  }

  if (!ALLOWED_PROTOCOLS.has(parsed.protocol)) {
    throw new Error('Only HTTPS URLs are allowed')
  }

  if (BLOCKED_HOSTNAMES.test(parsed.hostname)) {
    throw new Error('URL resolves to a private or loopback address')
  }

  // Block numeric IPs (may bypass hostname checks)
  if (/^\d+\.\d+\.\d+\.\d+$/.test(parsed.hostname)) {
    throw new Error('Numeric IP addresses are not allowed')
  }

  return parsed
}

// Usage
const safeUrl = validateExternalUrl(userProvidedUrl)
const response = await fetch(safeUrl.toString(), {
  redirect: 'error',       // don't follow redirects — they could redirect to internal URLs
})
```

---

## LLM / Prompt Injection Security

If your application passes user content to an LLM, treat prompt injection as a first-class threat. Users can craft inputs that override your system prompt, exfiltrate data, or make the model perform unintended actions.

```typescript
// Mitigations:

// 1. Never interpolate raw user input directly into system prompts
// ❌ DANGEROUS
const systemPrompt = `You are a helpful assistant. The user's name is ${userName}.`

// ✅ SAFER: Use structured separation
const messages = [
  { role: 'system', content: 'You are a helpful assistant. Address the user formally.' },
  { role: 'user', content: userInput },  // user input in its own turn, not in system
]

// 2. Define a strict output schema and validate responses
const responseSchema = z.object({
  action: z.enum(['summarize', 'translate', 'classify']),
  result: z.string().max(2000),
})

const raw = await callLLM(messages)
const parsed = responseSchema.safeParse(JSON.parse(raw))
if (!parsed.success) {
  throw new Error('LLM response did not match expected schema')
}

// 3. Sanitize user content before including in prompts
function sanitizeForPrompt(input: string): string {
  return input
    .slice(0, 2000)                           // length limit
    .replace(/\bignore\b.*\bprevious\b/gi, '[FILTERED]')  // naive filter — defense in depth only
}

// 4. Log all prompts and responses for audit
// 5. Implement output filtering for sensitive data patterns (SSNs, card numbers)
// 6. Use a separate, lower-privileged model for user-facing tasks where possible
```

**Key rules:**
- Treat LLM outputs as untrusted user input — validate and sanitize before acting on them
- Never execute LLM-generated code without human review and sandboxing
- Scope the LLM's tool access to the minimum required for the task
- Monitor for anomalous outputs (unusually long, unexpected language, data exfiltration patterns)

---

## Python Security Patterns

### Input Validation (Pydantic)

```python
from pydantic import BaseModel, EmailStr, field_validator
from fastapi import FastAPI, HTTPException

class CreateUserRequest(BaseModel):
    email: EmailStr
    name: str
    age: int

    @field_validator('name')
    @classmethod
    def name_must_be_valid(cls, v: str) -> str:
        if len(v) < 1 or len(v) > 100:
            raise ValueError('Name must be between 1 and 100 characters')
        return v.strip()

    @field_validator('age')
    @classmethod
    def age_must_be_valid(cls, v: int) -> int:
        if not 0 <= v <= 150:
            raise ValueError('Age must be between 0 and 150')
        return v

app = FastAPI()

@app.post('/users', status_code=201)
async def create_user(body: CreateUserRequest):
    # FastAPI + Pydantic validates automatically — returns 422 on schema failure
    user = await user_service.create(email=body.email, name=body.name, age=body.age)
    return {'data': user.dict()}
```

### Password Hashing (Python)

```python
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=['argon2'], deprecated='auto')

def hash_password(plain: str) -> str:
    return pwd_context.hash(plain)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)  # constant-time internally
```

### SQL Injection (SQLAlchemy)

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

# ❌ DANGEROUS — f-string in SQL
query = f"SELECT * FROM users WHERE email = '{user_email}'"
result = await session.execute(text(query))

# ✅ SAFE — bound parameters
result = await session.execute(
    text("SELECT * FROM users WHERE email = :email"),
    {"email": user_email},
)

# ✅ SAFEST — ORM (parameterized automatically)
from sqlalchemy import select
stmt = select(User).where(User.email == user_email)
result = await session.execute(stmt)
```

### Secrets (Python)

```python
from pydantic_settings import BaseSettings
from pydantic import SecretStr

class Settings(BaseSettings):
    database_url: str
    jwt_secret: SecretStr       # SecretStr prevents accidental logging
    api_key: SecretStr

    class Config:
        env_file = '.env'

settings = Settings()                    # raises ValidationError at startup if missing
secret_value = settings.jwt_secret.get_secret_value()  # explicit access only
```

---

## Blockchain / Solana Security

### Wallet Signature Verification

The correct library for Ed25519 signature verification on Solana is `@noble/curves`, not `@solana/web3.js` (which has no top-level `verify` export):

```typescript
import { ed25519 } from '@noble/curves/ed25519'
import bs58 from 'bs58'

export function verifySolanaWallet(
  walletAddress: string,    // base58-encoded public key
  messageText: string,
  signatureBase64: string,
): boolean {
  try {
    const publicKeyBytes = bs58.decode(walletAddress)
    const messageBytes = new TextEncoder().encode(messageText)
    const signatureBytes = Buffer.from(signatureBase64, 'base64')

    return ed25519.verify(signatureBytes, messageBytes, publicKeyBytes)
  } catch {
    return false   // any decoding or verification error = invalid
  }
}
```

### Transaction Validation (Solana)

```typescript
import {
  Connection,
  PublicKey,
  ParsedTransactionWithMeta,
  LAMPORTS_PER_SOL,
} from '@solana/web3.js'

const connection = new Connection(process.env.SOLANA_RPC_URL!, 'confirmed')

export async function verifySolanaPayment(
  signature: string,
  expectedRecipient: string,
  expectedAmountSOL: number,
): Promise<boolean> {
  const tx = await connection.getParsedTransaction(signature, {
    maxSupportedTransactionVersion: 0,
  })

  if (!tx || tx.meta?.err) return false   // failed or not found

  const recipientKey = new PublicKey(expectedRecipient)
  const expectedLamports = Math.round(expectedAmountSOL * LAMPORTS_PER_SOL)

  // Check post-balance change for recipient
  const recipientIndex = tx.transaction.message.accountKeys.findIndex(
    (key) => key.pubkey.equals(recipientKey),
  )
  if (recipientIndex === -1) return false

  const preBalance  = tx.meta!.preBalances[recipientIndex]
  const postBalance = tx.meta!.postBalances[recipientIndex]
  const received    = postBalance - preBalance

  return received >= expectedLamports
}
```

**Solana security rules:**
- Never sign transactions without displaying full details to the user
- Verify all instruction accounts and data — not just the transfer amount
- Re-verify transaction on-chain after user confirmation — don't trust client-side data
- Use `maxSupportedTransactionVersion: 0` in `getParsedTransaction` to handle versioned transactions
