---
name: hugging-face-datasets
description: >-
  Create, manage, query, and transform Hugging Face Hub datasets for LLM
  fine-tuning, SFT, RLHF, DPO, and training data pipelines. Supports
  initializing repos, defining configs/system prompts, streaming row updates,
  SQL-based querying/transformation via DuckDB, and pushing subsets to the Hub.
  Designed to work alongside the HF MCP server. Always activate when the user
  mentions Hugging Face datasets, HF Hub, training data creation, dataset
  preprocessing, pushing to Hub, querying a dataset with SQL, building a
  fine-tuning dataset, or filtering/transforming HF datasets — even if they
  don't say "skill".
compatibility: "Requires uv (https://astral.sh/uv) and HF_TOKEN env var (Write-access). Optional: HF MCP server."
---

# Hugging Face Datasets Skill
<!-- v2.2.0 -->

## ⚡ Quick Setup

Before running any script, ensure:

```bash
# 1. Install uv (if not already installed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. Set your HF token (Write access required for push operations)
export HF_TOKEN="hf_your_token_here"
# For private datasets, token must have read+write repo permissions.
```

All scripts use PEP 723 inline dependency management — `uv run` auto-installs
requirements on first run. No manual `pip install` needed.

---

## Tool Selection Matrix

| Task | Use |
|------|-----|
| Discover / search datasets | **HF MCP Server** (`search_datasets`, `get_dataset_details`) |
| Get dataset metadata / README | **HF MCP Server** |
| Create a new dataset repo | `dataset_manager.py init` |
| Add / stream rows to a dataset | `dataset_manager.py add_rows` |
| Query / filter / transform data | `sql_manager.py` (DuckDB SQL) |
| Push a subset to Hub | `sql_manager.py --push-to` |
| Export to Parquet / JSONL | `sql_manager.py export` |
| Python pipeline integration | `HFDatasetSQL` class in `sql_manager.py` |

---

## Scripts

Both scripts live in `scripts/` relative to this SKILL.md:

- `scripts/sql_manager.py` — SQL querying, filtering, transformation, export
- `scripts/dataset_manager.py` — Dataset creation, configuration, row management

---

## SQL Querying (`sql_manager.py`)

Uses DuckDB's `hf://` protocol for direct, zero-download access to any public
(or private, with token) dataset.

### HF Path Format

```
hf://datasets/{dataset_id}@~parquet/{config}/{split}/*.parquet
```

The `@~parquet` revision auto-converts any format to Parquet on the fly.

### Core Commands

```bash
# Explore structure
uv run scripts/sql_manager.py describe --dataset "cais/mmlu"
uv run scripts/sql_manager.py unique   --dataset "cais/mmlu" --column "subject"
uv run scripts/sql_manager.py histogram --dataset "cais/mmlu" --column "subject"
uv run scripts/sql_manager.py sample   --dataset "cais/mmlu" --n 5

# Query
uv run scripts/sql_manager.py query \
  --dataset "cais/mmlu" \
  --sql "SELECT * FROM data WHERE subject='nutrition' LIMIT 10"

# Count with filter
uv run scripts/sql_manager.py count --dataset "cais/mmlu" --where "subject='nutrition'"

# Structured transform (no raw SQL required)
uv run scripts/sql_manager.py transform \
  --dataset "cais/mmlu" \
  --select "subject, COUNT(*) as cnt" \
  --group-by "subject" \
  --order-by "cnt DESC" \
  --limit 10

# Export locally
uv run scripts/sql_manager.py export \
  --dataset "cais/mmlu" \
  --sql "SELECT * FROM data WHERE subject='nutrition'" \
  --output "nutrition.parquet" --format parquet   # or: jsonl, csv

# Query a specific config or split
uv run scripts/sql_manager.py query \
  --dataset "ibm/duorc" --config "ParaphraseRC" --split "test" \
  --sql "SELECT * FROM data LIMIT 5"

# Query all splits at once
uv run scripts/sql_manager.py query \
  --dataset "cais/mmlu" --split "*" \
  --sql "SELECT COUNT(*) FROM data"

# Push result to Hub (creates new dataset)
uv run scripts/sql_manager.py query \
  --dataset "cais/mmlu" \
  --sql "SELECT * FROM data WHERE subject IN ('nutrition','anatomy','clinical_knowledge')" \
  --push-to "{your-username}/mmlu-medical-subset" --private

# Raw SQL — cross-dataset joins, full hf:// paths
uv run scripts/sql_manager.py raw --sql "
  SELECT a.*, b.*
  FROM 'hf://datasets/dataset1@~parquet/default/train/*.parquet' a
  JOIN 'hf://datasets/dataset2@~parquet/default/train/*.parquet' b
  ON a.id = b.id
  LIMIT 100
"
```

