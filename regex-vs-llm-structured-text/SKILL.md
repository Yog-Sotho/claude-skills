---
name: regex-vs-llm-structured-text
description: Decision framework and implementation patterns for choosing between regex and LLM when parsing structured text. Always activate when the user is parsing repeating structured formats (quizzes, forms, invoices, tables, documents), deciding how to extract data from text, building a text processing pipeline, or asking about cost/accuracy tradeoffs in text extraction. Also activate when the user is about to send all their text to an LLM — they almost certainly don't need to.
---

# Regex vs LLM for Structured Text Parsing

A practical decision framework for parsing structured text. The key insight: regex handles 95–98% of cases cheaply and deterministically. Reserve LLM calls for the remaining edge cases only.

## Workflow

When this skill activates:

1. **Apply the decision framework** to determine the right approach for the user's format.
2. **Adapt the implementation patterns** to the user's specific structure — the examples below use quiz/Q&A format, but the architecture applies to any repeating pattern (forms, invoices, log lines, tables).
3. **Implement confidence scoring** to programmatically route edge cases to LLM — don't eyeball it.
4. **Wire the hybrid pipeline** only if LLM validation is actually needed. Many datasets won't need it.

---

## Decision Framework

```
Is the text format consistent and repeating?
├── Yes (>90% follows a pattern) → Start with Regex
│   ├── Regex handles 95%+ → Done, no LLM needed
│   └── Regex handles <95% → Add LLM for edge cases only
└── No (free-form, highly variable) → Use LLM directly
```

When in doubt, write the regex first. Even an imperfect pattern gives you a baseline and real data on what the edge cases actually are.

---

## Architecture

```
Source Text
    │
    ▼
[Regex Parser] ─── Extracts structure (95–98% of items)
    │
    ▼
[Cleaner] ──────── Strips noise (markers, page numbers, artifacts)
    │
    ▼
[Confidence Scorer] ─── Flags low-confidence extractions
    │
    ├── High confidence (≥0.95) → Direct output
    │
    └── Low confidence (<0.95) → [LLM Validator] → Output
```

---

## Implementation

The examples below parse quiz/Q&A format. Adapt the regex pattern and field names to your structure — the surrounding pipeline (confidence scoring, LLM fallback, hybrid orchestration) is reusable as-is.

### 1. Regex Parser

```python
import re
import json
import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)

@dataclass(frozen=True)
class ParsedItem:
    id: str
    text: str
    choices: tuple[str, ...]
    answer: str
    confidence: float = 1.0

def parse_structured_text(content: str) -> list[ParsedItem]:
    """Parse structured text using regex. Adapt the pattern to your format."""
    # Example: numbered Q&A with A-D choices and an Answer line.
    # Replace this pattern for forms, invoices, log lines, etc.
    pattern = re.compile(
        r"(?P<id>\d+)\.\s*(?P<text>.+?)\n"
        r"(?P<choices>(?:[A-D]\..+?\n)+)"
        r"Answer:\s*(?P<answer>[A-D])",
        re.MULTILINE | re.DOTALL,
    )
    items = []
    for match in pattern.finditer(content):
        try:
            choices = tuple(
                c.strip() for c in re.findall(r"[A-D]\.\s*(.+)", match.group("choices"))
            )
            items.append(ParsedItem(
                id=match.group("id"),
                text=match.group("text").strip(),
                choices=choices,
                answer=match.group("answer"),
            ))
        except Exception as exc:
            logger.warning("Skipping malformed item near offset %d: %s", match.start(), exc)
    return items
```

### 2. Confidence Scoring

Confidence scoring is what makes the hybrid approach work. Without it, you're guessing which items need LLM help. Tune the signals and penalties to your domain.

```python
@dataclass(frozen=True)
class ConfidenceFlag:
    item_id: str
    score: float
    reasons: tuple[str, ...]

def score_confidence(item: ParsedItem) -> ConfidenceFlag:
    """Score extraction confidence. Add domain-specific signals as needed."""
    reasons: list[str] = []
    score = 1.0

    # Structural signals
    if len(item.choices) < 3:
        reasons.append("few_choices")
        score -= 0.3

    if not item.answer:
        reasons.append("missing_answer")
        score -= 0.5

    # Content quality signals
    if len(item.text) < 10:
        reasons.append("short_text")
        score -= 0.2

    if len(item.text) > 1000:
        reasons.append("suspiciously_long_text")  # Possible merge of multiple items
        score -= 0.2

    # Encoding / artifact signals
    if any(ord(c) > 0xFFFF for c in item.text):
        reasons.append("unusual_unicode")
        score -= 0.15

    if re.search(r"[\x00-\x08\x0b\x0c\x0e-\x1f]", item.text):
        reasons.append("control_characters")
        score -= 0.25

    # Truncation signal — text ends mid-sentence
    if item.text and item.text[-1] not in ".?!\"'":
        reasons.append("possible_truncation")
        score -= 0.1

    return ConfidenceFlag(
        item_id=item.id,
        score=max(0.0, score),
        reasons=tuple(reasons),
    )

def identify_low_confidence(
    items: list[ParsedItem],
    threshold: float = 0.95,
) -> list[ConfidenceFlag]:
    """Return flags for all items below the confidence threshold."""
    flags = [score_confidence(item) for item in items]
    return [f for f in flags if f.score < threshold]
```

