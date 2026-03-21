---
name: data-scraper-agent
description: Build a fully automated AI-powered data collection agent for any public source — job boards, prices, news, GitHub, sports, anything. Scrapes on a schedule, enriches data with a free LLM (Gemini Flash), stores results in Notion/Sheets/Supabase, and learns from user feedback. Runs 100% free on GitHub Actions. Use when the user wants to monitor, collect, or track any public data automatically, or says things like "build a bot that checks...", "monitor X for me", "collect data from...", or "alert me when...".
---

# Data Scraper Agent

Build a production-ready, AI-powered data collection agent for any public data source.
Runs on a schedule, enriches results with a free LLM, stores to a database, and improves over time.

**Stack: Python · Gemini Flash (free) · GitHub Actions (free) · Notion / Sheets / Supabase**

## Workflow

1. **Ask the five questions** (Step 1 below) to pin down scope before writing any code.
2. **Generate the directory structure** (Step 2) so the user sees the full shape upfront.
3. **Read `references/implementation.md`** for the full step-by-step code for Steps 3–9.
4. **Adapt code to the user's stack** — language, storage provider, source type.
5. **Apply the quality checklist** before marking the agent complete.
6. **Flag ethical/legal concerns proactively** — `robots.txt`, rate limits, ToS, login walls.

---

## Core Concepts

### The Three Layers

```
COLLECT → ENRICH → STORE
  │           │        │
Scraper    AI (LLM)  Database
runs on    scores /  Notion /
schedule   summarises Sheets /
           & classifies Supabase
```

### Free Stack

| Layer | Tool | Why |
|---|---|---|
| **Scraping** | `requests` + `BeautifulSoup` | No cost, covers 80% of public sites |
| **JS-rendered sites** | `playwright` (free) | When HTML scraping fails |
| **AI enrichment** | Gemini Flash via REST API | 500 req/day, 1M tokens/day — free |
| **Storage** | Notion API | Free tier, great UI for review |
| **Schedule** | GitHub Actions cron | Free for public repos |
| **Learning** | JSON feedback file in repo | Zero infra, persists in git |

### AI Model Fallback Chain

```
gemini-2.0-flash-lite (30 RPM) →
gemini-2.0-flash (15 RPM) →
gemini-2.5-flash (10 RPM) →
gemini-flash-lite-latest (final fallback)
```

### Batch AI Calls — Never One Per Item

```python
# ❌ BAD: 33 API calls for 33 items — hits rate limit instantly
for item in items:
    result = call_ai(item)

# ✅ GOOD: 7 API calls for 33 items (batch_size=5)
for batch in chunks(items, size=5):
    results = call_ai(batch)
```

---

## Step 1: Understand the Goal

Ask the user:

1. **What to collect:** "What data source? URL / API / RSS / public endpoint?"
2. **What to extract:** "What fields matter? Title, price, URL, date, score?"
3. **How to store:** "Where? Notion, Google Sheets, Supabase, or local file?"
4. **How to enrich:** "Should AI score, summarise, classify, or match each item?"
5. **Frequency:** "How often? Every hour, daily, weekly?"

Common prompts to surface the use case:
- Job boards → score relevance to resume
- Product prices → alert on drops
- GitHub repos → summarise new releases
- News feeds → classify by topic and sentiment
- Sports results → extract stats to a tracker
- Events calendar → filter by interest

---

## Step 2: Directory Structure

Generate this for the user:

```
my-agent/
├── config.yaml              # User customises this — no code changes needed
├── profile/
│   └── context.md           # User context the AI uses (resume, interests, criteria)
├── scraper/
│   ├── __init__.py
│   ├── main.py              # Orchestrator: scrape → enrich → store
│   ├── filters.py           # Rule-based pre-filter (fast, runs before AI)
│   └── sources/
│       ├── __init__.py
│       └── source_name.py   # One file per data source
├── ai/
│   ├── __init__.py
│   ├── client.py            # Gemini REST client with model fallback
│   ├── pipeline.py          # Batch AI analysis
│   └── memory.py            # Learn from user feedback
├── storage/
│   ├── __init__.py
│   └── notion_sync.py       # Or sheets_sync.py / supabase_sync.py
├── data/
│   └── feedback.json        # User decision history (auto-updated)
├── .env.example
├── .gitignore
├── setup.py                 # One-time DB/schema creation
├── requirements.txt
└── .github/
    └── workflows/
        └── scraper.yml      # GitHub Actions schedule
```

Then read `references/implementation.md` for the full code for each module.

---

## Scraping Patterns (Quick Reference)

Choose the right method for the source:

**REST API** (easiest — prefer this when available):
```python
resp = requests.get(url, headers=HEADERS, params={"q": query}, timeout=15)
resp.raise_for_status()
items = resp.json().get("results", [])
```

**HTML scraping** (handle None — `.select_one()` can return None):
```python
soup = BeautifulSoup(resp.text, "lxml")
for card in soup.select(".listing-card"):
    heading = card.select_one("h2, h3")
    if not heading:
        continue                          # ← guard against NoneType crash
    title = heading.get_text(strip=True)
    a_tag = card.select_one("a")
    if not a_tag:
        continue
    href = a_tag["href"]
    if not href.startswith("http"):
        href = f"https://example.com{href}"
```

