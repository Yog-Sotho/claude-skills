---
name: hugging-face-tool-builder
description: >-
  Create reusable, composable CLI scripts and utilities for the Hugging Face
  API and hf CLI. Builds bash/Python/TSX tools that chain, pipe, and stream
  HF API data — model search, dataset discovery, model card parsing, trending
  models, paper lookups, repo management, and more. Always activate when the
  user wants to query the HF API, build an HF shell script or pipeline, search
  or filter HF models/datasets, parse model cards, automate Hub tasks, or says
  anything like "write a script to..." or "build a tool that..." involving
  Hugging Face — even if they don't say "skill".
compatibility: "Requires: HF_TOKEN env var (Read-access minimum), jq, curl. Optional: hf CLI (huggingface_hub), uv or tsx for Python/TypeScript scripts."
---

# Hugging Face Tool Builder
<!-- v1.1.0 -->

## ⚡ Quick Setup

```bash
# Required: HF token (Read access for public data; Write for uploads/repos)
export HF_TOKEN="hf_your_token_here"

# Verify auth
curl -s -H "Authorization: Bearer ${HF_TOKEN}" https://huggingface.co/api/whoami-v2 | jq .name

# Check hf CLI (optional but unlocks repo/jobs/upload commands)
hf version

# Check jq (required for all JSON processing)
jq --version
```

---

## Language & Tool Selection

| Use | When |
|-----|------|
| **bash + curl + jq** | Default — single API calls, pipelines, streaming NDJSON, simple filters |
| **Python (uv run)** | Complex logic, multi-step transforms, pagination, reuse across scripts |
| **TSX** | Type-safe data contracts, user-facing CLI tools, structured output schemas |
| **hf CLI** | Repo management, file download/upload, jobs, cache — anything needing Hub auth flows |

When in doubt: start with bash. Upgrade to Python only if branching/looping complexity warrants it.

---

## Script Standards

All scripts must follow these rules:

**Interface:**
- `--help` flag required — describe all inputs, outputs, and env vars
- Accept `--token` as an explicit override for `HF_TOKEN` (fall back to env var)
- Emit to stdout; errors/warnings to stderr

**Auth:**
```bash
TOKEN="${HF_TOKEN:-}"
curl -s -H "Authorization: Bearer ${TOKEN}" https://huggingface.co/api/...
```
Always use `HF_TOKEN` as the auth header — it raises rate limits and unlocks gated/private content.

**Error handling (bash):**
```bash
set -euo pipefail
# Check deps at startup
command -v jq >/dev/null 2>&1 || { echo "jq required" >&2; exit 1; }
```

**Output format:**
- Pipeable scripts → NDJSON (one JSON object per line): `jq -c '.'`
- Terminal/final output → pretty JSON: `jq '.'`
- Plain lists → one item per line (grep/xargs friendly)

**Delivery:**
- Save finished scripts to `scripts/` relative to this skill dir
- Make executable: `chmod +x scripts/my_script.sh`
- Share usage examples and a sample pipeline invocation

**Investigate before committing:** Query a small sample (`?limit=5`) to understand the API response shape before building the full script.

---

## API Endpoints

Base URL: `https://huggingface.co`

| Endpoint | Purpose |
|----------|---------|
| `/api/models` | Search/filter models (by task, library, language, etc.) |
| `/api/datasets` | Search/filter datasets |
| `/api/spaces` | Search/filter Spaces |
| `/api/collections` | Browse curated collections |
| `/api/daily_papers` | Today's featured papers |
| `/api/trending` | Trending models, datasets, spaces |
| `/api/whoami-v2` | Verify token identity and permissions |
| `/api/notifications` | User notifications (auth required) |
| `/api/settings` | User settings (auth required) |
| `/oauth/userinfo` | OAuth user info |

### Discovering Endpoint Details

The full API is documented via OpenAPI at `https://huggingface.co/.well-known/openapi.json`.

⚠️ **Do not fetch this URL directly** — it's too large to process. Use `jq` to extract only what you need:

```bash
# List all ~160 endpoints
curl -s "https://huggingface.co/.well-known/openapi.json" | jq '.paths | keys | sort'

# Inspect a specific endpoint's parameters
curl -s "https://huggingface.co/.well-known/openapi.json" | jq '.paths["/api/models"]'

# Explore response schema for models
curl -s "https://huggingface.co/.well-known/openapi.json" | jq '.components.schemas.ModelInfo'
```

Always constrain exploratory queries to small result sets (`?limit=3`) to understand shape before designing the full script.

---

## hf CLI Reference

The `hf` CLI is the preferred tool for repo/file/jobs operations:

```
hf auth          — login, logout, token management
hf download      — download files or full repos from the Hub
hf upload        — upload files or folders
hf repo          — create, delete, move repos
hf repo-files    — list, delete individual files in a repo
hf jobs          — run and manage HF Jobs (GPU/CPU compute)
hf cache         — inspect and clean local model cache
hf env           — print environment info (useful for debugging)
```

