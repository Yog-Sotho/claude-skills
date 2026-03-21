---
name: codebase-onboarding
description: Analyze an unfamiliar codebase and generate a structured onboarding guide with architecture map, key entry points, conventions, and a starter CLAUDE.md. Always activate when the user says "onboard me", "help me understand this codebase", "walk me through this repo", "generate a CLAUDE.md", "update the CLAUDE.md", "I just joined this project", or when opening a repo with Claude Code for the first time. Also activate when the user seems lost in an unfamiliar codebase — even without an explicit request.
---

# Codebase Onboarding

Systematically analyze an unfamiliar codebase and produce a structured onboarding guide. Designed for developers joining a new project or setting up Claude Code in an existing repo for the first time.

## Workflow

1. **Determine the scope** from user context — full onboarding (guide + CLAUDE.md), CLAUDE.md only, or update existing CLAUDE.md.
2. **Run Phase 1 (Reconnaissance)** using Glob and Grep — not Read. Read files selectively only to resolve ambiguous signals.
3. **Run Phase 2 (Architecture Mapping)** from the reconnaissance data.
4. **Run Phase 3 (Convention Detection)** — identify patterns already in use.
5. **Detect monorepo structure** if applicable (see Monorepo Handling below).
6. **Produce Phase 4 outputs** per the scope:
   - **Full onboarding**: print the Onboarding Guide to the conversation, then write `CLAUDE.md` to the project root.
   - **CLAUDE.md only**: write `CLAUDE.md` to the project root, no guide.
   - **Update existing CLAUDE.md**: read it first, merge new findings, mark additions clearly.
7. **Flag conflicts and unknowns** — if reconnaissance finds contradictions or cannot determine something, say so explicitly rather than guessing.

---

## Phase 1: Reconnaissance

Gather raw signals about the project. Run all checks in parallel — do not Read every file.

```
1. Package manifest detection
   → package.json, go.mod, Cargo.toml, pyproject.toml, pom.xml, build.gradle,
     Gemfile, composer.json, mix.exs, pubspec.yaml

2. Framework fingerprinting
   → next.config.*, nuxt.config.*, angular.json, vite.config.*,
     django settings, flask app factory, fastapi main, rails config

3. Entry point identification
   → main.*, index.*, app.*, server.*, cmd/, src/main/

4. Directory structure snapshot
   → Top 2 levels of the directory tree, ignoring: node_modules, vendor,
     .git, dist, build, __pycache__, .next, .turbo, .cache

5. Config and tooling detection
   → .eslintrc*, .prettierrc*, tsconfig.json, Makefile, Dockerfile,
     docker-compose*, .github/workflows/, .env.example,
     turbo.json, nx.json, pnpm-workspace.yaml

6. Test structure detection
   → tests/, test/, __tests__/, *_test.go, *.spec.ts, *.test.js,
     pytest.ini, jest.config.*, vitest.config.*

7. Monorepo signals
   → turbo.json, nx.json, pnpm-workspace.yaml, lerna.json,
     packages/ or apps/ at root, workspace field in package.json
```

**Conflicting signals:** if reconnaissance finds contradictions (e.g., package.json lists React but vite.config suggests Vue, or two test runners are configured), read the relevant files to resolve it. If still ambiguous, report both and note which appears dominant.

---

## Phase 2: Architecture Mapping

From the reconnaissance data, identify:

**Tech Stack**
- Language(s) and version constraints
- Framework(s) and major libraries
- Database(s) and ORMs
- Build tools and bundlers
- CI/CD platform

**Architecture Pattern**
- Monolith, monorepo, microservices, or serverless
- Frontend/backend split or full-stack
- API style: REST, GraphQL, gRPC, tRPC

**Key Directories**
Map top-level directories to their purpose — skip self-explanatory names like src/:

```
src/components/  → React UI components
src/api/         → API route handlers
src/lib/         → Shared utilities
src/db/          → Database models and migrations
tests/           → Test suites
scripts/         → Build and deployment scripts
```

**Data Flow**
Trace one representative request from entry to response:
- Where does a request enter? (router, handler, controller)
- How is it validated? (middleware, schemas, guards)
- Where is business logic? (services, models, use cases)
- How does it reach the database? (ORM, raw queries, repositories)

---

## Phase 3: Convention Detection

**Naming Conventions**
- File naming: kebab-case, camelCase, PascalCase, snake_case
- Component/class naming patterns
- Test file naming: *.test.ts, *.spec.ts, *_test.go

**Code Patterns**
- Error handling style: try/catch, Result types, error codes
- Dependency injection or direct imports
- State management approach
- Async patterns: callbacks, promises, async/await, channels

**Git Conventions**
- Branch naming from recent branches
- Commit message style from recent commits
- PR workflow (squash, merge, rebase)
- If history is unavailable or shallow (e.g., git clone --depth 1): skip and note "Git history too shallow to detect conventions"

**Footguns and Gotchas**
Actively look for project-specific traps new developers commonly hit:
- Non-standard package manager requirements (e.g., "must use pnpm, not npm")
- Environment setup quirks (.env files that must be created manually, secrets required locally)
- Commands that look safe but are not (e.g., a make clean that also drops the database)
- Implicit service dependencies (e.g., "Redis must be running before the dev server starts")
- Known flaky tests or test setup requiring special flags

