---
name: llm-app-patterns
description: Production-ready patterns for building LLM applications — RAG pipelines, agent architectures, prompt engineering, caching, observability, and LLMOps. Always activate when the user is designing an LLM-powered application, implementing RAG or retrieval, building agents with tools, setting up LLMOps monitoring, choosing between agent architectures, or asking about prompt chaining, caching, evaluation, or streaming LLM responses. Also activate when the user asks about prompt injection, model fallback, cost optimization, or embedding model selection.
---

# LLM Application Patterns

Production patterns for building reliable, observable, and cost-effective LLM applications.

## Workflow

When this skill activates:

1. **Identify the task** — RAG pipeline, agent architecture, prompt engineering, caching, observability, or evaluation.
2. **Use the Architecture Decision Matrix** to choose the right pattern before writing code.
3. **Apply the relevant section** — each section is self-contained.
4. **For complete implementations** of multi-agent systems, evaluation frameworks, streaming, and context window management, see `references/advanced.md`.
5. **Flag prompt injection risks** proactively — user input passed to LLMs without sanitization is a security issue in every LLM app.

---

## Architecture Decision Matrix

Choose the right pattern before building:

| Pattern | Use When | Complexity | Cost |
|---------|----------|-----------|------|
| Simple RAG | FAQ, docs search, known corpus | Low | Low |
| Hybrid RAG | Mixed query types, precision matters | Medium | Medium |
| ReAct Agent | Multi-step tasks, dynamic tool use | Medium | Medium |
| Function Calling | Structured tool dispatch, typed outputs | Low | Low |
| Plan-and-Execute | Long-horizon tasks, known workflow | High | High |
| Multi-Agent | Research, parallelizable sub-tasks | Very High | Very High |

**Default starting point:** Simple RAG or Function Calling. Add complexity only when simpler patterns demonstrably fail.

---

## 1. RAG Pipeline

```
Documents → Chunk → Embed → Store
Query ──────────────────── Retrieve → Rerank → Generate → Response + Citations
```

### 1.1 Chunking

```python
CHUNK_CONFIG = {
    "chunk_size": 512,       # tokens — tune per document type
    "chunk_overlap": 50,     # preserve cross-chunk context
    "separators": ["\n\n", "\n", ". ", " "],
}

# Strategy selection:
# FIXED_SIZE     — simple; may break mid-sentence
# SEMANTIC       — split on paragraphs/sections; better quality
# RECURSIVE      — tries separators in order; good default
# DOCUMENT_AWARE — respects headers, lists, code blocks; best for structured docs
```

### 1.2 Vector Database & Embedding Selection

| Database | Best For | Scale |
|----------|----------|-------|
| pgvector | Existing Postgres infra | Millions |
| Chroma | Development / prototyping | Thousands |
| Weaviate | Self-hosted, multi-modal | Millions |
| Pinecone | Managed production | Billions |

For embedding model options and dimension/cost tradeoffs, see **Embedding Model Reference** in `references/advanced.md`.

```python
EMBEDDING_MODELS = {
    "openai/text-embedding-3-small": {
        "dimensions": 1536,
        "cost": "$0.02/1M tokens",
        "quality": "Good for most use cases",
    },
    "openai/text-embedding-3-large": {
        "dimensions": 3072,
        "cost": "$0.13/1M tokens",
        "quality": "Best for complex queries",
    },
    "local/bge-large-en-v1.5": {
        "dimensions": 1024,
        "cost": "Free (compute only)",
        "quality": "Comparable to OpenAI small",
    },
}
```

### 1.3 Retrieval Strategies

```python
# Basic semantic search
def semantic_search(query: str, top_k: int = 5) -> list[Document]:
    return vector_db.similarity_search(embed(query), top_k=top_k)

# Hybrid search — semantic + keyword (BM25) via Reciprocal Rank Fusion
def hybrid_search(query: str, top_k: int = 5, alpha: float = 0.7) -> list[Document]:
    """alpha=1.0 pure semantic, alpha=0.0 pure BM25, 0.7 good default."""
    semantic = vector_db.similarity_search(embed(query), top_k=top_k * 2)
    keyword  = bm25_index.search(query, top_k=top_k * 2)
    return rrf_merge(semantic, keyword, alpha=alpha)[:top_k]

# Multi-query: generate variations to improve recall
def multi_query_retrieval(query: str) -> list[Document]:
    variations = llm.generate_variations(query, n=3)
    all_docs = [doc for q in variations for doc in semantic_search(q)]
    return deduplicate_by_id(all_docs)
```