Most useful for scripting: `hf download`, `hf upload`, `hf repo-files`, `hf jobs uv run`

---

## Reference Scripts

All paths are relative to this skill directory. Read the relevant file before designing a script of that type.

**Feature examples** (read for patterns before building similar scripts):
- `references/hf_model_papers_auth.sh` — multi-step chain: trending → model metadata → model card parsing; auth hygiene for gated content
- `references/find_models_by_paper.sh` — resilient query strategy with `--token` flag, retry logic for narrow arXiv searches
- `references/hf_model_card_frontmatter.sh` — `hf` CLI download + YAML frontmatter extraction → NDJSON (license, pipeline tag, gated flag)

**Baseline starters** (minimal boilerplate for each language):
- `references/baseline_hf_api.sh` — bash + curl + jq, raw JSON, `HF_TOKEN` header
- `references/baseline_hf_api.py` — Python baseline
- `references/baseline_hf_api.tsx` — TypeScript executable baseline

**Composable utility:**
- `references/hf_enrich_models.sh` — reads model IDs from stdin → fetches metadata per ID → emits NDJSON (use as a pipeline stage)

---

## Composable Pipeline Patterns

These patterns demonstrate the preferred piping style:

```bash
# Top 10 trending models by downloads
references/baseline_hf_api.sh 25 \
  | jq -r '.[].id' \
  | references/hf_enrich_models.sh \
  | jq -s 'sort_by(.downloads) | reverse | .[:10]'

# Extract id + downloads, sort inline
references/baseline_hf_api.sh 50 \
  | jq '[.[] | {id, downloads}] | sort_by(.downloads) | reverse | .[:10]'

# Batch model card frontmatter for multiple models
printf '%s\n' openai/gpt-oss-120b meta-llama/Meta-Llama-3.1-8B \
  | references/hf_model_card_frontmatter.sh \
  | jq -s 'map({id, license, has_extra_gated_prompt})'

# Filter to only Apache-2.0 text-generation models
curl -s "https://huggingface.co/api/models?filter=text-generation&limit=50" \
  -H "Authorization: Bearer ${HF_TOKEN}" \
  | jq '[.[] | select(.cardData.license == "apache-2.0") | {id, downloads, likes}]'
```

---

## Common Script Patterns

### Paginated fetch (Python with uv)
```python
# /// script
# dependencies = ["huggingface-hub>=0.26", "requests"]
# ///
import requests, os, sys

token = os.environ.get("HF_TOKEN", "")
headers = {"Authorization": f"Bearer {token}"} if token else {}
url = "https://huggingface.co/api/models"
params = {"limit": 100, "full": "true"}

while url:
    r = requests.get(url, headers=headers, params=params)
    r.raise_for_status()
    for model in r.json():
        print(model["id"])
    # HF API returns Link header for pagination
    link = r.headers.get("Link", "")
    url = link.split(";")[0].strip("<>") if 'rel="next"' in link else None
    params = {}
```

### Model card frontmatter extraction (bash)
```bash
MODEL_ID="meta-llama/Meta-Llama-3.1-8B"
hf download "${MODEL_ID}" README.md --local-dir /tmp/card
python3 -c "
import sys
text = open('/tmp/card/README.md').read()
import re
m = re.search(r'^---\n(.*?)\n---', text, re.DOTALL)
print(m.group(1) if m else '')
"
```

### Search with fallback query strategy
```bash
search_models() {
  local query="$1"
  local result
  result=$(curl -s "https://huggingface.co/api/models?search=${query}&limit=10" \
    -H "Authorization: Bearer ${HF_TOKEN}")
  count=$(echo "$result" | jq 'length')
  if [ "$count" -eq 0 ]; then
    # Strip arXiv prefix and retry
    query=$(echo "$query" | sed 's/^arxiv://i')
    result=$(curl -s "https://huggingface.co/api/models?search=${query}&limit=10" \
      -H "Authorization: Bearer ${HF_TOKEN}")
  fi
  echo "$result"
}
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `401 Unauthorized` | Missing or expired `HF_TOKEN` | Re-export a valid token |
| `429 Too Many Requests` | Rate limited (no token) | Set `HF_TOKEN` — raises limits significantly |
| `403 Forbidden` on gated model | Token lacks gated access | Accept model terms on HF website, then retry |
| Empty results from search | Query too narrow (e.g. bare arXiv ID) | Strip prefix, broaden query, check spelling |
| `jq: command not found` | jq not installed | `brew install jq` / `apt install jq` |
| `hf: command not found` | hf CLI not installed | `pip install huggingface_hub` (installs `hf`) |
| Pagination stops early | Default limit too low | Add `?limit=100` and follow `Link` response header |
| Gated model card missing | README not in default download | Use `hf download model README.md` explicitly |