### SQL Reference

```sql
-- Use `data` as the table alias in all queries (auto-expanded to hf:// path)

-- String ops
LENGTH(col)                         -- character count
LOWER(col), UPPER(col)
regexp_replace(col, '\n', ' ')      -- regex replace
regexp_matches(col, 'pattern')      -- regex filter

-- Array ops (HF arrays are 1-indexed in DuckDB)
choices[1]                          -- first element
array_length(choices)               -- array size
unnest(choices)                     -- expand array to rows
-- Note: MMLU `answer` field is 0–3; use choices[answer+1] for 1-indexed access

-- Aggregations
COUNT(*), SUM(col), AVG(col)
GROUP BY col HAVING cnt > 100

-- Reproducible sampling
USING SAMPLE 1000                       -- random N rows
USING SAMPLE 10 PERCENT (RESERVOIR, 42) -- reproducible 10%

-- Window functions
ROW_NUMBER() OVER (PARTITION BY subject ORDER BY question)
```

### Python API

```python
from sql_manager import HFDatasetSQL

sql = HFDatasetSQL()

results  = sql.query("cais/mmlu", "SELECT * FROM data WHERE subject='nutrition' LIMIT 10")
schema   = sql.describe("cais/mmlu")
samples  = sql.sample("cais/mmlu", n=5, seed=42)
count    = sql.count("cais/mmlu", where="subject='nutrition'")
dist     = sql.histogram("cais/mmlu", "subject")

url = sql.push_to_hub(
    "cais/mmlu",
    "{your-username}/nutrition-subset",
    sql="SELECT * FROM data WHERE subject='nutrition'",
    private=True
)

sql.export_to_parquet("cais/mmlu", "output.parquet", sql="SELECT * FROM data LIMIT 100")
sql.close()
```

---

## Dataset Creation (`dataset_manager.py`)

### Recommended Workflow

```bash
# 1. DISCOVER — use HF MCP server
#    search_datasets("conversational AI training")
#    get_dataset_details("{username}/dataset-name")

# 2. INITIALIZE
uv run scripts/dataset_manager.py init --repo_id "{your-username}/dataset-name" [--private]

# 3. CONFIGURE (attach system prompt / metadata)
uv run scripts/dataset_manager.py config \
  --repo_id "{your-username}/dataset-name" \
  --system_prompt "$(cat system_prompt.txt)"

# 4. QUICK SETUP (init + template in one step)
uv run scripts/dataset_manager.py quick_setup \
  --repo_id "{your-username}/dataset-name" --template classification

# 5. ADD ROWS
uv run scripts/dataset_manager.py add_rows \
  --repo_id "{your-username}/dataset-name" \
  --template qa \
  --rows_json '[{"question": "What is AI?", "answer": "Artificial Intelligence..."}]'

# 6. STATS
uv run scripts/dataset_manager.py stats --repo_id "{your-username}/dataset-name"

# 7. LIST TEMPLATES
uv run scripts/dataset_manager.py list_templates
```

### Data Templates

**Chat** (`--template chat`) — multi-turn / tool-use conversations
```json
{
  "messages": [
    {"role": "user",      "content": "Natural user request"},
    {"role": "assistant", "content": "Response with tool usage"},
    {"role": "tool",      "content": "Tool response", "tool_call_id": "call_123"}
  ],
  "scenario": "Description of use case",
  "complexity": "simple|intermediate|advanced"
}
```

