---
name: coding-typescript
description: TypeScript, JavaScript, React, and Node.js coding standards — naming conventions, type safety, error handling, immutability, React patterns, performance, and code quality. Always activate when writing, reviewing, or refactoring TypeScript or JavaScript code; setting up linting or type-checking; enforcing naming or structural conventions; or when the user asks how to structure a component, handle an error, type a value, or write a hook. Also activate proactively when spotting `any`, unsafe mutations, swallowed errors, or missing cleanup in user code.
---

# TypeScript & React Coding Standards

Consistent, type-safe, maintainable patterns for TypeScript, React, and Node.js.

## Workflow

When this skill activates:

1. **Identify the task** — new code, code review, refactor, or standards setup.
2. **Apply the relevant section** — navigate directly; don't repeat unrelated rules.
3. **Flag violations proactively** if spotted in user code — `any`, unsafe mutations, swallowed errors, missing `key` props, and unguarded `await response.json()` are the most common.
4. **For extended patterns** (Result types, barrel exports, advanced generics), see `references/advanced.md`.

---

## Core Principles

1. **Readability first** — code is read far more than written; optimise for the reader
2. **KISS** — simplest solution that works; no premature optimisation
3. **DRY** — extract common logic; avoid copy-paste programming
4. **YAGNI** — don't build features before they're needed; add complexity only when required
5. **Errors are information** — never swallow them; always preserve the original cause

---

## Naming

```typescript
// ✅ Variables — descriptive nouns
const marketSearchQuery = 'election'
const isUserAuthenticated = true
const totalRevenue = 1000

// ❌ Too short to be meaningful
const q = 'election'
const flag = true
const x = 1000

// ✅ Functions — verb-noun pattern
async function fetchMarketData(marketId: string): Promise<Market> { }
function calculateSimilarity(a: number[], b: number[]): number { }
function isValidEmail(email: string): boolean { }

// ❌ Noun-only or no types
async function market(id: string) { }
function similarity(a, b) { }
```

### File Naming

```
components/Button.tsx          # PascalCase for components
hooks/useAuth.ts               # camelCase with 'use' prefix
lib/formatDate.ts              # camelCase for utilities
types/market.types.ts          # .types suffix
```

---

## Type Safety

### Never Use `any` — Use `unknown` with Narrowing

```typescript
// ❌ any disables the type checker entirely
function process(data: any) {
  return data.name.toUpperCase()  // no safety; runtime crash if name is missing
}

// ✅ unknown forces you to prove the type before using it
function process(data: unknown): string {
  if (typeof data !== 'object' || data === null || !('name' in data)) {
    throw new TypeError('Invalid data shape')
  }
  const { name } = data as { name: unknown }
  if (typeof name !== 'string') throw new TypeError('name must be a string')
  return name.toUpperCase()
}
```

### Define Interfaces — Never Infer from `any`

```typescript
// ✅ Explicit interface with literal union for status
interface Market {
  id: string
  name: string
  status: 'active' | 'resolved' | 'closed'
  createdAt: Date
}

function getMarket(id: string): Promise<Market> { }

// ❌ any defeats TypeScript's purpose
function getMarket(id: any): Promise<any> { }
```

### `satisfies` — Validate Without Widening (TS 4.9+)

```typescript
// ✅ satisfies checks the type but preserves literal narrowing
const config = {
  env: 'production',
  timeout: 5000,
} satisfies Record<string, string | number>

config.env  // type is 'production', not string — literals preserved

// ❌ type assertion widens to the declared type
const config: Record<string, string | number> = { env: 'production', timeout: 5000 }
config.env  // type is string — literal lost
```

---

## Immutability

Mutate only with intent and a comment explaining why. The default is immutable.

```typescript
// ✅ Spread for objects and arrays
const updatedUser = { ...user, name: 'New Name' }
const updatedItems = [...items, newItem]
const withoutFirst = items.slice(1)

// ✅ Sort safely — Array.sort() mutates in place, so copy first
const sortedMarkets = [...markets].sort((a, b) => b.volume - a.volume)

// ❌ Direct mutation — breaks referential equality, confuses React re-renders
user.name = 'New Name'
items.push(newItem)
markets.sort((a, b) => b.volume - a.volume)  // mutates original array!
```

When mutation is genuinely justified (tight loop, large array, proven bottleneck), add a comment:

```typescript
// Deliberately mutating in-place — profiled 40% faster on 100K+ item arrays
items.push(newItem)
```

---

## Error Handling

### Preserve the Cause Chain

