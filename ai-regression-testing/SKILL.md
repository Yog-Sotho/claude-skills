---
name: ai-regression-testing
description: Regression testing strategies specifically for AI-assisted development, where the same model writes and reviews code — creating systematic blind spots that only automated tests can catch. Always activate when an AI agent (Claude Code, Cursor, Codex) has modified backend logic or API routes, a bug was found and fixed, sandbox/production path parity needs verification, a bug-check or review command is being run, or the user asks how to prevent AI-introduced regressions. Also activate when the user reports a bug was "fixed" but keeps coming back.
---

# AI Regression Testing

Testing patterns specifically designed for AI-assisted development, where the same model writes code and reviews it — creating systematic blind spots that only automated tests can catch.

## Workflow

When this skill activates:

1. **Identify the triggering event** — new bug found, AI just made changes, setting up a bug-check workflow, or configuring CI.
2. **For a newly found bug**: write the regression test *before* applying the fix. A failing test that proves the bug exists is more valuable than a passing test that proves the fix worked.
3. **For a bug-check run**: enforce the mandatory sequence — automated tests → build check → AI code review → regression test for each finding.
4. **Adapt the patterns to the user's stack** — the examples below use TypeScript/Vitest/Next.js, but the patterns apply to any language. See the Python equivalent at the bottom of this skill.
5. **Propose CI integration** after any new test is written — a regression test that only runs locally will be bypassed.

---

## The Core Problem

When an AI writes code and then reviews its own work, it carries the same assumptions into both steps. This creates a predictable failure pattern:

```
AI writes fix → AI reviews fix → AI says "looks correct" → Bug still exists
```

**Real-world example** (observed in production — same bug introduced 4 times):

```
Fix 1: Added notification_settings to API response
  → Forgot to add it to the SELECT query
  → AI reviewed and missed it (same blind spot)

Fix 2: Added it to SELECT query
  → TypeScript build error (column not in generated types)
  → AI reviewed Fix 1 but didn't catch the SELECT issue

Fix 3: Changed to SELECT *
  → Fixed production path, forgot sandbox path
  → AI reviewed and missed it AGAIN

Fix 4: Regression test caught it instantly on first run ✅
```

The pattern: **sandbox/production path inconsistency** is the #1 AI-introduced regression.

---

## Sandbox-Mode API Testing

Most projects with AI-friendly architecture have a sandbox/mock mode. This enables fast, DB-free regression tests.

### Setup (Vitest + Next.js App Router)

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import path from "path";

export default defineConfig({
  test: {
    environment: "node",
    globals: true,
    include: ["__tests__/**/*.test.ts"],
    setupFiles: ["__tests__/setup.ts"],
  },
  resolve: { alias: { "@": path.resolve(__dirname, ".") } },
});
```

```typescript
// __tests__/setup.ts — force sandbox mode, no database needed
process.env.SANDBOX_MODE = "true";
process.env.NEXT_PUBLIC_SUPABASE_URL = "";
process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY = "";
```

### Test Helpers

```typescript
// __tests__/helpers.ts
import { NextRequest } from "next/server";

export function createTestRequest(
  url: string,
  options?: {
    method?: string;
    body?: Record<string, unknown>;
    headers?: Record<string, string>;
    sandboxUserId?: string;
  },
): NextRequest {
  const { method = "GET", body, headers = {}, sandboxUserId } = options ?? {};
  const fullUrl = url.startsWith("http") ? url : `http://localhost:3000${url}`;
  const reqHeaders: Record<string, string> = { ...headers };

  if (sandboxUserId) reqHeaders["x-sandbox-user-id"] = sandboxUserId;

  const init: { method: string; headers: Record<string, string>; body?: string } = {
    method,
    headers: reqHeaders,
  };

  if (body) {
    init.body = JSON.stringify(body);
    reqHeaders["content-type"] = "application/json";
  }

  return new NextRequest(fullUrl, init);
}

export async function parseResponse(response: Response) {
  const json = await response.json();
  return { status: response.status, json };
}
```

### Writing Regression Tests

**Write the test before the fix.** A failing test that proves the bug exists is the most valuable artifact of a debugging session.

```typescript
// __tests__/api/user/profile.test.ts
import { describe, it, expect } from "vitest";
import { createTestRequest, parseResponse } from "../../helpers";
import { GET } from "@/app/api/user/profile/route";

// API contract — every field the frontend depends on
const REQUIRED_FIELDS = [
  "id", "email", "full_name", "phone", "role",
  "created_at", "avatar_url",
  "notification_settings",  // ← added after BUG-R1 was found missing
];

