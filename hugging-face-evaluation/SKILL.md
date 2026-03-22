---
name: hugging-face-evaluation
description: >-
  Add, import, and manage evaluation results in Hugging Face model cards.
  Supports extracting eval tables from README content, importing benchmark
  scores from Artificial Analysis API, and running custom model evaluations
  with vLLM/lighteval/inspect-ai on HF Jobs or locally. Works with the
  model-index metadata format for leaderboard and Papers with Code integration.
  Always activate when the user mentions HF model card evaluation, benchmark
  scores, model-index YAML, evaluation results, leaderboard submission,
  Artificial Analysis benchmarks, lighteval, inspect-ai, running evals on
  HF Jobs, or adding eval metrics to a model card — even if they don't
  say "skill".
compatibility: "Requires: uv (https://astral.sh/uv), HF_TOKEN env var (Write-access). Optional: AA_API_KEY for Artificial Analysis import."
---

# Hugging Face Evaluation Skill
<!-- v1.4.0 -->

## ⚡ Quick Setup

```bash
# Install uv (if needed)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Required: Write-access HF token
export HF_TOKEN="hf_your_token_here"

# Optional: only needed for Artificial Analysis import
export AA_API_KEY="your_aa_api_key"
# Or use a .env file — loaded automatically if python-dotenv is installed
```

All scripts use PEP 723 inline dependencies — `uv run` auto-installs on first run.

---

## ⚠️ CRITICAL: Check for Existing PRs Before Creating Any New One

```bash
uv run scripts/evaluation_manager.py get-prs --repo-id "{your-username}/model-name"
```

If open PRs exist: **do not create a new one.** Show the user the existing PR URLs and wait for explicit confirmation before proceeding. This prevents duplicate PRs on repos you don't own.

---

## Method Selection

| Goal | Method |
|------|--------|
| Extract eval table already in README | **Method 1** — `extract-readme` |
| Import scores from Artificial Analysis API | **Method 2** — `import-aa` |
| Evaluate a model via HF inference provider | **Method 3** — HF Jobs + inspect-ai |
| Evaluate any HF model on GPU with vLLM | **Method 4** — lighteval or inspect-ai + vLLM |
| Own the repo? | Use `--apply` to push directly |
| Don't own the repo? | Use `--create-pr` (always run `get-prs` first) |

---

## Method 1: Extract from README

Parses markdown evaluation tables in an existing README and converts them to `model-index` YAML.

```bash
# Step 1: Inspect tables — get table numbers and column hints
uv run scripts/evaluation_manager.py inspect-tables --repo-id "{your-username}/model-name"

# Step 2: Preview YAML (prints to stdout by default — review before applying)
uv run scripts/evaluation_manager.py extract-readme \
  --repo-id "{your-username}/model-name" \
  --table 1 \
  [--model-column-index N]           # preferred: index from inspect-tables output
  [--model-name-override "Exact Header"]  # fallback: exact column header text
  [--task-type "text-generation"]    # sets task.type in model-index
  [--dataset-name "Custom Benchmarks"]

# Step 3a: Push directly (if you own the repo)
uv run scripts/evaluation_manager.py extract-readme \
  --repo-id "{your-username}/model-name" --table 1 --apply

# Step 3b: Open a PR (if you don't own the repo — run get-prs first)
uv run scripts/evaluation_manager.py extract-readme \
  --repo-id "other-username/model-name" --table 1 --create-pr
```

**Column matching notes:**
- Prefer `--model-column-index` (integer from inspect-tables output)
- For `--model-name-override`, the value must be the exact column header text
- For transposed tables (models as rows), ensure only one row is extracted
- Matching is fuzzy-normalized: `"OLMo-3-32B"` → tokens `{olmo, 3, 32b}` — exact token match required

---

## Method 2: Import from Artificial Analysis

Fetches benchmark scores from the Artificial Analysis API and adds them to a model card.