**Classification** (`--template classification`)
```json
{"text": "Input text", "label": "class_label", "confidence": 0.95,
 "metadata": {"domain": "technology", "language": "en"}}
```

**QA** (`--template qa`)
```json
{"question": "...", "answer": "...", "context": "...",
 "answer_type": "factual|explanatory|opinion", "difficulty": "easy|medium|hard"}
```

**Completion** (`--template completion`)
```json
{"prompt": "...", "completion": "...",
 "domain": "code|creative|technical|conversational", "style": "..."}
```

**Tabular** (`--template tabular`)
```json
{
  "columns": [{"name": "feature1", "type": "numeric", "description": "..."}],
  "data":    [{"feature1": 123, "target": "class_a"}]
}
```

### Bundled Example Sets

The `examples/` directory contains ready-to-use training rows:

| File | Contents |
|------|----------|
| `examples/training_examples.json` | MCP tool-use: debugging, project setup, DB analysis |
| `examples/diverse_training_examples.json` | Educational chat, git workflows, code analysis, content generation |

```bash
# Use one set
uv run scripts/dataset_manager.py add_rows \
  --repo_id "{your-username}/dataset-name" \
  --rows_json "$(cat examples/training_examples.json)"

# Merge both sets
uv run scripts/dataset_manager.py add_rows \
  --repo_id "{your-username}/dataset-name" \
  --rows_json "$(jq -s '.[0] + .[1]' examples/training_examples.json examples/diverse_training_examples.json)"
```

---

## End-to-End Workflows

### Build a Fine-Tuning Subset from an Existing Dataset
```bash
uv run scripts/sql_manager.py describe  --dataset "cais/mmlu"
uv run scripts/sql_manager.py histogram --dataset "cais/mmlu" --column "subject"
uv run scripts/sql_manager.py query \
  --dataset "cais/mmlu" \
  --sql "SELECT question, choices[answer+1] AS correct_answer, subject FROM data
         WHERE subject IN ('nutrition','anatomy','clinical_knowledge')" \
  --push-to "{your-username}/mmlu-medical-qa" --private
```

### Quality-Filtered SFT Dataset
```bash
uv run scripts/sql_manager.py query \
  --dataset "squad" \
  --sql "SELECT * FROM data
         WHERE LENGTH(context) > 500 AND LENGTH(question) > 20" \
  --push-to "{your-username}/squad-filtered"
```

### Multi-Split Merge → Parquet
```bash
uv run scripts/sql_manager.py export \
  --dataset "cais/mmlu" --split "*" \
  --output "mmlu_all.parquet"
```

### Process Locally → Push as Training Dataset
```bash
# 1. Extract raw source
uv run scripts/sql_manager.py export \
  --dataset "cais/mmlu" \
  --sql "SELECT question, subject FROM data WHERE subject='nutrition'" \
  --output "nutrition_source.jsonl" --format jsonl

# 2. Run your preprocessing pipeline on nutrition_source.jsonl

# 3. Push processed data
uv run scripts/dataset_manager.py init --repo_id "{your-username}/nutrition-training"
uv run scripts/dataset_manager.py add_rows \
  --repo_id "{your-username}/nutrition-training" \
  --template qa \
  --rows_json "$(cat processed_data.json)"
```

---

## Troubleshooting

| Error | Likely Cause | Fix |
|-------|-------------|-----|
| `401 Unauthorized` | Missing or invalid `HF_TOKEN` | Re-export with a valid Write-access token |
| `403 Forbidden` | Token lacks repo write permissions | Generate a new token with `write` scope |
| `Repository exists` | `init` on existing repo | Script notifies and continues; safe to proceed |
| `Invalid JSON` | Malformed `--rows_json` | Validate with `echo '...' \| python3 -m json.tool` |
| `KeyError: 'answer'` | Wrong column name for dataset | Run `describe` first to inspect schema |
| `choices[N]` off-by-one | DuckDB arrays are 1-indexed | Use `choices[answer+1]` for 0-indexed fields |
| Network timeout | Transient HF Hub issue | Scripts auto-retry; re-run if persistent |
