---
name: search-first
description: >-
  Enforce a research-before-coding workflow — search for existing tools,
  libraries, and patterns before writing custom code. Use this skill whenever
  the user asks to add functionality, integrate a new capability, create a
  utility or helper, or add a dependency. Trigger on phrases like "add X
  functionality", "integrate X", "I need a library for", "write a helper to",
  "implement X from scratch", or any time you're about to write non-trivial
  code that might already exist as a package or tool. When in doubt, search
  first — reinventing a solved problem is never the right call.
---

# /search-first — Research Before You Code

Systematizes the "search for existing solutions before implementing" workflow.
Always run this before writing non-trivial code that might already be solved.

---

## Workflow

```
1. NEED ANALYSIS      → Define what's needed, language/framework constraints
2. SEARCH             → Repo → packages → MCP/skills → web
3. EVALUATE           → Score candidates against decision matrix
4. DECIDE             → Adopt / Extend / Compose / Build inline / Skip
5. IMPLEMENT          → Minimal code, no reinvention
```

---

## Step 1 — Need Analysis

Before searching, state clearly:
- What capability is needed (one sentence)
- Language and framework in use
- Any hard constraints (license, no new deps, size budget, etc.)

---

## Step 2 — Search (in this order)

### 2a. Check the repo first
Does this already exist in the codebase?
- Search for relevant function/class names in existing modules
- Check `requirements.txt` / `pyproject.toml` / `package.json` — the dep may already be installed

### 2b. Search package registries
Use `web_search` to find candidates:
- Python: `web_search("pypi <capability> python")` or `"<capability> python library site:pypi.org"`
- Node: `web_search("npm <capability> javascript")`
- Prioritize packages with: recent commits, high download counts, active issue tracker

### 2c. Check available MCP servers and skills
- Scan the `available_skills` list in the current context — a skill may already cover this
- Check connected MCP servers (visible in Settings → Integrations) — an integration may provide the capability natively

### 2d. Web search for reference implementations
`web_search("<capability> best library <language> <year>")` — look for recent comparisons, awesome lists, or authoritative recommendations.

---

## Step 3 — Evaluate candidates

Score each candidate on:

| Criterion | What to check |
|---|---|
| Functionality | Does it cover the full need, or just part? |
| Maintenance | Last commit < 12 months? Open issues responded to? |
| Community | Download count, GitHub stars, Stack Overflow presence |
| Docs | README quality, examples, API reference |
| License | MIT/Apache = safe; GPL = check; no license = avoid |
| Dependencies | Does it pull in a large transitive tree for a small feature? |

---

## Step 4 — Decision matrix

| Signal | Action |
|---|---|
| Exact match, well-maintained, permissive license | **Adopt** — install and use directly |
| Partial match, good foundation | **Extend** — install + write thin wrapper |
| Multiple weak matches | **Compose** — combine 2–3 small packages |
| Nothing suitable, but need is real | **Build** — write custom, informed by research |
| Trivial to implement (< ~20 lines), any package is overkill | **Build inline** — no dependency needed |
| Cost of dependency exceeds benefit | **Skip** — reconsider whether the feature is necessary |

---

## Step 5 — Implement

- **Adopt/Extend/Compose**: install the package, write the minimal glue code, document why it was chosen
- **Build**: apply elite-coder delivery standards — complete, typed, error-handled
- **Build inline**: keep it small; if it grows beyond ~30 lines, reconsider whether a package is warranted

---

## Search shortcuts by category

> Verify currency before recommending — these reflect common choices as of early 2026.

**Python / ML**
- HTTP clients → `httpx` (async-native, retries built-in)
- Validation → `pydantic` (v2 preferred)
- Data processing → `polars` (fast), `pandas` (ecosystem)
- Document parsing → `pdfplumber`, `unstructured`, `mammoth`
- CLI → `typer`, `click`

**JavaScript / TypeScript**
- HTTP → native `fetch` + `ky` for retries/hooks
- Validation → `zod`
- Testing → `vitest` (modern), `jest`
- Markdown → `remark`, `unified`

**Dev tooling**
- Linting (Python) → `ruff` (replaces pylint + flake8)
- Linting (JS) → `eslint`
- Formatting → `black` / `prettier`
- Pre-commit → `pre-commit`

---

## Examples

### Example 1: "Add dead link checking"
```
Need:       Check markdown files for broken links
Search:     web_search("markdown dead link checker npm")
Found:      textlint-rule-no-dead-link (score: 9/10, active, MIT)
Decision:   ADOPT
Result:     npm install + config — zero custom code
```

### Example 2: "Add resilient HTTP client"
```
Need:       HTTP client with retries and timeout handling (Python)
Search:     web_search("python http client retries 2025")
Found:      httpx has built-in retry support via transport layer
Decision:   ADOPT
Result:     httpx already in pyproject.toml — no new dep needed
```

### Example 3: "Validate config files against a schema"
```
Need:       Validate project JSON configs against a schema (Node)
Search:     web_search("json schema validator cli npm")
Found:      ajv-cli (score: 8/10) — validates but needs a schema file
Decision:   EXTEND — install ajv-cli, write project-specific schema
Result:     1 package + 1 schema file, no custom validation logic
```

### Example 4: "Format a duration as human-readable string"
```
Need:       Convert seconds to "2h 15m 30s" string (Python)
Search:     pypi search → nothing worth the dependency
Decision:   BUILD INLINE — 12-line utility function, no package needed
Result:     Custom function in utils.py, zero new dependencies
```

---

## Anti-patterns

- **Jumping to code**: writing a utility without checking if one exists
- **Ignoring existing deps**: not checking if the package is already installed
- **Stale shortcuts**: recommending a library without verifying it's still maintained
- **Over-customizing**: wrapping a library so heavily it loses its benefits
- **Dependency bloat**: pulling in a large package for a feature coverable in 15 lines