---

## Monorepo Handling

If monorepo signals are detected in Phase 1, adapt the analysis:

- **Map the workspace layout**: identify apps/, packages/, or equivalent and what each contains
- **Distinguish root vs. package config**: root-level package.json, turbo.json, tsconfig.json apply globally; per-package configs override locally
- **Identify the build tool**: Turborepo, Nx, pnpm workspaces, Lerna — each has different run conventions
- **Note the correct dev command**: running from root vs. from a specific package directory

```bash
# Turborepo
turbo dev                      # all apps
turbo dev --filter=web         # specific app

# pnpm workspaces
pnpm --filter web dev

# Nx
nx serve web
```

- **Flag cross-package dependencies**: if packages import from each other, note the internal package names so new devs do not try to npm install them as external packages

---

## Phase 4: Output Templates

### Output 1: Onboarding Guide

Print to the conversation. Target: scannable in under 2 minutes.

```markdown
# Onboarding Guide: [Project Name]

## Overview
[2-3 sentences: what this project does, who it serves, and its current state]

## Tech Stack
| Layer | Technology | Version |
|-------|-----------|---------|
| Language | [detected] | [version] |
| Framework | [detected] | [version] |
| Database | [detected] | [version] |
| Testing | [detected] | - |

## Architecture
[Diagram or prose: how components connect]
[For monorepos: include workspace layout map]

## Key Entry Points
- **[Entry point]**: [path] — [what it does]
- **[Config source of truth]**: [path] — [what it controls]

## Directory Map
[Top-level directory to purpose; omit self-explanatory names]

## Request Lifecycle
[Trace one API request from entry to response — 4-6 steps]

## Common Commands
- **Dev server**: [command]
- **Tests**: [command]
- **Lint**: [command]
- **Build**: [command]

## Conventions
- [File naming pattern]
- [Error handling approach]
- [Test file naming]
- [Git commit style if detectable]

## Where to Look
| I want to... | Look at... |
|--------------|-----------|
| [common task] | [path] |
| [common task] | [path] |

## Gotchas & Known Issues
- [Non-obvious setup requirement or footgun]
- [Any conflicting signals that could not be fully resolved]
- Note "Could not determine [X]" for anything genuinely unclear — never guess
```

### Output 2: CLAUDE.md

Write to the project root as CLAUDE.md. If one already exists: read it first, preserve all existing project-specific instructions, merge new findings, and mark additions with a comment.

Target length: under 100 lines. Focused beats exhaustive.

```markdown
# [Project Name] — Claude Instructions

## Tech Stack
[One-line summary of key technologies]

## Commands
- Dev: [command]
- Test: [command]
- Lint: [command]
- Build: [command]

## Project Structure
[Key directory to purpose — only non-obvious directories]

## Code Conventions
- [File naming — e.g., "Components: PascalCase; utilities: kebab-case"]
- [Error handling — e.g., "Use Result<T> types, not throw"]
- [Import style — e.g., "Absolute imports via @/ alias; no relative imports above 2 levels"]

## Testing
- Run: [command]
- Pattern: [test file convention — e.g., co-located *.test.ts]
- Coverage: [command if configured]

## Do / Don't
- YES: [Something to always do — e.g., "Use pnpm, not npm or yarn"]
- YES: [Project-specific pattern to follow]
- NO: [Common mistake — e.g., "Don't run migrations directly; use make migrate"]
- NO: [Footgun found during analysis]

## Git
- Commits: [style if detectable — e.g., "Conventional commits: feat/fix/chore"]
- Branches: [naming pattern if detectable]
```

---

## Best Practices

- **Don't Read everything** — reconnaissance uses Glob and Grep; Read selectively only for ambiguous signals
- **Verify, don't guess** — if a framework is detected from config but the actual code differs, trust the code
- **Respect existing CLAUDE.md** — read it first, enhance rather than replace, mark additions clearly
- **Stay concise** — the onboarding guide is for scanning, not archiving
- **Flag unknowns explicitly** — "Could not determine test runner" is better than a wrong answer
- **Prioritize footguns** — new devs lose the most time to non-obvious setup requirements; surface these prominently in both outputs

## Anti-Patterns to Avoid

- CLAUDE.md over 100 lines — keep it focused
- Listing every dependency — highlight only those that shape how code is written
- Explaining self-explanatory directory names
- Copying the README — the onboarding guide adds structural insight the README does not have
- Guessing at conventions that cannot be confirmed from the code or config

---

## Scope Examples

**Full onboarding** — "Onboard me" / "Help me understand this codebase" / "I just joined this project"
→ All 4 phases → Onboarding Guide printed to conversation → CLAUDE.md written to project root

**CLAUDE.md only** — "Generate a CLAUDE.md for this project"
→ Phases 1-3 → write CLAUDE.md only

**Update CLAUDE.md** — "Update the CLAUDE.md" / "Refresh the Claude instructions"
→ Read existing CLAUDE.md → Phases 1-3 → merge findings → mark new additions
