---
name: warmstart
description: Refresh project intelligence and re-orient on the current codebase. Use this skill whenever the user says things like "catch me up", "what was I working on?", "re-orient yourself", "summarize the project state", "what's in this repo?", "what changed?", "project status", "context refresh", "/warm", or after they mention switching branches, pulling changes, cloning a repo, or starting a new session on an existing project. Trigger even when the request is implicit — if the user dives straight into a task without context and the project state is unknown, run this proactively.
allowed-tools: Bash
---

# Warm Start

Refresh project intelligence by running the warm-start script if available, with a
graceful fallback to native git/filesystem tools if it is not.

## Step 1 — Check for warm-start script

```bash
if [ -f ~/.claude/scripts/warm-start.sh ]; then
  ~/.claude/scripts/warm-start.sh <<< '{"source":"manual","cwd":"'"$(pwd)"'"}'
else
  echo "WARM-START SCRIPT NOT FOUND"
  echo "Expected location: ~/.claude/scripts/warm-start.sh"
  echo "Falling back to native project scan..."
fi
```

If the script runs successfully, read its output carefully — it contains git state,
recent changes, stack information, and learnings from previous sessions. Use this
to re-orient on what the user is working on, then proceed with the task.

## Step 2 — Fallback (if script is missing or errors)

Run these commands to build a manual context brief:

```bash
# Git state
git -C "$(pwd)" log --oneline -10 2>/dev/null || echo "(not a git repo)"
git -C "$(pwd)" status --short 2>/dev/null
git -C "$(pwd)" branch --show-current 2>/dev/null

# Project structure
ls -1 "$(pwd)"

# Key config/manifest files (first 40 lines each)
for f in README.md pyproject.toml setup.py package.json requirements.txt Makefile; do
  [ -f "$(pwd)/$f" ] && echo "=== $f ===" && head -40 "$(pwd)/$f"
done
```

Synthesize the output into a brief mental model: what is this project, what is the
current branch, what was recently changed, and what appears to be in-progress.

## Step 3 — Focus on arguments (if provided)

If the user passed arguments (e.g., `/warm auth-module` or `/warm recent bugs`),
focus your context summary on that specific area rather than giving a general overview.

Arguments received: $ARGUMENTS

**Examples:**
- `/warm` → full project re-orientation
- `/warm auth` → focus on authentication-related files/recent changes
- `/warm last PR` → summarize the most recent commit range / diff

## Installing the warm-start script

If the script is missing and the user wants to set it up, the expected location is:

```
~/.claude/scripts/warm-start.sh
```

This is a user-maintained shell script that emits a JSON blob or structured text
summarizing project state. It can be as simple as a `git log` wrapper or as rich
as a full project intelligence report. Claude can help the user write one on request.
