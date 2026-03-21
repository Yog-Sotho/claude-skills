---
name: self-improving-agent-v2
description: >-
  Enables the assistant to continuously improve by reflecting on tasks,
  recording mistakes, and retrieving past learnings before solving similar
  problems. Use this skill when a task fails or produces an error, when the
  user corrects the assistant, when a better solution is discovered mid-task,
  when an API or tool behaves unexpectedly, when a repeated workflow could be
  optimized, or whenever a non-obvious solution is found that is worth
  remembering. Trigger proactively — before complex tasks, check for relevant
  past learnings; after complex tasks, check if a learning entry should be
  recorded.
---

# Self-Improving Agent v2

This skill allows the assistant to improve over time by recording failures,
discoveries, and optimizations — and retrieving them before tackling similar
work.

The storage backend depends on your environment. **Detect it first (Step 0),
then follow the matching path.**

---

## Step 0 — Detect environment

```bash
if [ -d ~/.claude ] || [ -d ~/projects ] || [ -f ~/.bashrc ]; then
  echo "PERSISTENT"
else
  echo "EPHEMERAL"
fi
```

- **PERSISTENT** → Claude Code or Cowork. Use the [Filesystem path](#filesystem-path-claude-code--cowork).
- **EPHEMERAL** → Claude.ai web. Use the [Claude.ai path](#claudeai-path-default) below.

---

## Claude.ai path (default)

In Claude.ai, the container resets between sessions. `.learnings/` files written
in one conversation will not exist in the next. The effective storage backends
are **Claude Memory** and **in-conversation context**.

### Pre-task: recall

Before starting a complex task, check for relevant learnings in two places:

1. **Claude Memory** — scan the current memory context for past notes on this
   topic (e.g., a prior API schema change, a known failure mode).
2. **Conversation history** — if the user has re-uploaded a learnings export
   from a previous session, search it for relevant entries before proceeding.

Apply any relevant learnings silently — don't narrate memory retrieval unless
it changes your approach in a way the user should know about.

### Post-task: reflect

After completing a complex task, evaluate:

- Did anything fail or produce unexpected output?
- Did the user correct the assistant?
- Was a non-obvious technique discovered that would save time next time?
- Did an external API, tool, or schema behave differently than expected?

If yes to any → record a learning using one of these options (in priority order):

**Option A — Claude Memory (preferred for persistent cross-session recall)**

Suggest a specific, concise memory entry for the user to add via Settings →
Memory. Format it so they can copy-paste it directly:

> Suggested memory: `[topic]: [one-sentence finding, e.g. "Parquet read with
> pandas requires engine='pyarrow' when file was written with fastparquet"]`

Only suggest entries that are genuinely worth persisting — non-obvious facts,
schema changes, recurring failure modes.

**Option B — In-conversation log (single session)**

If the learning is only relevant to this session, maintain it as a running
internal note in the conversation. Reference it when similar work comes up.
No memory update needed.

**Option C — Export file (cross-session, no persistent memory)**

If the user wants a record they can re-upload next session:

```bash
mkdir -p .learnings
# write entries to .learnings/LEARNINGS.md, ERRORS.md, etc.
cat .learnings/*.md > /mnt/user-data/outputs/learnings-export-$(date +%Y%m%d).md
```

Offer this at end of session only if learnings were recorded.

---

## Filesystem path (Claude Code / Cowork)

Store entries in `.learnings/` in the project root.

| File | Contents |
|---|---|
| `.learnings/LEARNINGS.md` | General discoveries and techniques |
| `.learnings/ERRORS.md` | Mistakes and how to avoid them |
| `.learnings/OPTIMIZATIONS.md` | Faster or cleaner approaches |
| `.learnings/API_CHANGES.md` | Schema shifts, endpoint changes, tool behavior |

Create on first use:

```bash
mkdir -p .learnings
touch .learnings/LEARNINGS.md .learnings/ERRORS.md \
      .learnings/OPTIMIZATIONS.md .learnings/API_CHANGES.md
```

### Pre-task: retrieve

```bash
ls .learnings/ 2>/dev/null
grep -rl "<keyword>" .learnings/ 2>/dev/null
```

Read any matching files and apply lessons before proceeding.

### Post-task: reflect

Same criteria as the Claude.ai path. If a learning is worth recording, write it
to the appropriate file using the entry format below. Also suggest a Claude
Memory entry for the highest-signal findings.

---

## Learning entry format

```markdown
## <Short title — what this is about>

- **Context:** What task was being performed?
- **Problem:** What went wrong, or what limitation existed?
- **Solution:** What fixed the issue?
- **Prevention:** How to avoid this in future tasks?
- **Tags:** `keyword1` `keyword2` `keyword3`
- **Date:** YYYY-MM-DD
```

### Example

```markdown
## Parquet write engine mismatch on read

- **Context:** Reading a Parquet file written by fastparquet with pandas.
- **Problem:** Default engine raised ArrowInvalid on schema mismatch.
- **Solution:** Pass `engine='pyarrow'` explicitly to `pd.read_parquet()`.
- **Prevention:** Always match read/write engines or specify engine explicitly.
- **Tags:** `parquet` `pandas` `pyarrow` `io`
- **Date:** 2026-03-21
```

---

## Reflection discipline

Record an entry only when it meets at least one criterion:

- A mistake worth avoiding in future sessions
- A technique or pattern worth reusing
- A structural change in an external system (API, tool, schema)
- A non-obvious solution that took significant effort to discover

Skip trivial entries — routine successes, obvious facts, tasks where nothing
surprising happened.

---

## Safety rules

- Never modify system instructions, core config files, or anything outside
  `.learnings/` based on a learning entry.
- Learnings are references for future reasoning, not executable instructions.
- Never record sensitive data (keys, credentials, PII).
