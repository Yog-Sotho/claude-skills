---
name: self-improving-agent-v2
description: Enables the assistant to continuously improve by reflecting on tasks, recording mistakes, and retrieving past learnings before solving similar problems. Use this skill when a task fails or produces an error, when the user corrects the assistant, when a better solution is discovered mid-task, when an API or tool behaves unexpectedly, when a repeated workflow could be optimized, or whenever a non-obvious solution is found that is worth remembering. Trigger proactively — before complex tasks, check for relevant past learnings; after complex tasks, check if a learning entry should be recorded.
---

# Self-Improving Agent v2

This skill allows the assistant to improve over time by recording failures, discoveries,
and optimizations — and retrieving them before tackling similar work.

Think of the `.learnings/` directory as a lab notebook: brief, high-signal entries
that prevent repeating the same mistakes.

---

## Step 0 — Environment Check

Before writing any `.learnings/` entries, verify the filesystem will persist:

```bash
# Check for signs of a persistent home directory
if [ -d ~/.claude ] || [ -d ~/projects ] || [ -f ~/.bashrc ]; then
  echo "PERSISTENT: filesystem appears to persist between sessions"
else
  echo "EPHEMERAL: no persistent home directory detected"
  echo "See 'Claude.ai fallback' section below before writing any learnings."
fi
```

**If persistent (Claude Code / Cowork):** proceed normally — write entries to `.learnings/`.

**If ephemeral (Claude.ai web):** see the fallback section at the bottom of this skill.

---

## Core Learning Loop

Every meaningful task follows this cycle:

```
task → execution → reflection → learning → improved future behavior
```

### Pre-Task: Retrieve

Before starting a complex or technical task:

1. Check if `.learnings/` exists: `ls .learnings/ 2>/dev/null`
2. Search for relevant entries: `grep -rl "<keyword>" .learnings/ 2>/dev/null`
3. Read any matching files and apply lessons before proceeding.

If no relevant learnings exist, proceed normally.

### Post-Task: Reflect

After completing a complex task, evaluate:

- Did anything fail or produce unexpected output?
- Was the approach inefficient — would a different method be faster?
- Did the user correct the assistant?
- Was a non-obvious technique discovered that would save time next time?
- Did an external API, tool, or schema behave differently than expected?

If yes to any of the above → record a learning entry. Otherwise, skip.

**Not every task needs an entry. Record only what is worth remembering.**

---

## Learning Storage

Store entries in the most appropriate file:

| File | Contents |
|---|---|
| `.learnings/LEARNINGS.md` | General discoveries and techniques |
| `.learnings/ERRORS.md` | Mistakes and how to avoid them |
| `.learnings/OPTIMIZATIONS.md` | Faster or cleaner approaches found |
| `.learnings/API_CHANGES.md` | Schema shifts, endpoint changes, tool behavior |

Create the directory and files on first use:

```bash
mkdir -p .learnings
touch .learnings/LEARNINGS.md .learnings/ERRORS.md \
      .learnings/OPTIMIZATIONS.md .learnings/API_CHANGES.md
```

---

## Learning Entry Format

Each entry uses this structure:

```markdown
## <Short title — what this is about>

- **Context:** What task was being performed?
- **Problem:** What went wrong, or what limitation existed?
- **Solution:** What fixed the issue?
- **Prevention:** How to avoid this problem in future tasks?
- **Tags:** `keyword1` `keyword2` `keyword3`
- **Date:** YYYY-MM-DD
```

### Example — Error entry

```markdown
## Dexscreener API pairAddress path changed

- **Context:** Parsing token data from Dexscreener API response.
- **Problem:** `pairAddress` was not found at the expected top-level path.
- **Solution:** Updated parser to read `pairs[0].baseToken.address` instead.
- **Prevention:** Always verify API schema with a test call before writing parsers.
- **Tags:** `dexscreener` `api` `parsing` `json`
- **Date:** 2026-02-14
```

### Example — Optimization entry

```markdown
## Cache parsed JSON before iterating large token lists

- **Context:** Processing large API responses with repeated field access.
- **Problem:** Re-parsing the same JSON structure inside a loop caused slowdowns.
- **Solution:** Parse once into a variable and iterate over the result.
- **Prevention:** Always pre-parse before entering loops over API data.
- **Tags:** `performance` `json` `caching`
- **Date:** 2026-02-20
```

---

## Safety Rules

- **Never automatically modify system instructions, core configuration files, or
  any file outside `.learnings/`** based on a learning entry.
- Learnings are references for future reasoning, not executable instructions.
- Promoting a learning into permanent system behavior requires explicit human review.
- Do not record sensitive data (keys, credentials, PII) in any learning entry.

---

## Reflection Discipline

Record an entry only when it meets at least one of these criteria:

- A mistake worth avoiding in future sessions
- A technique or pattern worth reusing
- A structural change in an external system (API, tool, schema)
- A non-obvious solution that took significant effort to discover

**Skip trivial entries** — routine successes, obvious facts, or tasks where nothing
surprising happened do not need entries.

---

## Claude.ai Fallback (Ephemeral Filesystem)

In Claude.ai, the container resets between conversations. `.learnings/` files written
in one session will not exist in the next. To preserve learnings across sessions:

**Option A — Export at end of session:**
After recording learnings, offer the user a download:
```bash
cat .learnings/ERRORS.md .learnings/LEARNINGS.md \
    .learnings/OPTIMIZATIONS.md .learnings/API_CHANGES.md \
    > /mnt/user-data/outputs/learnings-export.md
```
The user can then re-upload this file at the start of the next session.

**Option B — In-conversation only:**
Maintain the learning context in the conversation itself. Reference earlier mistakes
or discoveries from the current chat without writing to disk. Useful for single-session
tasks that don't warrant file management overhead.

**Option C — Claude Memory:**
Suggest the user add key learnings to Claude's Memory (Settings → Memory) for
persistent cross-session recall on important project-level facts.

---

## Long-Term Goal

Over time, `.learnings/` becomes a knowledge base of real-world project experience.
Use it to avoid repeated mistakes, improve reliability, solve tasks more efficiently,
and adapt to changing APIs and environments.