### 3. LLM Validator (Edge Cases Only)

Only called for items that failed confidence scoring. Use the cheapest capable model — Haiku-class is sufficient for structured extraction.

```python
def validate_with_llm(
    item: ParsedItem,
    original_text: str,
    client,
) -> ParsedItem:
    """Use LLM to fix low-confidence extractions. Returns corrected item or original."""
    prompt = (
        "You are a structured text extractor. "
        "Given the original text and a potentially incorrect extraction, "
        "return a corrected JSON object with keys: id, text, choices (list), answer. "
        "If the extraction is already correct, return the JSON unchanged. "
        "Return ONLY valid JSON — no explanation, no markdown fences.\n\n"
        f"Original text:\n{original_text}\n\n"
        f"Current extraction:\n{json.dumps({'id': item.id, 'text': item.text, 'choices': list(item.choices), 'answer': item.answer})}"
    )

    try:
        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=500,
            messages=[{"role": "user", "content": prompt}],
        )
        raw = response.content[0].text.strip()
        # Strip accidental markdown fences
        raw = re.sub(r"^```(?:json)?\s*|\s*```$", "", raw, flags=re.MULTILINE).strip()
        data = json.loads(raw)
        return ParsedItem(
            id=str(data["id"]),
            text=data["text"],
            choices=tuple(data["choices"]),
            answer=data["answer"],
            confidence=0.85,  # LLM-validated items carry slightly lower confidence
        )
    except (json.JSONDecodeError, KeyError, IndexError) as exc:
        logger.warning("LLM validation failed for item %s: %s — keeping original", item.id, exc)
        return item  # Fall back to original rather than crashing
```

### 4. Hybrid Pipeline

```python
def process_document(
    content: str,
    *,
    llm_client=None,
    confidence_threshold: float = 0.95,
) -> list[ParsedItem]:
    """Full pipeline: regex → confidence check → LLM for edge cases only."""
    # Step 1: Regex extraction (handles 95–98% of items)
    items = parse_structured_text(content)
    if not items:
        logger.warning("Regex extracted zero items — check your pattern against the input format")
        return []

    # Step 2: Confidence scoring
    low_confidence = identify_low_confidence(items, confidence_threshold)

    logger.info(
        "Parsed %d items: %d high-confidence, %d flagged for review",
        len(items), len(items) - len(low_confidence), len(low_confidence),
    )

    if not low_confidence or llm_client is None:
        return items

    # Step 3: LLM validation (only for flagged items)
    low_conf_ids = {f.item_id for f in low_confidence}
    return [
        validate_with_llm(item, content, llm_client) if item.id in low_conf_ids else item
        for item in items
    ]
```

---

## Real-World Metrics

From a production quiz parsing pipeline (410 items):

| Metric | Value |
|--------|-------|
| Regex success rate | 98.0% |
| Low-confidence items | 8 (2.0%) |
| LLM calls needed | ~5 |
| Cost savings vs. all-LLM | ~95% |
| Test coverage | 93% |

---

## Best Practices

- **Start with regex** — even an imperfect pattern gives you a baseline and reveals what edge cases actually exist
- **Use confidence scoring** to route edge cases programmatically, not by eyeballing output
- **Use the cheapest LLM** for validation — Haiku-class models handle structured extraction well
- **Never mutate parsed items** — return new instances from every transformation step
- **Log pipeline metrics** (items parsed, LLM calls triggered, failures) so degradation is visible
- **TDD works well for parsers** — write tests for known patterns first, then add edge case inputs as you discover them
- **Handle failures gracefully** — a malformed item should be logged and skipped, not crash the pipeline

## Anti-Patterns to Avoid

- Sending all text to an LLM when regex handles 95%+ of cases — expensive, slow, and non-deterministic
- Using regex for genuinely free-form, variable text — LLM is the right tool there
- Skipping confidence scoring and hoping regex "just works" — edge cases will accumulate silently
- Mutating parsed objects during cleaning or validation steps
- Letting parse errors propagate as exceptions — log and continue
- Not testing with malformed input: missing fields, encoding issues, truncated records