describe("GET /api/user/profile", () => {
  it("returns all required fields", async () => {
    const req = createTestRequest("/api/user/profile");
    const { status, json } = await parseResponse(await GET(req));

    expect(status).toBe(200);
    for (const field of REQUIRED_FIELDS) {
      expect(json.data).toHaveProperty(field);
    }
  });

  // Named after the bug — makes git blame useful
  it("notification_settings is present and not undefined (BUG-R1 regression)", async () => {
    const req = createTestRequest("/api/user/profile");
    const { json } = await parseResponse(await GET(req));

    expect("notification_settings" in json.data).toBe(true);
    const ns = json.data.notification_settings;
    expect(ns === null || typeof ns === "object").toBe(true);
  });
});
```

---

## Common AI Regression Patterns

### Pattern 1: Sandbox/Production Path Mismatch

**Frequency**: Most common — 3 of 4 observed regressions.

```typescript
// ❌ AI adds field to production path only
function isSandboxMode(): boolean {
  return process.env.SANDBOX_MODE === "true";
}

if (isSandboxMode()) {
  return { data: { id, email, name } };                          // missing new field
}
return { data: { id, email, name, notification_settings } };    // production has it

// ✅ Both paths return the same shape — null is fine, missing is not
if (isSandboxMode()) {
  return { data: { id, email, name, notification_settings: null } };
}
return { data: { id, email, name, notification_settings } };
```

**Test to catch it** — assert both paths against the same contract:

```typescript
it("sandbox mode returns the same fields as production contract", async () => {
  // SANDBOX_MODE=true is forced by setup.ts — this always runs sandbox path
  const { json } = await parseResponse(await GET(createTestRequest("/api/user/messages")));

  expect(Array.isArray(json.data)).toBe(true);
  // ⚠ Never gate assertions on data length — empty response = vacuous pass
  // Always seed sandbox data or assert the empty case explicitly:
  expect(json.data.length).toBeGreaterThan(0);   // fail fast if sandbox has no seed data
  for (const item of json.data) {
    expect(item).toHaveProperty("partner_name");
  }
});
```

### Pattern 2: SELECT Clause Omission

**Frequency**: Common with Supabase/Prisma when adding new columns.

```typescript
// ❌ Column in response but not in SELECT — always undefined
const { data } = await supabase
  .from("users")
  .select("id, email, name")   // notification_settings missing
  .single();

return { data: { ...data, notification_settings: data.notification_settings } };
// → notification_settings is silently undefined

// ✅ Explicit select or SELECT *
const { data } = await supabase
  .from("users")
  .select("*")
  .single();
```

**Test to catch it**: the required-fields assertion in Pattern 1 catches this automatically.

### Pattern 3: Error State Leakage

**Frequency**: Moderate — when adding error handling to components with existing data state.

```typescript
// ❌ Error set but stale data not cleared
catch (err) {
  setError("Failed to load");
  // previous tab's reservations still displayed alongside error message
}

// ✅ Clear related state on error
catch (err) {
  setReservations([]);
  setError("Failed to load");
}
```

### Pattern 4: Optimistic Update Without Rollback

```typescript
// ❌ UI updated optimistically, no rollback on API failure
const handleRemove = async (id: string) => {
  setItems(prev => prev.filter(i => i.id !== id));
  await fetch(`/api/items/${id}`, { method: "DELETE" });
  // API failure → item gone from UI, still in DB
};

// ✅ Capture snapshot, rollback on failure
const handleRemove = async (id: string) => {
  const previous = [...items];
  setItems(prev => prev.filter(i => i.id !== id));
  try {
    const res = await fetch(`/api/items/${id}`, { method: "DELETE" });
    if (!res.ok) throw new Error(`${res.status}`);
  } catch {
    setItems(previous);
    setError("Delete failed — please try again");
  }
};
```

### Pattern 5: Type Cast Masking Null

**Frequency**: Subtle — AI adds type assertions that hide undefined/null at compile time.

```typescript
// ❌ Non-null assertion hides that the field is actually missing
const name = user.profile!.name;   // TypeScript sees string, runtime sees undefined

// ❌ Optional chaining silently returns undefined — frontend treats as "no value"
const name = user.profile?.name;   // downstream code assumes string, gets undefined

// ✅ Validate presence explicitly before use
if (!user.profile?.name) {
  throw new Error("profile.name missing from API response");
}
const name = user.profile.name;
```

**Test to catch it** — assert value type, not just presence:

```typescript
it("user.name is a non-empty string, not null or undefined (BUG-R5)", async () => {
  const { json } = await parseResponse(await GET(createTestRequest("/api/user/profile")));
  expect(typeof json.data.full_name).toBe("string");
  expect(json.data.full_name.length).toBeGreaterThan(0);
});
```

---

## Bug-Check Workflow

### Custom Command

```markdown
<!-- .claude/commands/bug-check.md -->
# Bug Check