### 1.4 Generation with Citations

```python
RAG_PROMPT = """\
Answer the question using ONLY the context below.
If the context is insufficient, say "I don't have enough information."

Context:
{context}

Question: {question}
Answer:"""

def generate_with_rag(question: str) -> dict:
    docs = hybrid_search(question, top_k=5)

    # Respect context window — truncate if needed
    context = build_context(docs, max_tokens=4000)

    response = llm.generate(RAG_PROMPT.format(context=context, question=question))
    return {"answer": response, "sources": [d.metadata for d in docs]}

def build_context(docs: list[Document], max_tokens: int) -> str:
    """Build context string that fits within token budget."""
    parts, total = [], 0
    for doc in docs:
        tokens = count_tokens(doc.content)
        if total + tokens > max_tokens:
            break
        parts.append(doc.content)
        total += tokens
    return "\n\n---\n\n".join(parts)
```

---

## 2. Agent Architectures

### 2.1 Function Calling (Recommended Default)

Structured tool dispatch — the model returns typed tool calls, your code executes them:

```python
TOOLS = [
    {
        "name": "search_web",
        "description": "Search the web for current information",
        "input_schema": {
            "type": "object",
            "properties": {"query": {"type": "string"}},
            "required": ["query"],
        },
    },
]

class FunctionCallingAgent:
    MAX_ITERATIONS = 10   # always guard against infinite loops

    def run(self, question: str) -> str:
        messages = [{"role": "user", "content": question}]

        for _ in range(self.MAX_ITERATIONS):
            response = self.llm.chat(messages=messages, tools=TOOLS, tool_choice="auto")

            # No tool calls — final answer
            if not response.tool_calls:
                return response.content

            # CRITICAL: append the assistant message BEFORE tool results.
            # response.content may be None when tool calls are present — use [] as fallback.
            # Omitting this step causes a 400 error on the next API call.
            messages.append({
                "role": "assistant",
                "content": response.content or [],
            })

            for tool_call in response.tool_calls:
                result = self._execute_tool(tool_call.name, tool_call.arguments)
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": str(result),
                })

        raise RuntimeError(f"Agent exceeded {self.MAX_ITERATIONS} iterations")
```

### 2.2 ReAct Pattern

For models or workflows where native function calling isn't available:

```python
REACT_PROMPT = """\
You are an assistant that uses tools to answer questions.

Available tools:
{tools_description}

Format:
Thought: [reasoning about what to do]
Action: tool_name(arguments)
Observation: [result — filled in by system]
... (repeat as needed)
Thought: I have enough information.
Final Answer: [response]

Question: {question}
"""

class ReActAgent:
    MAX_ITERATIONS = 10

    def run(self, question: str) -> str:
        messages = [
            {"role": "user", "content": REACT_PROMPT.format(
                tools_description=self._format_tools(),
                question=question,
            )}
        ]

        for _ in range(self.MAX_ITERATIONS):
            response = self.llm.chat(messages=messages)
            messages.append({"role": "assistant", "content": response.content})

            if "Final Answer:" in response.content:
                return self._extract_final_answer(response.content)

            action = self._parse_action(response.content)
            if action:
                observation = self._execute_tool(action)
                messages.append({"role": "user", "content": f"Observation: {observation}"})

        raise RuntimeError(f"Agent exceeded {self.MAX_ITERATIONS} iterations")
```

### 2.3 Plan-and-Execute

For complex long-horizon tasks with a predictable step structure:

```python
class PlanAndExecuteAgent:
    def run(self, task: str) -> str:
        plan = self.planner.create_plan(task)   # ["Step 1: ...", "Step 2: ..."]
        results = []

        for i, step in enumerate(plan):
            result = self.executor.execute(step, context=results)
            results.append(result)

            # Replan if intermediate result changes scope
            if self._needs_replan(task, results):
                plan = self.planner.replan(
                    task, completed=results, remaining=plan[i+1:]
                )

        return self.synthesizer.summarize(task, results)
```

### 2.4 Multi-Agent Collaboration