```bash
# Preview / dry run
AA_API_KEY="your-key" uv run scripts/evaluation_manager.py import-aa \
  --creator-slug "anthropic" \
  --model-name "claude-sonnet-4" \
  --repo-id "{your-username}/model-name"

# Create a PR (always check get-prs first)
uv run scripts/evaluation_manager.py import-aa \
  --creator-slug "anthropic" \
  --model-name "claude-sonnet-4" \
  --repo-id "{your-username}/model-name" \
  --create-pr
```

---

## Method 3: Run Evaluation on HF Jobs (Inference Providers)

Submits an eval job to HF infrastructure using `inspect-ai`. No GPU required for CPU tasks.

```bash
# CPU task
HF_TOKEN=$HF_TOKEN \
hf jobs uv run scripts/inspect_eval_uv.py \
  --flavor cpu-basic \
  --secret HF_TOKEN=$HF_TOKEN \
  -- --model "meta-llama/Llama-2-7b-hf" --task "mmlu"

# GPU task (A10G)
HF_TOKEN=$HF_TOKEN \
hf jobs uv run scripts/inspect_eval_uv.py \
  --flavor a10g-small \
  --secret HF_TOKEN=$HF_TOKEN \
  -- --model "meta-llama/Llama-2-7b-hf" --task "gsm8k"

# Or use the Python helper
uv run scripts/run_eval_job.py \
  --model "meta-llama/Llama-2-7b-hf" \
  --task "mmlu" \
  --hardware "t4-small"
```

---

## Method 4: Run Custom Model Evaluation with vLLM

For any HF model on local GPU or HF Jobs GPU. Requires `nvidia-smi` and `uv`.

**Before running:** verify `nvidia-smi` shows an available GPU.

### Option A: lighteval (HF's evaluation library)

```bash
# Local GPU
uv run scripts/lighteval_vllm_uv.py \
  --model meta-llama/Llama-3.2-1B \
  --tasks "leaderboard|mmlu|5"

# Multiple tasks
uv run scripts/lighteval_vllm_uv.py \
  --model meta-llama/Llama-3.2-1B \
  --tasks "leaderboard|mmlu|5,leaderboard|gsm8k|5"

# Instruction-tuned model
uv run scripts/lighteval_vllm_uv.py \
  --model meta-llama/Llama-3.2-1B-Instruct \
  --tasks "leaderboard|mmlu|5" --use-chat-template

# accelerate backend (fallback if vLLM not supported)
uv run scripts/lighteval_vllm_uv.py \
  --model meta-llama/Llama-3.2-1B \
  --tasks "leaderboard|mmlu|5" --backend accelerate

# Via HF Jobs
hf jobs uv run scripts/lighteval_vllm_uv.py \
  --flavor a10g-small \
  --secret HF_TOKEN=$HF_TOKEN \
  -- --model meta-llama/Llama-3.2-1B --tasks "leaderboard|mmlu|5"
```

**lighteval task format:** `suite|task|num_fewshot` — e.g. `leaderboard|mmlu|5`, `lighteval|hellaswag|0`, `bigbench|abstract_narrative_understanding|0`.
Full task list: https://github.com/huggingface/lighteval/blob/main/examples/tasks/all_tasks.txt

### Option B: inspect-ai (UK AI Safety Institute framework)

```bash
# Local GPU
uv run scripts/inspect_vllm_uv.py \
  --model meta-llama/Llama-3.2-1B --task mmlu

# HF Transformers backend (for architectures vLLM doesn't support)
uv run scripts/inspect_vllm_uv.py \
  --model meta-llama/Llama-3.2-1B --task mmlu --backend hf

# Multi-GPU
uv run scripts/inspect_vllm_uv.py \
  --model meta-llama/Llama-3.2-70B --task mmlu --tensor-parallel-size 4

# Via HF Jobs
hf jobs uv run scripts/inspect_vllm_uv.py \
  --flavor a10g-small \
  --secret HF_TOKEN=$HF_TOKEN \
  -- --model meta-llama/Llama-3.2-1B --task mmlu
```

