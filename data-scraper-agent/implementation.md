# Implementation Reference — Steps 3–9

Full code for each module in the data-scraper-agent architecture.
Read this file when building the agent after completing Steps 1–2 in SKILL.md.

---

## Step 3: Scraper Source Template

```python
# scraper/sources/my_source.py
"""
[Source Name] — scrapes [what] from [where].
Method: [REST API / HTML scraping / RSS feed]
"""
import time
import requests
from bs4 import BeautifulSoup
from datetime import datetime, timezone
from scraper.filters import is_relevant

HEADERS = {
    "User-Agent": "Mozilla/5.0 (compatible; research-bot/1.0)",
}


def fetch() -> list[dict]:
    """
    Returns items with a consistent schema.
    Every item must have at minimum: name, url, date_found.
    """
    results = []

    resp = requests.get("https://api.example.com/items", headers=HEADERS, timeout=15)
    resp.raise_for_status()

    for item in resp.json().get("results", []):
        if not is_relevant(item.get("title", "")):
            continue
        results.append(_normalise(item))
        time.sleep(0.5)                   # ← polite delay between items

    return results


def _normalise(raw: dict) -> dict:
    """Convert raw API/HTML data to the standard schema."""
    return {
        "name": raw.get("title", ""),
        "url": raw.get("link", ""),
        "source": "MySource",
        "date_found": datetime.now(timezone.utc).date().isoformat(),
        # add domain-specific fields here
    }
```

---

## Step 4: Gemini AI Client

```python
# ai/client.py
import os
import json
import time
import requests

MODEL_FALLBACK = [
    "gemini-2.0-flash-lite",
    "gemini-2.0-flash",
    "gemini-2.5-flash",
    "gemini-flash-lite-latest",
]

_last_call: float = 0.0


def generate(prompt: str, model: str = "", rate_limit: float = 7.0) -> dict:
    """Call Gemini with auto-fallback on 429. Returns parsed JSON or {}."""
    global _last_call

    api_key = os.environ.get("GEMINI_API_KEY", "")
    if not api_key:
        return {}

    # Enforce rate limit before the call
    elapsed = time.time() - _last_call
    if elapsed < rate_limit:
        time.sleep(rate_limit - elapsed)

    models = [model] + [m for m in MODEL_FALLBACK if m != model] if model else MODEL_FALLBACK

    for m in models:
        url = (
            f"https://generativelanguage.googleapis.com/v1beta/models/"
            f"{m}:generateContent?key={api_key}"
        )
        payload = {
            "contents": [{"parts": [{"text": prompt}]}],
            "generationConfig": {
                "responseMimeType": "application/json",
                "temperature": 0.3,
                "maxOutputTokens": 2048,
            },
        }
        try:
            resp = requests.post(url, json=payload, timeout=30)
            _last_call = time.time()       # ← record time AFTER the call completes
            if resp.status_code == 200:
                return _parse(resp)
            if resp.status_code in (429, 404):
                time.sleep(2)
                continue
            return {}
        except requests.RequestException:
            _last_call = time.time()
            return {}

    return {}


def _parse(resp) -> dict:
    try:
        text = (
            resp.json()
            .get("candidates", [{}])[0]
            .get("content", {})
            .get("parts", [{}])[0]
            .get("text", "")
            .strip()
        )
        # Strip accidental markdown fences
        if text.startswith("```"):
            text = text.split("\n", 1)[-1].rsplit("```", 1)[0]
        return json.loads(text)
    except (json.JSONDecodeError, KeyError, IndexError):
        return {}
```

---

## Step 5: AI Pipeline (Batch)

```python
# ai/pipeline.py
import json
import yaml
from pathlib import Path
from ai.client import generate


def analyse_batch(
    items: list[dict],
    context: str = "",
    preference_prompt: str = "",
) -> list[dict]:
    """Analyse items in batches. Returns items enriched with AI fields."""
    config = yaml.safe_load((Path(__file__).parent.parent / "config.yaml").read_text())
    ai_cfg = config.get("ai", {})
    model = ai_cfg.get("model", "gemini-2.5-flash")
    rate_limit = ai_cfg.get("rate_limit_seconds", 7.0)
    min_score = ai_cfg.get("min_score", 0)
    batch_size = ai_cfg.get("batch_size", 5)

    batches = [items[i : i + batch_size] for i in range(0, len(items), batch_size)]
    print(f"  [AI] {len(items)} items → {len(batches)} API calls")

    enriched = []
    for i, batch in enumerate(batches):
        print(f"  [AI] Batch {i + 1}/{len(batches)}...")
        prompt = _build_prompt(batch, context, preference_prompt, config)
        result = generate(prompt, model=model, rate_limit=rate_limit)

        analyses = result.get("analyses", [])
        for j, item in enumerate(batch):
            ai = analyses[j] if j < len(analyses) else {}
            if ai:
                score = max(0, min(100, int(ai.get("score", 0))))
                if min_score and score < min_score:
                    continue
                enriched.append({
                    **item,
                    "ai_score": score,
                    "ai_summary": ai.get("summary", ""),
                    "ai_notes": ai.get("notes", ""),
                })
            else:
                enriched.append(item)

    return enriched