**RSS feed**:
```python
import xml.etree.ElementTree as ET
root = ET.fromstring(resp.text)
for item in root.findall(".//item"):
    title = item.findtext("title", "")
    link = item.findtext("link", "")
    pub_date = item.findtext("pubDate", "")
```

**Paginated API**:
```python
page = 1
while True:
    resp = requests.get(url, params={"page": page, "limit": 50}, timeout=15)
    data = resp.json()
    items = data.get("results", [])
    if not items or not data.get("has_more"):
        break
    for item in items:
        results.append(_normalise(item))
    page += 1
    time.sleep(1)                         # ← rate limit between pages
```

**JS-rendered pages** (Playwright — only when HTML scraping returns empty):
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch()
    page = browser.new_page()
    page.goto(url, wait_until="networkidle")
    page.wait_for_selector(".listing", timeout=10_000)
    html = page.content()
    browser.close()

soup = BeautifulSoup(html, "lxml")
```

---

## Ethical & Legal Guidelines

Always surface these before building:

- **Check `robots.txt`** — respect `Disallow` rules; use the public API instead if one exists
- **Rate limit all HTTP requests** — add `time.sleep(1–2)` between page fetches; don't hammer servers
- **No login walls** — only scrape publicly accessible pages; never automate authentication
- **Terms of Service** — some sites explicitly prohibit scraping; flag this to the user
- **Personal data** — avoid scraping PII; if unavoidable, don't store it

---

## config.yaml Template

```yaml
# Customise this file — no code changes needed

filters:
  required_keywords: []      # item must contain at least one (empty = no filter)
  blocked_keywords: []       # item must not contain any

priorities:
  - "example priority 1"
  - "example priority 2"

storage:
  provider: "notion"         # notion | sheets | supabase | sqlite

feedback:
  positive_statuses: ["Saved", "Applied", "Interested"]
  negative_statuses: ["Skip", "Rejected", "Not relevant"]

ai:
  enabled: true
  model: "gemini-2.5-flash"
  min_score: 0               # filter out items below this score (0 = keep all)
  rate_limit_seconds: 7      # seconds between Gemini API calls
  batch_size: 5              # items per API call
```

---

## Free Tier Limits Reference

| Service | Free Limit | Notes |
|---|---|---|
| Gemini Flash Lite | 30 RPM, 1500 RPD | ~56 req/day at 3-hr intervals |
| Gemini 2.0 Flash | 15 RPM, 1500 RPD | Good fallback |
| Gemini 2.5 Flash | 10 RPM, 500 RPD | Use sparingly |
| GitHub Actions | Unlimited (public repos) | ~20 min/day typical |
| Notion API | Unlimited | ~200 writes/day practical |
| Supabase | 500MB DB, 2GB transfer | Fine for most agents |
| Google Sheets API | 300 req/min | Works for small agents |

---

## Anti-Patterns

| ❌ Anti-pattern | Problem | ✅ Fix |
|---|---|---|
| One LLM call per item | Hits rate limits instantly | Batch 5 items per call |
| Hardcoded keywords in code | Not reusable | Move all config to `config.yaml` |
| No `time.sleep` between requests | IP ban | Rate limit scraper, not just AI |
| Secrets in code or config | Security risk | `.env` + GitHub Secrets |
| No deduplication | Duplicate rows pile up | Check URL before every push |
| Ignoring `robots.txt` | Legal / ethical risk | Respect crawl rules; prefer public APIs |
| JS-rendered sites with `requests` | Empty response | Use Playwright or find the underlying API |
| `maxOutputTokens` too low | Truncated JSON, parse errors | Use 2048+ for batch responses |
| No null guard on `.select_one()` | NoneType crash | Always check before `.get_text()` |

---

## Quality Checklist

Before marking the agent complete:

- [ ] `config.yaml` controls all user-facing settings — no hardcoded values in code
- [ ] `profile/context.md` holds user-specific context for AI matching
- [ ] `time.sleep` between HTTP page fetches (not just between AI calls)
- [ ] Deduplication by URL before every storage push
- [ ] Gemini client has 4-model fallback chain
- [ ] Batch size ≤ 5 items per AI call; `maxOutputTokens` ≥ 2048
- [ ] `.env` in `.gitignore`; `.env.example` provided
- [ ] GitHub Actions workflow commits `feedback.json` after each run
- [ ] HTML scraping has null guards on all `.select_one()` calls
- [ ] `robots.txt` and ToS reviewed with user

---

## Real-World Examples

```
"Build me an agent that monitors Hacker News for AI startup funding news"
"Scrape product prices from 3 e-commerce sites and alert when they drop"
"Track new GitHub repos tagged 'llm' or 'agents' — summarise each one"
"Collect Chief of Staff job listings and score them against my resume"
"Monitor a subreddit for posts mentioning my company — classify sentiment"
"Scrape new arXiv papers on a topic daily"
"Track sports fixture results in a Google Sheet"
"Build a real estate listing watcher — alert on new properties under ₹1 Cr"
```
