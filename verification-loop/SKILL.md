---
name: verification-loop
description: Run a structured pre-PR verification sweep across build, types, lint, tests, security, and diff review. Always activate this skill when the user says things like "verify my changes", "run checks", "is this ready for PR?", "check my code", "run the verification loop", or after completing a feature, refactor, or significant change. Also activate proactively before any PR creation — even if not explicitly asked. Works for TypeScript, JavaScript, and Python projects.
---

# Verification Loop

A structured, phased quality gate for code changes — designed to catch issues before they reach a PR or production.

## Workflow

When this skill activates:

1. **Detect the project type** from context (package.json → JS/TS, pyproject.toml / setup.py → Python, or ask if ambiguous).
2. **Run phases in order.** Each phase gates the next — if Phase 1 (build) fails, skip phases 2–4 since their output is unreliable on a broken build. Always run Phase 5 (security) and Phase 6 (diff) regardless.
3. **Collect results as you go** — don't wait until the end to report failures.
4. **Produce the verification report** after all phases complete.
5. **If issues are found**, list them prioritized by severity and offer to fix them.

---

## Phase 1: Build

A broken build means nothing else is trustworthy. Fix this before anything else.

```bash
# JavaScript / TypeScript
npm run build 2>&1 | tail -30
# or
pnpm build 2>&1 | tail -30

# Python (check package installs cleanly)
pip install -e . 2>&1 | tail -20
# or if using build tools
python -m build 2>&1 | tail -20
```

**Gate**: If build fails → skip Phases 2, 3, 4. Report the failure immediately. Still run Phases 5 and 6.

---

## Phase 2: Type Check

Type errors are silent runtime bugs. Treat errors as blockers; warnings as informational.

```bash
# TypeScript
npx tsc --noEmit 2>&1 | head -50

# Python
pyright . 2>&1 | head -50
# or
mypy . 2>&1 | head -50
```

Report the count of errors vs. warnings. Fix all errors before marking types as PASS.

---

## Phase 3: Lint

Lint failures indicate code style violations or real logic issues (unused vars, unreachable code, etc.).

```bash
# JavaScript / TypeScript
npm run lint 2>&1 | head -50
# or directly
npx eslint src/ 2>&1 | head -50

# Python
ruff check . 2>&1 | head -50
```

Distinguish between errors (must fix) and warnings (should fix). Auto-fixable issues can be resolved with:

```bash
npx eslint src/ --fix    # JS/TS
ruff check . --fix       # Python
```

---

## Phase 4: Test Suite

Tests verify behavior hasn't regressed. Coverage below 80% is a flag worth surfacing to the user.

```bash
# JavaScript / TypeScript
npm run test -- --coverage 2>&1 | tail -60
# or
npx vitest run --coverage 2>&1 | tail -60

# Python
pytest --cov=. --cov-report=term-missing 2>&1 | tail -60
```

Report:
- Total tests, passed, failed, skipped
- Overall coverage %
- Any newly failing tests (compare against known baseline if available)

If tests fail, identify whether it's a pre-existing failure or something introduced by the current change.

---

## Phase 5: Security Scan

Always run this, even if the build failed. Secrets in code are an immediate risk regardless of build state.

Prefer a dedicated tool if available:

```bash
# TruffleHog (best coverage, detects 700+ secret types)
trufflehog filesystem . 2>&1 | head -30

# detect-secrets (good for CI enforcement)
detect-secrets scan 2>&1 | head -30
```

If no dedicated tool is installed, run a targeted grep as a fallback — but note it's a spot-check, not a full scan:

```bash
# Common secret patterns (not exhaustive)
grep -rn --include="*.ts" --include="*.tsx" --include="*.js" --include="*.py" \
  -E "(sk-|api_key|API_KEY|secret_key|SECRET_KEY|private_key|PRIVATE_KEY|password\s*=|bearer\s)" \
  . 2>/dev/null | grep -v "node_modules" | grep -v ".git" | head -20

# Check for debug artifacts
grep -rn "console\.log\|debugger\|breakpoint()" --include="*.ts" --include="*.tsx" src/ 2>/dev/null | head -10
```

Flag any hits for manual review — false positives are expected with grep; use judgment.

---

## Phase 6: Diff Review

Review what actually changed before declaring the work done. Surprises here are common after a long session.

```bash
# Summary of changed files
git diff --stat HEAD

# Names only (for scoping)
git diff --name-only HEAD

# Full diff if needed for a specific file
git diff HEAD -- path/to/file
```

For each changed file, check:
- **Unintended changes** — files touched that shouldn't have been
- **Missing error handling** — new code paths without try/catch or error returns
- **Edge cases** — null inputs, empty arrays, concurrent access, boundary values
- **TODOs / FIXMEs** left in — intentional or forgotten?
- **Commented-out code** — should it be deleted or is it temporary?

---

## Verification Report

After all phases, always produce this report:

```
VERIFICATION REPORT
===================
Project:   [name and type — JS/TS/Python]
Scope:     [feature / refactor / bugfix — inferred from diff]

Phase 1 — Build:     [PASS / FAIL]
Phase 2 — Types:     [PASS / FAIL] (X errors, Y warnings)
Phase 3 — Lint:      [PASS / FAIL] (X errors, Y warnings)
Phase 4 — Tests:     [PASS / FAIL] (X/Y passed, Z% coverage)
Phase 5 — Security:  [PASS / WARN / FAIL] (X issues found)
Phase 6 — Diff:      [CLEAN / REVIEW NEEDED] (X files changed)

Overall:   [✅ READY / ⚠️ READY WITH NOTES / ❌ NOT READY] for PR

Issues (prioritized):
1. [BLOCKER] ...
2. [SHOULD FIX] ...
3. [OPTIONAL] ...
```

Use `BLOCKER` for anything that must be fixed before merging, `SHOULD FIX` for things that will cause problems soon, and `OPTIONAL` for style or polish items.

---

## Continuous Mode

For long sessions, run the verification loop at natural breakpoints rather than on a timer — the right cadence is task-driven, not clock-driven:

- After completing a discrete unit of work (function, component, endpoint)
- Before switching context to a different part of the codebase
- When something "feels off" after a complex change
- Always before opening a PR

To invoke, just say "run verification", "check my changes", or "are we good?".