For research or highly parallelizable tasks, consider orchestrating multiple specialized agents.
See `references/advanced.md` for the **full fault-tolerant implementation** with per-agent error handling, partial-result synthesis, and revision loops.

```python
class AgentTeam:
    """Coordination skeleton — see advanced.md for production version."""

    def __init__(self):
        self.agents = {
            "researcher": ResearchAgent(),
            "analyst":    AnalystAgent(),
            "writer":     WriterAgent(),
            "critic":     CriticAgent(),
        }
        self.coordinator = CoordinatorAgent()

    def solve(self, task: str) -> str:
        assignments = self.coordinator.decompose(task)
        results = {}

        for assignment in assignments:
            agent = self.agents.get(assignment.agent)   # .get() — safe on unknown agent
            if agent is None:
                continue
            result = agent.execute(assignment.subtask, context=results)
            results[assignment.id] = result

        critique = self.agents["critic"].review(results)
        if critique.needs_revision:
            return self._revise_with_feedback(task, results, critique)

        return self.coordinator.synthesize(results)
```

---

## 3. Prompt Engineering

### 3.1 Prompt Templates

```python
from string import Formatter

class PromptTemplate:
    def __init__(self, template: str):
        self.template = template
        # Infer variables from template — no manual list needed
        self.variables = {
            fname for _, fname, _, _ in Formatter().parse(template) if fname
        }

    def format(self, **kwargs) -> str:
        missing = self.variables - set(kwargs)
        if missing:
            raise ValueError(f"Missing template variables: {missing}")
        return self.template.format(**kwargs)

    def with_examples(self, examples: list[dict]) -> "PromptTemplate":
        """Prepend few-shot examples to template."""
        shots = "\n\n".join(
            f"Input: {e['input']}\nOutput: {e['output']}" for e in examples
        )
        return PromptTemplate(f"{shots}\n\n{self.template}")
```

### 3.2 Prompt Versioning and A/B Testing

```python
import hashlib
from datetime import datetime, timezone

class PromptRegistry:
    def register(self, name: str, template: str, version: str) -> None:
        self.db.save({
            "name": name,
            "template": template,
            "version": version,
            # datetime.utcnow() is deprecated in 3.12+ — always use timezone-aware form
            "created_at": datetime.now(timezone.utc).isoformat(),
            "metrics": {},
        })

    def get(self, name: str, version: str = "latest") -> str:
        return self.db.get(name, version)

    def ab_test(self, name: str, user_id: str) -> str:
        """Deterministic bucketing — stable across process restarts.
        Never use Python's hash() — it's randomized per process."""
        variants = self.db.get_all_versions(name)
        # md5 is fine here — not used for security, just stable bucketing
        bucket = int(hashlib.md5(user_id.encode()).hexdigest(), 16) % len(variants)
        return variants[bucket]

    def record_outcome(self, prompt_id: str, outcome: dict) -> None:
        self.db.update_metrics(prompt_id, outcome)
```

### 3.3 Prompt Chaining

```python
class PromptChain:
    def __init__(self, steps: list[dict]):
        """
        steps: [{"name": str, "prompt": str, "output_key": str,
                 "parser": Callable | None}]
        """
        self.steps = steps

    def run(self, initial_input: str) -> dict:
        context: dict = {"input": initial_input}
        results = []

        for step in self.steps:
            prompt = step["prompt"].format(**context)
            output = llm.generate(prompt)

            # Apply output parser if provided
            if step.get("parser"):
                output = step["parser"](output)

            context[step["output_key"]] = output
            results.append({"step": step["name"], "output": output})

        return {
            "final_output": context[self.steps[-1]["output_key"]],
            "intermediate_results": results,
        }
```

---

## 4. Production Patterns

### 4.1 Caching

Cache deterministic outputs (temperature=0). Include model and all parameters in the key:

```python
import hashlib, json
from dataclasses import dataclass

@dataclass
class CacheStats:
    hits: int = 0
    misses: int = 0

class LLMCache:
    def __init__(self, redis_client, ttl_seconds: int = 3600):
        self.redis = redis_client
        self.ttl = ttl_seconds
        self.stats = CacheStats()

    def _cache_key(self, prompt: str, model: str, **kwargs) -> str:
        content = json.dumps({"model": model, "prompt": prompt, **kwargs}, sort_keys=True)
        return f"llm:{hashlib.sha256(content.encode()).hexdigest()}"

    def get_or_generate(self, prompt: str, model: str, **kwargs) -> str:
        # Only cache deterministic outputs
        if kwargs.get("temperature", 1.0) != 0:
            return llm.generate(prompt, model=model, **kwargs)

        key = self._cache_key(prompt, model, **kwargs)
        cached = self.redis.get(key)
        if cached:
            self.stats.hits += 1
            return cached.decode()

        self.stats.misses += 1
        response = llm.generate(prompt, model=model, **kwargs)
        self.redis.setex(key, self.ttl, response)
        return response
```

### 4.2 Rate Limiting and Retry

```python
import threading, time
from tenacity import retry, wait_exponential, stop_after_attempt, retry_if_exception

class RateLimiter:
    """Thread-safe sliding-window rate limiter."""

    def __init__(self, requests_per_minute: int):
        self.rpm = requests_per_minute
        self.timestamps: list[float] = []
        self._lock = threading.Lock()

    def acquire(self) -> None:
        with self._lock:   # thread-safe — prevents race conditions
            now = time.time()
            self.timestamps = [t for t in self.timestamps if now - t < 60]
            if len(self.timestamps) >= self.rpm:
                sleep_for = 60 - (now - self.timestamps[0])
                time.sleep(max(0, sleep_for))
            self.timestamps.append(time.time())

def _is_retryable(exc: Exception) -> bool:
    """Only retry transient errors — not auth failures or bad requests."""
    if isinstance(exc, RateLimitError):
        return True
    if isinstance(exc, APIError):
        return getattr(exc, "status_code", 0) >= 500  # 5xx only
    return False

@retry(
    wait=wait_exponential(multiplier=1, min=4, max=60),
    stop=stop_after_attempt(5),
    retry=retry_if_exception(_is_retryable),  # don't retry 400/401/403
)
def call_llm_with_retry(prompt: str) -> str:
    return llm.generate(prompt)
```

### 4.3 Model Fallback

```python
class LLMWithFallback:
    def __init__(self, primary: str, fallbacks: list[str]):
        self.models = [primary] + fallbacks

    def generate(self, prompt: str, **kwargs) -> str:
        last_error: Exception | None = None
        for model in self.models:
            try:
                return llm.generate(prompt, model=model, **kwargs)
            except (RateLimitError, APIError) as e:
                if isinstance(e, APIError) and getattr(e, "status_code", 0) < 500:
                    raise   # 4xx errors won't be fixed by switching models
                logger.warning("Model %s failed: %s", model, e)
                last_error = e
        raise RuntimeError("All models failed") from last_error
```

---

## 5. LLMOps and Observability

### 5.1 Metrics

Track these at minimum:

```python
LLM_METRICS = {
    # Performance
    "latency_p50_ms":       "50th percentile end-to-end latency",
    "latency_p99_ms":       "99th percentile — catches tail latency",
    "time_to_first_token":  "Streaming UX metric",

    # Quality
    "user_satisfaction":    "Thumbs up / thumbs down ratio",
    "task_completion_rate": "% tasks reaching a final answer",
    "hallucination_rate":   "% responses flagged by evaluator",

    # Cost
    "cost_usd_per_request": "Average $ per call",
    "tokens_per_request":   "Average input + output tokens",
    "cache_hit_rate":       "% served from cache — directly reduces cost",

    # Reliability
    "error_rate":           "% failed requests",
    "retry_rate":           "% requests needing at least one retry",
    "fallback_rate":        "% requests that hit a fallback model",
}
```

### 5.2 Structured Logging and Tracing