## Step 1: Automated Tests (mandatory — cannot skip)

    npm run test    # Vitest regression suite
    npm run build   # TypeScript type check

- FAIL → report as highest-priority bug, stop
- PASS → continue to Step 2

## Step 2: AI Code Review (with known blind spots)

1. Sandbox/production path consistency
2. API response shape matches frontend contract
3. SELECT clause completeness
4. Error handling with state cleanup
5. Optimistic update rollback
6. Non-null assertion usage

## Step 3: For each bug found, write a regression test before fixing it
```

### Workflow Sequence

```
User runs /bug-check
  │
  ├─ Step 1: npm run test
  │   ├─ FAIL → mechanical bug found, report & stop
  │   └─ PASS → continue
  │
  ├─ Step 2: npm run build
  │   ├─ FAIL → type error found, report & stop
  │   └─ PASS → continue
  │
  ├─ Step 3: AI review with blind-spot checklist
  │
  └─ Step 4: For each finding → write regression test → then fix
```

---

## CI Integration

A regression test that only runs locally will be bypassed. Lock it into CI immediately after writing it.

```yaml
# .github/workflows/regression.yml
name: Regression Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npm run test          # regression suite — blocks merge on failure
      - run: npm run build         # type check — blocks merge on type errors
```

For AI-agent workflows (Claude Code, Cursor), configure the agent to run `npm run test` before committing. A test that fails post-commit is worth far less than one that blocked the commit.

---

## Python / Other Stacks

The patterns above are TypeScript-specific, but the principles apply universally. Python equivalents:

```python
# pytest equivalent of the sandbox setup
# conftest.py
import os
import pytest

@pytest.fixture(autouse=True, scope="session")
def sandbox_mode():
    os.environ["SANDBOX_MODE"] = "true"
    os.environ["DATABASE_URL"] = ""

# Equivalent required-fields regression test
REQUIRED_FIELDS = ["id", "email", "full_name", "notification_settings"]

def test_profile_returns_all_required_fields(client):
    response = client.get("/api/user/profile")
    assert response.status_code == 200
    data = response.json()["data"]
    for field in REQUIRED_FIELDS:
        assert field in data, f"Missing required field: {field}"

# Pattern 5 equivalent — assert type, not just presence
def test_full_name_is_non_empty_string(client):
    data = client.get("/api/user/profile").json()["data"]
    assert isinstance(data["full_name"], str), "full_name must be a string"
    assert len(data["full_name"]) > 0, "full_name must not be empty (BUG-R5)"
```

---

## Strategy: Test Where Bugs Were Found

Don't aim for coverage percentage. Aim for regression prevention:

```
Bug found in /api/user/profile     → Write test for profile API
Bug found in /api/user/messages    → Write test for messages API
No bug ever in /api/user/settings  → Don't write test (yet)
```

**Why this works for AI development:**
1. AI makes the **same category of mistake** repeatedly across the codebase
2. Bugs cluster in complex areas (auth, multi-path logic, state management)
3. Once a regression test exists, that exact bug **cannot silently recur**
4. Test count grows organically with bug fixes — no wasted effort

---

## Quick Reference

| Pattern | Test Strategy | Priority |
|---|---|---|
| Sandbox/production mismatch | Assert same response shape in sandbox mode | 🔴 High |
| SELECT clause omission | Assert all required fields present and non-undefined | 🔴 High |
| Error state leakage | Assert state cleared on error response | 🟡 Medium |
| Missing rollback | Assert UI state restored on API failure | 🟡 Medium |
| Type cast masking null | Assert field type, not just presence | 🟡 Medium |

---

## DO / DON'T

**DO:**
- Write the regression test **before** fixing the bug — a failing test proves the bug exists
- Name tests after the bug they prevent: `"BUG-R1 regression"` survives in git blame forever
- Assert sandbox data length > 0 before iterating — empty response is a vacuous pass
- Run tests as the **first** step of every bug-check, before any code review
- Lock regression tests into CI immediately — a local-only test will be skipped
- Keep the suite fast (< 2s total via sandbox mode)

**DON'T:**
- Trust AI self-review as a substitute for automated tests — the blind spots are structural
- Gate field assertions on `if (data.length > 0)` — seed the sandbox or assert the empty case
- Write tests for code that has never produced a bug
- Skip sandbox path coverage because "it's just mock data" — that's where bugs hide
- Aim for coverage percentage — aim for regression prevention