def _build_prompt(batch, context, preference_prompt, config):
    priorities = config.get("priorities", [])
    items_text = "\n\n".join(
        f"Item {i + 1}: {json.dumps({k: v for k, v in item.items() if not k.startswith('_')})}"
        for i, item in enumerate(batch)
    )

    return f"""Analyse these {len(batch)} items and return a JSON object.

# Items
{items_text}

# User Context
{context[:800] if context else "Not provided"}

# User Priorities
{chr(10).join(f"- {p}" for p in priorities)}

{preference_prompt}

# Instructions
Return: {{"analyses": [{{"score": <0-100>, "summary": "<2 sentences>", "notes": "<why this matches or doesn't>"}} for each item in order]}}
Be concise. Score 90+=excellent match, 70-89=good, 50-69=ok, <50=weak.
Return ONLY valid JSON. No markdown fences, no preamble."""
```

---

## Step 6: Feedback Learning System

```python
# ai/memory.py
"""Learn from user decisions to improve future scoring."""
import json
from pathlib import Path

FEEDBACK_PATH = Path(__file__).parent.parent / "data" / "feedback.json"


def load_feedback() -> dict:
    if FEEDBACK_PATH.exists():
        try:
            return json.loads(FEEDBACK_PATH.read_text())
        except (json.JSONDecodeError, OSError):
            pass
    return {"positive": [], "negative": []}


def save_feedback(fb: dict) -> None:
    FEEDBACK_PATH.parent.mkdir(parents=True, exist_ok=True)
    FEEDBACK_PATH.write_text(json.dumps(fb, indent=2))


def build_preference_prompt(feedback: dict, max_examples: int = 15) -> str:
    """Convert feedback history into a prompt bias section."""
    lines = []
    if feedback.get("positive"):
        lines.append("# Items the user LIKED (positive signal):")
        for e in feedback["positive"][-max_examples:]:
            lines.append(f"- {e}")
    if feedback.get("negative"):
        lines.append("\n# Items the user SKIPPED/REJECTED (negative signal):")
        for e in feedback["negative"][-max_examples:]:
            lines.append(f"- {e}")
    if lines:
        lines.append("\nUse these patterns to bias scoring on new items.")
    return "\n".join(lines)
```

**Wiring feedback:** after each run, query your storage for items with positive/negative statuses and call `save_feedback()` with the extracted patterns. Implement this as a separate `feedback_sync.py` script triggered by the same GitHub Actions workflow.

---

## Step 7: Notion Storage

```python
# storage/notion_sync.py
import os
from notion_client import Client
from notion_client.errors import APIResponseError

_client = None


def get_client() -> Client:
    global _client
    if _client is None:
        _client = Client(auth=os.environ["NOTION_TOKEN"])
    return _client


def get_existing_urls(db_id: str) -> set[str]:
    """Fetch all stored URLs for deduplication. Handles pagination."""
    client, seen, cursor = get_client(), set(), None
    while True:
        kwargs = {"start_cursor": cursor} if cursor else {}
        resp = client.databases.query(database_id=db_id, page_size=100, **kwargs)
        for page in resp["results"]:
            url = page["properties"].get("URL", {}).get("url", "")
            if url:
                seen.add(url)
        if not resp["has_more"]:
            break
        cursor = resp["next_cursor"]
    return seen


def push_item(db_id: str, item: dict) -> bool:
    """Push one item to Notion. Returns True on success."""
    props = {
        "Name": {"title": [{"text": {"content": item.get("name", "")[:100]}}]},
        "URL": {"url": item.get("url")},
        "Source": {"select": {"name": item.get("source", "Unknown")}},
        "Date Found": {"date": {"start": item.get("date_found")}},
        "Status": {"select": {"name": "New"}},
    }
    if item.get("ai_score") is not None:
        props["AI Score"] = {"number": item["ai_score"]}
    if item.get("ai_summary"):
        props["Summary"] = {"rich_text": [{"text": {"content": item["ai_summary"][:2000]}}]}
    if item.get("ai_notes"):
        props["Notes"] = {"rich_text": [{"text": {"content": item["ai_notes"][:2000]}}]}

    try:
        get_client().pages.create(parent={"database_id": db_id}, properties=props)
        return True
    except APIResponseError as e:
        print(f"[notion] Push failed: {e}")
        return False


