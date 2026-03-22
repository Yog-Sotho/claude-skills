# Reflexion — Worked Examples & Patterns

Reference patterns for the three reflexion modes in Claude.ai context.

---

## Reflect: Complexity Triage Examples

### Quick Path (trivial task)
User: "What's the capital of France?" → Answer: "Paris"
Reflect: "✅ Quick check: Correct and complete. Confidence: 99%"

### Standard Path (technical response)
User: "Write a Python function to debounce API calls"
→ Claude writes function
→ Reflect checks: signature correctness, edge cases (rapid calls, thread safety), idiomatic use, docstring
→ Confidence >70% threshold

### Deep Path (critical system)
User: "Design the auth middleware for our FastAPI app"
→ Claude produces design
→ Reflect: full security audit, JWT handling, session invalidation, rate limiting gaps, >90% confidence required

---

## Critique: Judge Persona Templates

### Judge A — Requirements Validator (internal monologue)
```
"The user asked for X. Does my output deliver X fully?
- Did I address all sub-requirements?
- Are my assumptions explicit and correct?
- Would the user consider this done?"
```

### Judge B — Solution Architect (internal monologue)
```
"Is this the right approach?
- Is there a simpler solution I overlooked?
- Does this scale to realistic load?
- Am I introducing unnecessary coupling?
- What would break in 6 months?"
```

### Judge C — Quality Reviewer (internal monologue)
```
"Is this production-quality?
- Would this pass a code review at a senior level?
- Is error handling complete?
- Is naming self-documenting?
- Are there any magic numbers or unclear constants?"
```

---

## Memorize: Curation Rules in Practice

### SAVE this insight
"User consistently catches missing return type annotations in Python → always include them"
Why: Actionable, reusable, affects future code.

### SKIP this insight
"User was working on a database sanitizer today"
Why: Not actionable, too session-specific, already likely in memory.

### SAVE this insight
"User's style: no placeholder TODOs — write the full implementation even for simple stubs"
Why: Hard rule, directly affects output quality, repeatedly relevant.

### SKIP this insight
"The fix worked by checking for None before calling .split()"
Why: Too specific to one bug — not generalizable.

---

## Chaining Modes

### Standard workflow
1. Complete task
2. `/reflect` → fix critical issues
3. `/critique` → get panel review if output is significant
4. `/memorize` → save reusable patterns before ending session

### Quick workflow (everyday use)
1. Complete task, include "then reflect" in prompt
2. Auto-reflect runs
3. Done — memorize only if genuinely reusable insight emerged

### Pre-commit equivalent (for code)
1. Share code
2. `/critique --focus=security` or `/critique --focus=architecture`
3. Address blockers
4. `/memorize` if new hard rules emerged
