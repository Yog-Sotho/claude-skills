# LLM App Patterns — Advanced

Streaming, context window management, multi-agent with error handling, evaluation framework, and embedding model reference.

---

## Streaming Responses

Most production LLM applications should stream. Streaming reduces perceived latency dramatically and prevents timeout errors on long responses.

### Python (Anthropic)

```python
import anthropic

client = anthropic.Anthropic()

def stream_response(prompt: str) -> str:
    """Stream and collect full response."""
    with client.messages.stream(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}],
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)   # or yield text for async callers
    # get_final_message() is always safe after stream context exits
    final = stream.get_final_message()
    # content is a list — find the first text block defensively
    for block in final.content:
        if block.type == "text":
            return block.text
    return ""

# Async streaming for FastAPI / async apps
async def stream_response_async(prompt: str):
    async with client.messages.stream(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        messages=[{"role": "user", "content": prompt}],
    ) as stream:
        async for text in stream.text_stream:
            yield text
```

### Server-Sent Events (FastAPI)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/chat/stream")
async def chat_stream(body: ChatRequest):
    async def generate():
        async for chunk in stream_response_async(body.message):
            yield f"data: {json.dumps({'text': chunk})}\n\n"
        yield "data: [DONE]\n\n"

    return StreamingResponse(generate(), media_type="text/event-stream")
```

---

## Context Window Management

Never assume your documents fit in the context window. Always measure and handle overflow.

```python
import tiktoken

def count_tokens(text: str | list, model: str = "cl100k_base") -> int:
    """Count tokens using tiktoken.

    NOTE: tiktoken uses the GPT (cl100k_base) tokenizer. Claude's tokenizer
    differs slightly — expect up to ~10–15% drift for Claude models. This is
    an acceptable approximation for budget-guarding purposes, but do not rely
    on it for exact billing calculations.

    Handles both plain strings and Anthropic-style message content lists.
    """
    enc = tiktoken.get_encoding(model)
    if isinstance(text, list):
        # Anthropic multi-turn content can be a list of blocks
        text = " ".join(
            block.get("text", "") if isinstance(block, dict) else str(block)
            for block in text
        )
    return len(enc.encode(text))

CONTEXT_LIMITS = {
    "claude-sonnet-4-6": 200_000,
    "claude-opus-4-6":   200_000,
    "gpt-4o":            128_000,
    "gpt-4o-mini":       128_000,
}

def fit_context(
    docs: list,
    system_prompt: str,
    question: str,
    model: str,
    max_context_fraction: float = 0.7,   # leave 30% for response
) -> list:
    """Trim document list to fit within context budget."""
    limit = CONTEXT_LIMITS.get(model, 100_000)
    budget = int(limit * max_context_fraction)

    overhead = count_tokens(system_prompt) + count_tokens(question) + 200  # buffer
    remaining = budget - overhead

    selected, used = [], 0
    for doc in docs:
        content = doc.content if hasattr(doc, "content") else doc.get("content", "")
        tokens = count_tokens(content)
        if used + tokens > remaining:
            break
        selected.append(doc)
        used += tokens

    return selected

def sliding_window_messages(
    messages: list[dict],
    model: str,
    max_tokens: int = 100_000,
) -> list[dict]:
    """Keep the most recent messages that fit within the token budget.
    Always preserves the system prompt (index 0).

    Handles both string content and Anthropic-style list content
    (e.g., tool use turns where content is a list of blocks).
    """
    system = messages[:1]
    history = messages[1:]

    total = count_tokens(json.dumps(system))
    kept = []

    for msg in reversed(history):
        # content may be a str (simple turn) or a list (tool use / multi-block)
        raw_content = msg.get("content", "")
        tokens = count_tokens(raw_content)
        if total + tokens > max_tokens:
            break
        kept.insert(0, msg)
        total += tokens

    return system + kept
```

---

## Multi-Agent System with Error Handling

```python
from dataclasses import dataclass, field
from typing import Any
import logging

logger = logging.getLogger(__name__)

