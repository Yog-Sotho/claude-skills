# TypeScript — Advanced Patterns

Barrel exports, Result types, advanced generics, and utility type patterns.

---

## Barrel Exports (`index.ts`)

Export everything from a directory through a single `index.ts` so imports stay clean:

```typescript
// components/ui/index.ts
export { Button } from './Button'
export { Input } from './Input'
export { Modal } from './Modal'
export type { ButtonProps, InputProps, ModalProps } from './types'

// Callers import from the directory, not individual files
import { Button, Input, Modal } from '@/components/ui'
```

**Rules:**
- One `index.ts` per directory that has shared exports
- Don't re-export everything blindly — only what external consumers should use
- Avoid barrel files inside deeply nested implementation directories; they slow down tree-shaking

---

## Result Type — Explicit Error Handling Without Exceptions

Throws work well for unexpected errors; for expected domain failures (validation, not-found, auth), a Result type makes the failure path explicit in the type signature:

```typescript
type Result<T, E = Error> =
  | { ok: true;  value: T }
  | { ok: false; error: E }

// Return a Result instead of throwing for domain errors
async function findUser(id: string): Promise<Result<User, 'not_found' | 'db_error'>> {
  try {
    const user = await db.users.findUnique({ where: { id } })
    if (!user) return { ok: false, error: 'not_found' }
    return { ok: true, value: user }
  } catch {
    return { ok: false, error: 'db_error' }
  }
}

// Callers are forced to handle both paths
const result = await findUser(id)
if (!result.ok) {
  if (result.error === 'not_found') return res.status(404).json({ error: 'User not found' })
  return res.status(500).json({ error: 'Internal error' })
}
const { value: user } = result
```

**When to use Result vs throw:**
- `throw` — unexpected errors (network failure, programming errors, invariant violations)
- `Result` — expected domain failures where the caller *must* handle each case

---

## Advanced Generics

### Constrained Generics

```typescript
// Constrain T to objects with an id field
function groupById<T extends { id: string }>(items: T[]): Record<string, T> {
  return Object.fromEntries(items.map(item => [item.id, item]))
}

// Works with any type that has id: string
const marketMap = groupById(markets)   // Record<string, Market>
const userMap   = groupById(users)     // Record<string, User>
```

### Conditional Types

```typescript
// Unwrap a Promise type
type Awaited<T> = T extends Promise<infer U> ? U : T

// Make specific keys required
type RequireFields<T, K extends keyof T> = T & Required<Pick<T, K>>

type MarketWithName = RequireFields<Partial<Market>, 'id' | 'name'>
// id and name are required; all other Market fields are optional
```

### Template Literal Types

```typescript
type EventName = 'market' | 'user' | 'order'
type EventAction = 'created' | 'updated' | 'deleted'
type Topic = `${EventName}.${EventAction}`
// 'market.created' | 'market.updated' | ... — 9 combinations, all checked

function subscribe(topic: Topic, handler: () => void): void { }
subscribe('market.created', handler)  // ✅
subscribe('market.archived', handler) // ❌ type error — 'archived' not in EventAction
```

---

## Utility Types — Quick Reference

```typescript
interface Market {
  id: string
  name: string
  status: 'active' | 'resolved'
  createdAt: Date
  internalNotes: string
}

// All fields optional
type PartialMarket = Partial<Market>

// All fields required
type RequiredMarket = Required<Market>

// Subset of fields
type MarketSummary = Pick<Market, 'id' | 'name' | 'status'>

// All fields except specified
type PublicMarket = Omit<Market, 'internalNotes'>

// Make specific fields readonly
type FrozenMarket = Readonly<Market>

// Union to object map
type StatusRecord = Record<Market['status'], Market[]>
// { active: Market[], resolved: Market[] }

// Extract union members matching a condition
type ActiveStatus = Extract<Market['status'], 'active'>  // 'active'
type InactiveStatus = Exclude<Market['status'], 'active'>  // 'resolved'
```

---

## Type Guards

```typescript
// Custom type guard — return type is a type predicate
function isMarket(value: unknown): value is Market {
  return (
    typeof value === 'object' &&
    value !== null &&
    'id' in value &&
    'name' in value &&
    'status' in value
  )
}

// Use in narrowing contexts
function processItem(item: unknown) {
  if (isMarket(item)) {
    console.log(item.name)  // item is Market here
  }
}
```

---

## Environment Variables — Type-Safe Access

Avoid stringly-typed `process.env.SOMETHING` calls scattered through the codebase:

```typescript
// lib/env.ts — validate at startup, export typed constants
import { z } from 'zod'

const envSchema = z.object({
  ANTHROPIC_API_KEY: z.string().min(1),
  DATABASE_URL: z.string().url(),
  NODE_ENV: z.enum(['development', 'production', 'test']).default('development'),
  PORT: z.coerce.number().default(3000),
})

// Throws at startup if any required variable is missing or malformed
export const env = envSchema.parse(process.env)

// Usage — typed and guaranteed non-empty
import { env } from '@/lib/env'
const client = new Anthropic({ apiKey: env.ANTHROPIC_API_KEY })
```
