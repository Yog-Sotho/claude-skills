The `skill-manager` package is a single `SKILL.md` file (no assets or scripts). Here it is split into its logical sections for GitHub readability:

---

### 📄 `skill-manager/SKILL.md`

**— PART 1: Frontmatter & Overview —**

```yaml
---
name: skill-manager
description: >-
  Manage the user's installed Claude skills via slash commands. Trigger this
  skill whenever the user types /list, /add, /edit, /save, or /delete in the
  context of skills, or says things like "list my skills", "add a new skill",
  "edit the X skill", "delete skill Y", "install a skill from this file",
  "update my skill", or "what skills do I have". Also trigger when the user
  uploads a .skill, .zip, or .md file and asks to install it. This skill is
  the authoritative handler for all skill CRUD operations — always use it when
  the user wants to inspect or modify their skillset.
---
```

```markdown
# Skill Manager

Provides a command interface for reading and managing the user's installed skills.

## Filesystem Layout

| Path | Type | Writable |
|------|------|----------|
| `/mnt/skills/user/` | User-installed skills | ✅ Yes |
| `/mnt/skills/public/` | Anthropic built-in skills | ❌ Read-only |
| `/mnt/skills/examples/` | Example/template skills | ❌ Read-only |

Each skill lives at `<base>/<skill-name>/SKILL.md` (plus optional assets/scripts/references subdirs).

**User-installable skills** go in `/mnt/skills/user/`. This is the only location you write to.

## Active Skills

The currently active (loaded) skills for this session are listed in the `available_skills` block
visible in the system prompt. Use that list as the source of truth for what is "in use" — a skill
is active if its `name` frontmatter field appears there.
```

---

**— PART 2: Command Reference —**

```markdown
## Command Reference

### `/list`

List every skill across all locations, with status flags.

**Algorithm:**
```
1. bash: find /mnt/skills -name "SKILL.md" | sort
2. For each SKILL.md, extract `name:` from YAML frontmatter (head -20, grep)
3. Determine location bucket: user / public / examples
4. Flag as [ACTIVE] if name appears in current session's available_skills
5. Render a clean table grouped by location bucket
```

**Output format:**
```
📦 Your Skills

USER SKILLS (editable)
  ✅ [ACTIVE]  elite-coder         — Activates elite-level coding discipline…
  ✅ [ACTIVE]  warmstart           — Refresh project intelligence…
  ○  [off]     my-old-skill        — …

BUILT-IN SKILLS (read-only)
  ✅ [ACTIVE]  docx                — Create, read, edit Word documents…
  ✅ [ACTIVE]  pdf                 — Anything with PDF files…

EXAMPLE SKILLS (read-only)
  ○  [off]     skill-creator       — Create and improve skills…
```

`[ACTIVE]` = loaded this session. `[off]` = installed but not in the active skill list.

---

### `/add`

Install a new skill into `/mnt/skills/user/`.

Accepted input formats:
- **`.skill` file** — a zip archive produced by `package_skill.py`. Extract with Python `zipfile`,
  install the inner `<skill-name>/` folder to `/mnt/skills/user/`.
- **`.zip` file** — same extraction logic as `.skill`.
- **`.md` file** — treat as a bare `SKILL.md`. Ask the user what to name the skill, then create
  `/mnt/skills/user/<n>/SKILL.md`.
- **Inline paste** — user pastes SKILL.md content into chat. Same as `.md` path.

**Algorithm:**
```
1. Detect the upload. Check /mnt/user-data/uploads/ for the most-recently
   modified file if the user says they uploaded something.
2. Determine format from extension (.skill / .zip / .md / unknown).
3. Extract or copy to /tmp/skill-install-staging/.
4. Validate: SKILL.md must exist and contain name + description frontmatter.
5. If valid, mv /tmp/skill-install-staging/<n>/ /mnt/skills/user/<n>/
6. Confirm installation to user.
```

**Error cases:**
- Duplicate name: warn the user and ask whether to overwrite or rename.
- Missing SKILL.md: reject and explain the expected structure.
- Read-only target (unexpected): report the exact error, suggest workaround.

---

### `/edit --skill <n>`

Open a skill for in-conversation editing.

**Algorithm:**
```
1. Resolve skill path: check /mnt/skills/user/<n>/SKILL.md first (preferred, writable).
   If not found, check public/examples — warn: edits require copying to user/ first.
2. Read SKILL.md content via bash (cat).
3. Display the full content in a fenced markdown code block.
4. Enter EDIT MODE: tell the user they can paste a modified version back, then /save.
```

**Read-only skill handling:** Offer to copy to user/ before editing.

**State tracking:** `EDIT_STATE = { skill_name, original_path, pending_content }`

---

### `/save`

Persist the current pending edit to disk.

**Pre-conditions:** `/edit` must have run this session. Target must be under `/mnt/skills/user/`.

**Algorithm:**
```
1. Re-validate: YAML frontmatter intact, no truncation markers ("…", "# rest unchanged").
2. Write to .tmp file, then rename (atomic).
3. Verify write (stat mtime).
4. Confirm: "✅ Saved <n>/SKILL.md (<N> lines)."
5. Clear EDIT_STATE.
```

---

### `/delete --skill <n>`

Permanently remove a skill from the user's skillset.

**Rules:** Only `/mnt/skills/user/<n>/` is deletable. Always confirm before deleting.

**Algorithm:**
```
1. Confirm /mnt/skills/user/<n>/ exists.
2. Show skill name + description.
3. Ask: "⚠️ Delete '<n>'? This cannot be undone. Type 'yes' to confirm."
4. On confirmation: rm -rf /mnt/skills/user/<n>/
5. Verify deletion, confirm to user.
```

---

### `/help`

Display the full command reference (syntax + description of each command).
```

---

**— PART 3: Implementation Notes & UX Principles —**

```markdown
## Implementation Notes

### Extracting YAML frontmatter

```python
import re

def extract_frontmatter_field(skill_md_path: str, field: str) -> str:
    text = open(skill_md_path).read()
    m = re.search(r'^---\n(.*?)\n---', text, re.DOTALL)
    if not m:
        return ""
    block = m.group(1)
    fm = re.search(rf'^{field}:\s*>?-?\n?(.*?)(?=\n\w|\Z)', block, re.DOTALL | re.MULTILINE)
    if fm:
        return ' '.join(fm.group(1).split())
    return ""
```

### Writing files safely

Always write to a `.tmp` file first, then rename atomically:

```bash
python3 -c "
content = open('/tmp/pending_edit.md').read()
with open('/mnt/skills/user/<n>/SKILL.md.tmp', 'w') as f:
    f.write(content)
import os; os.replace('/mnt/skills/user/<n>/SKILL.md.tmp',
                      '/mnt/skills/user/<n>/SKILL.md')
print('OK')
"
```

### Detecting uploads

```bash
ls -t /mnt/user-data/uploads/ | head -5
```

Then ask the user to confirm which file is the skill they want to install.

---

## UX Principles

- **Confirm before destructive ops.** `/delete` always requires explicit confirmation.
- **Show skill identity before acting.** Show name + description before editing or deleting.
- **Be specific on errors.** Show actual stderr, not just "something went wrong."
- **Graceful degradation.** If `/mnt/skills/user/` is unexpectedly read-only, explain why
  and suggest workarounds (e.g., re-uploading via the Claude.ai skill installer UI)