**Available inspect-ai tasks:** `mmlu`, `gsm8k`, `hellaswag`, `arc_challenge`, `truthfulqa`, `winogrande`, `humaneval`

### Option C: Helper Script (auto hardware selection)

```bash
uv run scripts/run_vllm_eval_job.py \
  --model meta-llama/Llama-3.2-1B \
  --task "leaderboard|mmlu|5" \
  --framework lighteval   # or: inspect

# Explicit hardware + tensor parallelism
uv run scripts/run_vllm_eval_job.py \
  --model meta-llama/Llama-3.2-70B \
  --task mmlu \
  --framework inspect \
  --hardware a100-large \
  --tensor-parallel-size 4
```

**Hardware guide:**

| Model Size | Recommended |
|-----------|-------------|
| < 3B | `t4-small` |
| 3B – 13B | `a10g-small` |
| 13B – 34B | `a10g-large` |
| 34B+ | `a100-large` |

---

## model-index Format Reference

```yaml
model-index:
  - name: Model Name        # plain text — no markdown formatting
    results:
      - task:
          type: text-generation
        dataset:
          name: Benchmark Dataset
          type: benchmark_type
        metrics:
          - name: MMLU
            type: mmlu
            value: 85.2
        source:
          name: Source Name
          url: https://source-url.com   # URLs only in source.url
```

---

## Common Workflows

**Update your own model from its README:**
```bash
uv run scripts/evaluation_manager.py inspect-tables --repo-id "{your-username}/model"
uv run scripts/evaluation_manager.py extract-readme \
  --repo-id "{your-username}/model" --table 1 --task-type "text-generation"
# Review YAML, then:
uv run scripts/evaluation_manager.py extract-readme \
  --repo-id "{your-username}/model" --table 1 --apply
```

**Submit a PR to someone else's model:**
```bash
# 1. Check PRs first — abort if any exist
uv run scripts/evaluation_manager.py get-prs --repo-id "other-username/model"

# 2. Only if no open PRs:
uv run scripts/evaluation_manager.py extract-readme \
  --repo-id "other-username/model" --table 1 --create-pr
```

**Import Artificial Analysis benchmarks:**
```bash
uv run scripts/evaluation_manager.py get-prs --repo-id "{your-username}/model"
AA_API_KEY=... uv run scripts/evaluation_manager.py import-aa \
  --creator-slug "anthropic" --model-name "claude-sonnet-4" \
  --repo-id "{your-username}/model" --create-pr
```

**Validate / inspect current model-index:**
```bash
uv run scripts/evaluation_manager.py show     --repo-id "{your-username}/model"
uv run scripts/evaluation_manager.py validate --repo-id "{your-username}/model"
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `401 Unauthorized` | Invalid or missing `HF_TOKEN` | Re-export a valid Write-access token |
| `Token does not have write access` | Token scope too narrow | Generate token with `write` repo permission |
| `No evaluation tables found in README` | README has no markdown tables with numeric scores | Verify table exists with `inspect-tables` |
| `Could not find model 'X'` | Name mismatch in transposed table | Run `inspect-tables` to list available model names; use `--model-name-override` with exact text |
| `AA_API_KEY not set` | Missing env var | Export or add to `.env` |
| `Model not found in Artificial Analysis` | Wrong slug/model-name | Verify `creator-slug` and `model-name` against AA API |
| `Payment required for hardware` | No billing on HF account | Add payment method at huggingface.co/settings/billing |
| CUDA OOM / vLLM OOM | Model too large for GPU | Use larger hardware flavor, lower `--gpu-memory-utilization`, or add `--tensor-parallel-size` |
| `Model architecture not supported by vLLM` | Unsupported arch | Use `--backend hf` (inspect-ai) or `--backend accelerate` (lighteval) |
| `Trust remote code required` | Custom model code | Add `--trust-remote-code` (e.g. Phi-2, Qwen) |
| `Chat template not found` | Base model, not instruct | Only use `--use-chat-template` for instruction-tuned models |