```python
import json, logging
from datetime import datetime, timezone
from opentelemetry import trace

logger = logging.getLogger(__name__)
tracer = trace.get_tracer(__name__)

class LLMLogger:
    def log_request(self, request_id: str, model: str, prompt: str,
                    prompt_tokens: int, user_id: str | None = None) -> None:
        logger.info(json.dumps({
            "event": "llm.request",
            "request_id": request_id,
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "model": model,
            "prompt_preview": prompt[:200],   # never log full prompt in prod
            "prompt_tokens": prompt_tokens,
            "user_id": user_id,
        }))

    def log_response(self, request_id: str, completion_tokens: int,
                     total_tokens: int, latency_ms: float,
                     finish_reason: str, cost_usd: float) -> None:
        logger.info(json.dumps({
            "event": "llm.response",
            "request_id": request_id,
            "completion_tokens": completion_tokens,
            "total_tokens": total_tokens,
            "latency_ms": latency_ms,
            "finish_reason": finish_reason,
            "cost_usd": cost_usd,
        }))

@tracer.start_as_current_span("llm_call")
def call_llm_traced(prompt: str) -> str:
    span = trace.get_current_span()
    span.set_attribute("llm.prompt.tokens", count_tokens(prompt))
    response = llm.generate(prompt)
    span.set_attribute("llm.response.tokens", response.usage.total_tokens)
    span.set_attribute("llm.finish_reason", response.finish_reason)
    return response.content
```

### 5.3 Evaluation Framework (Skeleton)

Implement the scoring methods below, or load the full LLM-as-judge implementation
from `references/advanced.md`:

```python
class LLMEvaluator:
    """Evaluate LLM outputs for quality."""

    def evaluate_response(self,
                          question: str,
                          response: str,
                          context: str | None = None,
                          ground_truth: str | None = None) -> dict:
        scores = {}

        # Relevance: Does it answer the question?
        scores["relevance"] = self._score_relevance(question, response)

        # Coherence: Is it well-structured?
        scores["coherence"] = self._score_coherence(response)

        # Groundedness: Is it based on provided context (if any)?
        if context:
            scores["groundedness"] = self._score_groundedness(response, context)

        # Accuracy: Compare with ground truth
        if ground_truth:
            scores["accuracy"] = self._score_accuracy(response, ground_truth)

        # Safety: Check for harmful content
        scores["safety"] = self._score_safety(response)

        scores["overall"] = sum(scores.values()) / len(scores)
        return scores

    def run_benchmark(self, test_cases: list[dict]) -> dict:
        """Run evaluation on a test set."""
        results = []
        for case in test_cases:
            response = llm.generate(case["prompt"])
            scores = self.evaluate_response(
                question=case["prompt"],
                response=response,
                context=case.get("context"),
                ground_truth=case.get("expected"),
            )
            results.append(scores)
        return self._aggregate_scores(results)

    # Implement with custom logic or LLM-as-a-judge (see references/advanced.md)
    def _score_relevance(self, question: str, response: str) -> float: ...
    def _score_coherence(self, response: str) -> float: ...
    def _score_groundedness(self, response: str, context: str) -> float: ...
    def _score_accuracy(self, response: str, ground_truth: str) -> float: ...
    def _score_safety(self, response: str) -> float: ...
    def _aggregate_scores(self, results: list[dict]) -> dict: ...
```

---

## 6. Security — Prompt Injection

Every LLM application that passes user input to a model is vulnerable to prompt injection. Treat it as seriously as SQL injection.

```python
# ❌ DANGEROUS: user content interpolated into system prompt
system = f"You are a helpful assistant. The user is {user_name}. Always be helpful."

# ✅ SAFE: user input isolated in its own message turn
messages = [
    {"role": "system", "content": "You are a helpful assistant. Be concise."},
    {"role": "user", "content": user_input},   # never in system prompt
]

# ✅ Validate and bound LLM outputs before acting on them
from pydantic import BaseModel

class AgentAction(BaseModel):
    tool: str
    arguments: dict

def parse_tool_call(raw: str) -> AgentAction:
    """Validate LLM output before executing — never eval() or exec() raw output."""
    data = json.loads(raw)
    action = AgentAction.model_validate(data)
    if action.tool not in ALLOWED_TOOLS:
        raise ValueError(f"Unknown tool: {action.tool}")
    return action
```

**Rules:**

- User input always goes in `user` role messages — never interpolated into system prompt
- Validate and schema-check LLM outputs before acting on them
- Never `eval()` or `exec()` LLM-generated code without sandboxing
- Scope tool access to the minimum needed for the task
- Log all prompts and responses for audit

For the full Prompt Injection Quick Reference table, see `references/advanced.md`.

---

For streaming responses, context window management, the full fault-tolerant multi-agent system, and the LLM-as-judge evaluation framework with implementations, see `references/advanced.md`.