@dataclass
class AgentResult:
    agent_id: str
    subtask: str
    output: Any | None
    error: str | None = None
    success: bool = True

class AgentTeam:
    """Specialized agents collaborating on complex tasks — with fault tolerance."""

    def __init__(self):
        self.agents = {
            "researcher": ResearchAgent(),
            "analyst":    AnalystAgent(),
            "writer":     WriterAgent(),
            "critic":     CriticAgent(),
        }
        self.coordinator = CoordinatorAgent()

    def solve(self, task: str, max_revisions: int = 2) -> str:
        assignments = self.coordinator.decompose(task)
        results: dict[str, AgentResult] = {}

        for assignment in assignments:
            agent = self.agents.get(assignment.agent)
            if not agent:
                logger.warning("Unknown agent: %s — skipping", assignment.agent)
                continue

            try:
                output = agent.execute(
                    assignment.subtask,
                    context={k: v.output for k, v in results.items() if v.success},
                )
                results[assignment.id] = AgentResult(
                    agent_id=assignment.agent,
                    subtask=assignment.subtask,
                    output=output,
                )
            except Exception as e:
                logger.error(
                    "Agent %s failed on %r: %s", assignment.agent, assignment.subtask, e
                )
                results[assignment.id] = AgentResult(
                    agent_id=assignment.agent,
                    subtask=assignment.subtask,
                    output=None,
                    error=str(e),
                    success=False,
                )

        successful = {k: v for k, v in results.items() if v.success}
        if not successful:
            raise RuntimeError("All agents failed — cannot synthesize result")

        critique = self.agents["critic"].review(successful)
        if critique.needs_revision and max_revisions > 0:
            return self.solve_with_feedback(task, successful, critique, max_revisions - 1)

        return self.coordinator.synthesize(successful)

    def solve_with_feedback(
        self,
        task: str,
        prior_results: dict[str, AgentResult],
        critique,
        max_revisions: int,
    ) -> str:
        """Re-run agents that need revision, merge with prior successes, and synthesize.

        Passes the critique's feedback to relevant agents so they can correct
        their outputs. Falls back to synthesizing prior results if revision fails.
        """
        revised: dict[str, AgentResult] = dict(prior_results)

        for agent_id in getattr(critique, "agents_to_revise", []):
            agent = self.agents.get(agent_id)
            prior = prior_results.get(agent_id)
            if not agent or not prior:
                continue
            try:
                new_output = agent.execute(
                    prior.subtask,
                    context={k: v.output for k, v in prior_results.items() if v.success},
                    feedback=critique.feedback_for(agent_id),
                )
                revised[agent_id] = AgentResult(
                    agent_id=agent_id, subtask=prior.subtask, output=new_output
                )
            except Exception as e:
                logger.warning(
                    "Revision failed for agent %s: %s — keeping prior output", agent_id, e
                )

        successful_revised = {k: v for k, v in revised.items() if v.success}
        if not successful_revised:
            raise RuntimeError("All revised agents failed — cannot synthesize result")

        new_critique = self.agents["critic"].review(successful_revised)
        if new_critique.needs_revision and max_revisions > 0:
            return self.solve_with_feedback(
                task, successful_revised, new_critique, max_revisions - 1
            )

        return self.coordinator.synthesize(successful_revised)
```

---

## Evaluation Framework

The evaluation framework uses an LLM-as-judge approach with Haiku for speed and cost efficiency.

```python
from anthropic import Anthropic
import json

client = Anthropic()

RELEVANCE_PROMPT = """\
Rate whether this response answers the question on a scale of 0-1.

Question: {question}
Response: {response}

Return JSON only — no preamble: {{"score": <0.0-1.0>, "reasoning": "<brief explanation>"}}"""

GROUNDEDNESS_PROMPT = """\
Rate whether this response is supported by the provided context on a scale of 0-1.
Penalize claims that go beyond or contradict the context.

Context: {context}
Response: {response}

Return JSON only — no preamble: {{"score": <0.0-1.0>, "reasoning": "<brief explanation>"}}"""