```typescript
// ✅ Wrap with cause — callers can inspect the original error
async function fetchData(url: string): Promise<unknown> {
  let response: Response
  try {
    response = await fetch(url)
  } catch (cause) {
    throw new Error(`Network request failed: ${url}`, { cause })
  }

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`)
  }

  try {
    return await response.json()
  } catch (cause) {
    throw new Error('Response body is not valid JSON', { cause })
  }
}

// ❌ Swallows the original error — callers get no diagnostic info
async function fetchData(url: string) {
  try {
    const response = await fetch(url)
    return response.json()
  } catch (error) {
    throw new Error('Failed to fetch data')  // original cause gone
  }
}
```

### Narrow Caught Errors — They Are `unknown` in Strict Mode

```typescript
try {
  await riskyOperation()
} catch (error) {
  // error is `unknown` — narrow before accessing properties
  const message = error instanceof Error ? error.message : String(error)
  logger.error('Operation failed', { message, cause: error })
  throw error  // rethrow if caller needs to handle it
}
```

---

## Async Patterns

```typescript
// ✅ Parallel — run independent async tasks together
const [users, markets, stats] = await Promise.all([
  fetchUsers(),
  fetchMarkets(),
  fetchStats(),
])

// ❌ Sequential when order doesn't matter — 3x slower
const users   = await fetchUsers()
const markets = await fetchMarkets()
const stats   = await fetchStats()

// ✅ Handle partial failures in parallel work
const results = await Promise.allSettled([fetchUsers(), fetchMarkets()])
for (const result of results) {
  if (result.status === 'rejected') logger.warn('Partial fetch failed', { reason: result.reason })
}
```

---

## React Patterns

### Component Structure

```typescript
interface ButtonProps {
  children: React.ReactNode
  onClick: () => void
  disabled?: boolean
  variant?: 'primary' | 'secondary'
}

export function Button({
  children,
  onClick,
  disabled = false,
  variant = 'primary',
}: ButtonProps) {
  return (
    <button onClick={onClick} disabled={disabled} className={`btn btn-${variant}`}>
      {children}
    </button>
  )
}
```

### List Rendering — Always Use `key`

```typescript
// ✅ Stable, unique key from data — not array index
{markets.map((market) => (
  <MarketCard key={market.id} market={market} />
))}

// ❌ Index as key — breaks when list is reordered or filtered
{markets.map((market, i) => (
  <MarketCard key={i} market={market} />
))}
```

### `useEffect` — Always Return Cleanup

```typescript
// ✅ Cleanup prevents memory leaks and stale updates
useEffect(() => {
  const controller = new AbortController()

  async function loadData() {
    try {
      const data = await fetchMarket(id, { signal: controller.signal })
      setMarket(data)
    } catch (error) {
      if (error instanceof Error && error.name === 'AbortError') return
      setError(error instanceof Error ? error.message : 'Unknown error')
    }
  }

  loadData()
  return () => controller.abort()  // cancel in-flight request on unmount
}, [id])

// ❌ No cleanup — setState called on unmounted component; memory leak
useEffect(() => {
  fetchMarket(id).then(setMarket)
}, [id])
```

### State — Use Functional Updates

```typescript
// ✅ Functional update — safe in async and concurrent scenarios
setCount(prev => prev + 1)
setItems(prev => [...prev, newItem])

// ❌ Stale closure — may read outdated value in async handlers
setCount(count + 1)
```

### Memoization — Measure Before Adding

```typescript
// ✅ useMemo for expensive derivations — copy before sort (sort mutates!)
const sortedMarkets = useMemo(
  () => [...markets].sort((a, b) => b.volume - a.volume),
  [markets],
)

// ✅ useCallback for stable function references passed to children
const handleSearch = useCallback((query: string) => {
  setSearchQuery(query)
}, [])

// ⚠ Don't memoize cheap operations — the memo itself has overhead
const doubled = useMemo(() => count * 2, [count])  // not worth it
```

### Custom Hooks

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(handler)  // clear on value/delay change
  }, [value, delay])

  return debouncedValue
}
```

### Conditional Rendering

```typescript
// ✅ Short-circuit — reads top-to-bottom cleanly
{isLoading && <Spinner />}
{error && <ErrorMessage error={error} />}
{data && <DataDisplay data={data} />}

// ❌ Nested ternary — unreadable past 2 branches
{isLoading ? <Spinner /> : error ? <ErrorMessage error={error} /> : data ? <DataDisplay data={data} /> : null}
```

### Lazy Loading

```typescript
const HeavyChart = lazy(() => import('./HeavyChart'))

export function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyChart />
    </Suspense>
  )
}
```

---

## Input Validation (API Routes)

Always guard `request.json()` — malformed bodies throw before Zod validation:

```typescript
import { z } from 'zod'
import { NextRequest, NextResponse } from 'next/server'

const CreateMarketSchema = z.object({
  name: z.string().min(1).max(200),
  description: z.string().min(1).max(2000),
  endDate: z.string().datetime(),
  categories: z.array(z.string()).min(1),
})

export async function POST(request: NextRequest) {
  // Step 1: guard JSON parse — malformed body throws here, not in safeParse
  let body: unknown
  try {
    body = await request.json()
  } catch {
    return NextResponse.json(
      { error: { code: 'invalid_json', message: 'Request body is not valid JSON' } },
      { status: 400 },
    )
  }

  // Step 2: validate schema — safeParse returns a result, never throws
  const result = CreateMarketSchema.safeParse(body)
  if (!result.success) {
    return NextResponse.json(
      { error: { code: 'validation_error', message: 'Validation failed', details: result.error.issues } },
      { status: 422 },
    )
  }

  // Step 3: proceed with type-safe validated data
  const market = await createMarket(result.data)
  return NextResponse.json({ data: market }, { status: 201 })
}
```

---

## Code Smells

### Long Functions

```typescript
// ❌ > 50 lines, doing multiple things
function processMarketData() { /* 100 lines */ }

// ✅ Composed from focused functions
function processMarketData() {
  const validated = validateData(raw)
  const transformed = transformData(validated)
  return saveData(transformed)
}
```

### Deep Nesting — Use Guard Clauses

```typescript
// ❌ 5+ levels of nesting — hard to follow
if (user) {
  if (user.isAdmin) {
    if (market?.isActive) {
      if (hasPermission) { /* do something */ }
    }
  }
}

// ✅ Early returns flatten the happy path
if (!user) return
if (!user.isAdmin) return
if (!market?.isActive) return
if (!hasPermission) return
// do something
```

### Magic Numbers

```typescript
// ❌ What do 3 and 500 mean?
if (retryCount > 3) { }
setTimeout(callback, 500)

// ✅ Named constants are self-documenting
const MAX_RETRIES = 3
const DEBOUNCE_DELAY_MS = 500

if (retryCount > MAX_RETRIES) { }
setTimeout(callback, DEBOUNCE_DELAY_MS)
```

---

## Comments

```typescript
// ✅ Explain WHY — the code already says WHAT
// Exponential backoff caps at 30s to avoid thundering herd on recovery
const delay = Math.min(1000 * Math.pow(2, retryCount), 30_000)

// ❌ Restating the code in English
// Increment counter by 1
count++
```

### JSDoc for Public Functions

```typescript
/**
 * Searches markets using semantic similarity.
 *
 * @param query - Natural language search query
 * @param limit - Maximum results to return (default: 10)
 * @returns Markets sorted by similarity score, highest first
 * @throws {Error} If the embedding service is unavailable
 *
 * @example
 * const results = await searchMarkets('election', 5)
 * console.log(results[0].name)  // "US Presidential Election 2026"
 */
export async function searchMarkets(query: string, limit = 10): Promise<Market[]> { }
```

---

## Testing

```typescript
// ✅ AAA pattern — Arrange / Act / Assert
test('calculates cosine similarity correctly for orthogonal vectors', () => {
  // Arrange
  const vector1 = [1, 0, 0]
  const vector2 = [0, 1, 0]

  // Act
  const similarity = calculateCosineSimilarity(vector1, vector2)

  // Assert
  expect(similarity).toBe(0)
})

// ✅ Descriptive test names — read as a spec
test('returns empty array when no markets match query', () => { })
test('throws when OpenAI API key is missing', () => { })
test('falls back to substring search when cache is unavailable', () => { })

// ❌ Vague
test('works', () => { })
test('test search', () => { })
```

---

## Performance

```typescript
// ✅ Select only needed columns — don't fetch what you won't use
const { data } = await supabase
  .from('markets')
  .select('id, name, status')
  .limit(10)

// ❌ SELECT * fetches everything including large text columns
const { data } = await supabase.from('markets').select('*')
```

---

## Project Structure (Next.js App Router)

```
src/
├── app/
│   ├── api/               # API route handlers
│   ├── markets/           # Market pages
│   └── (auth)/            # Auth pages (route group — no URL segment)
├── components/
│   ├── ui/                # Generic, reusable UI primitives
│   ├── forms/             # Form components
│   └── layouts/           # Layout wrappers
├── hooks/                 # Custom React hooks (use*.ts)
├── lib/
│   ├── api/               # API clients and fetchers
│   ├── utils/             # Pure utility functions
│   └── constants/         # App-wide constants
├── types/                 # Shared TypeScript interfaces
└── styles/                # Global CSS
```

For barrel exports, Result types, and advanced generics, see `references/advanced.md`.
