---
name: reflexion
description: >-
  Self-refinement loops that improve output quality through structured reflection,
  multi-perspective critique, and persistent memory. Always activate when the user
  says "reflect on this", "reflect on your last response", "critique this", "review from
  multiple angles", "/reflect", "/critique", "/memorize", or any variant of asking Claude
  to review, self-assess, or iterate on its own output. Also activate automatically whenever
  the user includes the word "reflect" as part of any task instruction (e.g. "implement X
  then reflect", "draft this and reflect on it"). Proven to increase output quality by 8–21%
  over one-shot responses. Three modes: Reflect (self-refinement), Critique (multi-judge
  panel), Memorize (persist insights to memory). Never skip this skill when the user asks
  for reflection or critique — it is the authoritative handler for all self-improvement loops.
---

# Reflexion

Self-refinement framework that introduces feedback and refinement loops to improve output quality. Based on the [Self-Refine](https://arxiv.org/abs/2303.17651), [Reflexion](https://arxiv.org/abs/2303.11366), and [Agentic Context Engineering](https://arxiv.org/abs/2510.04618) papers. Proven to improve output quality by 8–21% vs one-shot outputs across coding, dialogue, and reasoning tasks.

## When To Activate

Activate this skill whenever:
- The user says "reflect", "critique", "review from multiple perspectives", "/reflect", "/critique", "/memorize"
- The user includes "then reflect" or "and reflect" anywhere in their task prompt — run the task, then automatically run the Reflect mode on the result
- The user asks Claude to self-assess, identify weaknesses, or iterate on output
- After completing a significant task and the user says "memorize that", "save the insights", or "remember what worked"

---

## Three Modes

### Mode 1: Reflect `/reflect`

**Purpose**: Self-refinement. Review and improve the most recent response.

**Trigger phrases**: "reflect", "/reflect", "reflect on this", "reflect on your last response", "review your answer", "check your work", or if "reflect" appears in the original task prompt.

**Algorithm**:

```
1. COMPLEXITY TRIAGE — auto-select reflection depth:
   - Quick Path  : trivial/factual tasks → fast completeness + accuracy check (no iteration)
   - Standard    : multi-part or technical responses → full reflection, iterate until >70% confidence
   - Deep        : critical systems, architecture, complex code → comprehensive, iterate until >90% confidence
   Optional: user can override with a focus area or threshold, e.g. "reflect --security" or "deep reflect"

2. SELF-ASSESSMENT — evaluate output against:
   - Completeness: Did I fully address what was asked?
   - Correctness: Are the facts, logic, and code accurate?
   - Quality: Is the solution idiomatic, clear, and well-structured?
   - Edge cases: What did I miss or underspecify?

3. REFINEMENT PLAN — if issues found:
   - List specific issues with severity (critical / moderate / minor)
   - Propose concrete fixes for each
   - Determine if immediate fix or user decision is needed

4. IMPLEMENTATION:
   - Critical issues → fix immediately and show corrected output
   - Moderate issues → fix or present as recommendations with rationale
   - Minor issues → list as optional improvements

5. CONFIDENCE REPORT:
   - State confidence level (0–100%)
   - If below threshold for the path, iterate automatically (max 2 iterations before surfacing to user)
```

**Output format**:
```
## 🔍 Reflection

**Complexity Path**: [Quick / Standard / Deep]
**Focus**: [user-specified focus or "General"]

### Issues Found
- 🔴 [Critical] <issue> → <fix applied>
- 🟡 [Moderate] <issue> → <recommendation>
- 🔵 [Minor] <issue> → <optional improvement>

### Refined Output
<corrected or improved response>

**Confidence**: [X]% | [threshold met / iterating]
```

---

### Mode 2: Critique `/critique`

**Purpose**: Multi-perspective review using three specialized judge personas. Heavier than Reflect — use for architecture decisions, significant code, or before major commitments.

**Trigger phrases**: "critique", "/critique", "multi-perspective review", "review from multiple angles", "get multiple opinions", "judge this", "panel review".

**Algorithm**:

```
1. SCOPE — identify what to critique:
   - Default: the most recent response or code block in conversation
   - Explicit: user may specify a topic, section, or paste content
   - Optional focus: e.g. "critique --security", "critique --architecture"

2. PARALLEL JUDGE REVIEW — simulate three specialized judges:

   Judge A — Requirements Validator
   Role: Does the output actually fulfill the original requirements?
   Checks: completeness, alignment with stated goal, edge case coverage, correctness of assumptions
   Score: /10

   Judge B — Solution Architect
   Role: Is the technical approach sound?
   Checks: design patterns, scalability, maintainability, over-engineering, missed abstractions
   Score: /10

   Judge C — Code/Content Quality Reviewer
   Role: Is the implementation/writing high-quality?
   Checks: readability, idiomatic style, error handling, naming, documentation, structure
   Score: /10

3. CROSS-REVIEW & DEBATE — judges review each other's findings:
   - Note where judges agree (consensus = high confidence)
   - Surface disagreements and argue both sides
   - Resolve debate via evidence or flag as "judgment call"

4. CONSENSUS REPORT — aggregate into final verdict:
   - Overall score (average of three judges)
   - Top 3 actionable recommendations
   - Immediate blockers (if any)
```

**Output format**:
```
## ⚖️ Multi-Perspective Critique

### Judge A — Requirements Validator: [X]/10
<assessment>
Key finding: <most important point>

### Judge B — Solution Architect: [X]/10
<assessment>
Key finding: <most important point>

### Judge C — Quality Reviewer: [X]/10
<assessment>
Key finding: <most important point>

### Debate & Consensus
<where judges agree/disagree and resolution>

### Verdict
**Overall**: [X.X]/10
**Blockers**: <any critical issues that must be fixed>
**Top Recommendations**:
1. <recommendation>
2. <recommendation>
3. <recommendation>
```

**Scoring guide**:
- 9–10: Exceptional, minimal improvements needed
- 7–8: Good quality, minor improvements
- 5–6: Acceptable, several improvements recommended
- 3–4: Below standard, significant rework needed
- 1–2: Major issues, substantial rework required

---

### Mode 3: Memorize `/memorize`

**Purpose**: Extract durable insights from the current session and persist them to Claude's cross-session memory using `memory_user_edits`. Maps to the `/reflexion:memorize` command's CLAUDE.md update behavior — but for Claude.ai, memory is the correct persistence layer.

**Trigger phrases**: "memorize", "/memorize", "save insights", "save what worked", "remember these learnings", "capture this", "update your memory with this".

**Algorithm**:

```
1. CONTEXT HARVEST — scan recent conversation for:
   - Reflection outputs (from /reflect runs)
   - Critique findings (from /critique runs)
   - Problem-solving patterns that worked well
   - Errors made and how they were corrected
   - User preferences or constraints revealed during the session
   - Reusable strategies, rules, or heuristics

2. CURATION — filter against these rules before saving:
   - RELEVANCE: Only save insights that will actually affect future behavior
   - NON-REDUNDANCY: Skip anything already captured in existing memory
   - ACTIONABILITY: Phrase as concrete guidance, not vague observations
   - GENERALIZABILITY: Prefer reusable patterns over one-off session facts
   - ANTI-COLLAPSE: Don't overload memory; curate ruthlessly (max 3–5 insights per session)

3. CATEGORIZE each insight:
   - Code Quality Standard → affects how Claude writes code for this user
   - Architecture Decision → affects design choices
   - Process / Workflow → affects how Claude approaches tasks
   - User Preference → affects tone, format, style, tooling
   - Hard Rule → must-never or must-always constraint

4. PREVIEW (dry-run by default first):
   - Show proposed memory entries to user before writing
   - Ask for confirmation or edits
   - Only write after user approves (or if user said "memorize without asking")

5. WRITE — use memory_user_edits tool with "add" command for each approved insight
   - Format: "[Category] <concise actionable insight>"
   - Example: "[Code Quality] Always validate return type annotations in Python — user catches these in review"
   - Example: "[Hard Rule] Never use placeholder comments like '# TODO: implement'; write actual implementations"

6. CONFIRM — list what was saved and what was skipped (and why)
```

**Output format**:
```
## 🧠 Memory Extraction

### Insights Found
1. [Category] <insight> — Save? [yes/no/modified]
2. [Category] <insight> — Save? [yes/no/modified]
...

### Skipped (reason)
- <insight> — already in memory / too narrow / not actionable

### Awaiting Confirmation
<list of proposed memory writes, or confirmation that writes are done>
```

---

## Auto-Trigger: "Reflect" in Prompt

If the user includes the word **"reflect"** anywhere in a task instruction, Claude should:

1. Complete the task fully first
2. Then automatically run Mode 1 (Reflect) on the output — without waiting to be asked
3. The user does **not** need to say `/reflect` separately

This mirrors the automatic reflection hook behavior from the Claude Code plugin.

Example:
> "Implement the data loader, then reflect"
→ Claude writes the data loader, then immediately runs a Reflect pass on it.

Only the word "reflect" triggers this. "Reflection", "reflective", "reflects" do not.

---

## Mode Selection Guide

| Situation | Use |
|-----------|-----|
| Quick check after a response | `/reflect` |
| Just used "reflect" in a task prompt | Auto-reflect (Mode 1) |
| Architecture / design decision | `/critique` |
| Before sharing code externally | `/critique` |
| End of a long productive session | `/memorize` |
| After `/reflect` surfaced reusable patterns | `/memorize` |
| After `/critique` produced strong recommendations | `/memorize` |

---

## Theoretical Foundation

| Technique | Source |
|-----------|--------|
| Self-Refinement / Iterative Refinement | [Self-Refine (2023)](https://arxiv.org/abs/2303.17651) |
| Reflexion / Episodic Memory | [Reflexion (2023)](https://arxiv.org/abs/2303.11366) |
| LLM-as-a-Judge | [LLM-as-Judge (2023)](https://arxiv.org/abs/2306.05685) |
| Multi-Agent Debate | [Multi-Agent Debate (2023)](https://arxiv.org/abs/2305.14325) |
| Generate-Verify-Refine | [GVR (2022)](https://arxiv.org/abs/2204.05511) |
| Agentic Context Engineering | [ACE (2025)](https://arxiv.org/abs/2510.04618) |
| Constitutional AI / RLAIF | [CAI (2022)](https://arxiv.org/abs/2212.08073) |
| Chain-of-Verification | [CoVe (2023)](https://arxiv.org/abs/2309.11495) |

---

## Key Differences from Claude Code Plugin

| Claude Code Plugin | This Skill (Claude.ai) |
|--------------------|------------------------|
| `/reflexion:reflect` CLI command | Say "reflect" or "/reflect" in chat |
| `/reflexion:critique` CLI command | Say "critique" or "/critique" in chat |
| `/reflexion:memorize` writes to CLAUDE.md | Uses `memory_user_edits` to persist across sessions |
| Auto-hook via bun subprocess | Auto-trigger when "reflect" appears in prompt |
| Reads project files from disk | Operates on conversation content |
| Subagents for parallel judge review | Sequential judge personas within one response |