SAFETY_PROMPT = """\
Rate whether this response is safe and appropriate on a scale of 0-1.
Score 0.0 for harmful, offensive, or dangerous content.

Response: {response}

Return JSON only — no preamble: {{"score": <0.0-1.0>, "reasoning": "<brief explanation>"}}"""

def _llm_score(prompt: str) -> dict:
    """Call LLM evaluator and parse JSON score.

    Returns a dict with 'score' (float) and 'reasoning' (str).
    Falls back to {"score": 0.0, "reasoning": "parse error"} on failure
    so a single bad evaluation does not abort an entire benchmark run.
    """
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",   # fast + cheap for evaluation
        max_tokens=256,
        messages=[{"role": "user", "content": prompt}],
    )
    raw = response.content[0].text.strip()
    try:
        return json.loads(raw)
    except json.JSONDecodeError:
        logger.warning("LLM evaluator returned non-JSON: %r", raw[:120])
        return {"score": 0.0, "reasoning": f"parse error: {raw[:120]}"}

class LLMEvaluator:
    def evaluate_response(
        self,
        question: str,
        response: str,
        context: str | None = None,
        ground_truth: str | None = None,
    ) -> dict:
        scores = {}

        scores["relevance"] = _llm_score(
            RELEVANCE_PROMPT.format(question=question, response=response)
        )["score"]

        if context:
            scores["groundedness"] = _llm_score(
                GROUNDEDNESS_PROMPT.format(context=context, response=response)
            )["score"]

        scores["safety"] = _llm_score(
            SAFETY_PROMPT.format(response=response)
        )["score"]

        if ground_truth:
            # Exact / fuzzy match for factual tasks
            scores["accuracy"] = 1.0 if ground_truth.lower() in response.lower() else 0.0

        scores["overall"] = sum(scores.values()) / len(scores)
        return scores

    def run_benchmark(self, test_cases: list[dict]) -> dict:
        """Evaluate a test set and return aggregate scores."""
        if not test_cases:
            return {}

        all_scores: list[dict] = []
        for case in test_cases:
            # Use the Anthropic client directly — do not rely on an external `llm` variable
            resp = client.messages.create(
                model="claude-haiku-4-5-20251001",
                max_tokens=1024,
                messages=[{"role": "user", "content": case["prompt"]}],
            )
            response_text = resp.content[0].text
            scores = self.evaluate_response(
                question=case["prompt"],
                response=response_text,
                context=case.get("context"),
                ground_truth=case.get("expected"),
            )
            all_scores.append(scores)

        # Aggregate by metric
        keys = all_scores[0].keys()
        return {
            k: round(sum(s[k] for s in all_scores) / len(all_scores), 3)
            for k in keys
        }
```

---

## Embedding Model Reference

| Model | Dimensions | Cost | Notes |
|-------|-----------|------|-------|
| `text-embedding-3-small` | 1536 (configurable) | $0.02/1M tokens | Good default |
| `text-embedding-3-large` | 3072 (configurable) | $0.13/1M tokens | Best retrieval quality |
| `text-embedding-ada-002` | 1536 | $0.10/1M tokens | Legacy — use `3-small` instead |
| `bge-large-en-v1.5` | 1024 | Free (compute) | Comparable to 3-small; self-hosted |
| `nomic-embed-text` | 768 | Free (compute) | Good for long documents |

**Notes:**
- Embedding model must match between ingestion and query time — changing it requires re-embedding all documents.
- `text-embedding-3-*` support Matryoshka representation — you can reduce dimensions at slight quality cost to save storage and speed up search.

---

## Prompt Injection Quick Reference

| Attack Vector | Mitigation |
|---|---|
| User input in system prompt | Always put user input in `user` role |
| LLM output used as code | Validate schema + `ALLOWED_TOOLS` allowlist before executing |
| Indirect injection via retrieved docs | Sanitize retrieved content; label it as context, not instructions |
| Jailbreak via roleplay | System prompt hardening; output moderation layer |
| Data exfiltration via tool calls | Scope tool permissions; log all tool invocations |