def sync(db_id: str, items: list[dict]) -> tuple[int, int]:
    existing = get_existing_urls(db_id)
    added = skipped = 0
    for item in items:
        if item.get("url") in existing:
            skipped += 1
            continue
        if push_item(db_id, item):
            added += 1
            existing.add(item["url"])
        else:
            skipped += 1
    return added, skipped
```

---

## Step 8: Orchestrator (main.py)

```python
# scraper/main.py
import os
import sys
import yaml
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

from scraper.sources import my_source          # add your sources here

# NOTE: swap this import for storage.sheets_sync or storage.supabase_sync
# if the user chose a different storage provider.
from storage.notion_sync import sync


SOURCES = [
    ("My Source", my_source.fetch),
]


def ai_enabled() -> bool:
    return bool(os.environ.get("GEMINI_API_KEY"))


def main() -> None:
    # Read config once — do not re-read inside the same run
    config = yaml.safe_load((Path(__file__).parent.parent / "config.yaml").read_text())
    provider = config.get("storage", {}).get("provider", "notion")

    if provider == "notion":
        db_id = os.environ.get("NOTION_DATABASE_ID")
        if not db_id:
            print("ERROR: NOTION_DATABASE_ID not set")
            sys.exit(1)
    else:
        # Extend here for sheets (SHEET_ID) or supabase (SUPABASE_TABLE)
        print(f"ERROR: provider '{provider}' not yet wired in main.py")
        sys.exit(1)

    all_items: list[dict] = []
    for name, fetch_fn in SOURCES:
        try:
            items = fetch_fn()
            print(f"[{name}] {len(items)} items")
            all_items.extend(items)
        except Exception as e:
            print(f"[{name}] FAILED: {e}")

    # Deduplicate by URL
    seen: set[str] = set()
    deduped: list[dict] = []
    for item in all_items:
        url = item.get("url", "")
        if url and url not in seen:
            seen.add(url)
            deduped.append(item)

    print(f"Unique items after dedup: {len(deduped)}")

    if ai_enabled() and deduped:
        from ai.memory import load_feedback, build_preference_prompt
        from ai.pipeline import analyse_batch

        feedback = load_feedback()
        preference = build_preference_prompt(feedback)
        context_path = Path(__file__).parent.parent / "profile" / "context.md"
        context = context_path.read_text() if context_path.exists() else ""
        deduped = analyse_batch(deduped, context=context, preference_prompt=preference)
    else:
        print("[AI] Skipped — GEMINI_API_KEY not set or no items")

    added, skipped = sync(db_id, deduped)
    print(f"Done — {added} new, {skipped} existing/skipped")


if __name__ == "__main__":
    main()
```

---

## Step 9: GitHub Actions Workflow

```yaml
# .github/workflows/scraper.yml
name: Data Scraper Agent

on:
  schedule:
    - cron: "0 */3 * * *"   # every 3 hours — adjust to your needs
  workflow_dispatch:          # allow manual trigger

permissions:
  contents: write             # required to commit feedback.json

jobs:
  scrape:
    runs-on: ubuntu-latest
    timeout-minutes: 20

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"

      - run: pip install -r requirements.txt

      # Uncomment if Playwright is in requirements.txt
      # - name: Install Playwright browsers
      #   run: python -m playwright install chromium --with-deps

      - name: Run agent
        env:
          NOTION_TOKEN: ${{ secrets.NOTION_TOKEN }}
          NOTION_DATABASE_ID: ${{ secrets.NOTION_DATABASE_ID }}
          GEMINI_API_KEY: ${{ secrets.GEMINI_API_KEY }}
        run: python -m scraper.main

      - name: Commit feedback history
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add data/feedback.json || true
          git diff --cached --quiet || git commit -m "chore: update feedback history"
          git push
```

---

## Requirements Template

```
requests==2.32.3
beautifulsoup4==4.12.3
lxml==5.3.0
python-dotenv==1.0.1
pyyaml==6.0.2
notion-client==2.2.1    # if using Notion
# playwright==1.49.0    # uncomment for JS-rendered sites
```
