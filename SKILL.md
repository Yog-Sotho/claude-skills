---
name: claude-api
description: Anthropic Claude API patterns for Python and TypeScript. Covers the Messages API, streaming, tool use, vision, PDF input, extended thinking, prompt caching, batches, token counting, structured outputs, and agentic loops. Always activate when the user is building with the Claude API or Anthropic SDKs, code imports `anthropic` or `@anthropic-ai/sdk`, or the user asks about tool use, streaming, cost optimization, model selection, or any Anthropic API pattern — even if they don't use the word "API".
---

# Claude API

Production patterns for the Anthropic Claude API and SDKs.

## Workflow

When this skill activates:

1. **Identify the feature** — basic message, streaming, tool use, vision/PDF, thinking, caching, batches, or agentic loop.
2. **Use the correct model ID** from the table below — never use aliases in production.
3. **Read `references/patterns.md`** for complete agentic loop examples, TypeScript implementations, and structured outputs.
4. **Default to async** (`AsyncAnthropic`) for any web or async application context.
5. **Flag stale model IDs proactively** if spotted in user code — this is the most common mistake.

---

## Model Selection

| Model | API ID | Context | Best For |
|-------|--------|---------|---------|
| Opus 4.6 | `claude-opus-4-6` | 200K | Complex reasoning, long-horizon agents, research |
| Sonnet 4.6 | `claude-sonnet-4-6` | 200K | Coding, most production tasks — best default |
| Haiku 4.5 | `claude-haiku-4-5-20251001` | 200K | High-volume, cost-sensitive, classification |

**Rules:**
- Default to `claude-sonnet-4-6` unless the task clearly needs Opus depth or Haiku speed
- Always use pinned snapshot IDs (e.g., `claude-sonnet-4-6`) in production — aliases (`claude-sonnet-latest`) can change under you
- Check current IDs at: `GET https://api.anthropic.com/v1/models`

---

## Python SDK

### Installation

```bash
pip install anthropic
```

### Basic Message

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="You are a senior Python developer. Be concise.",
    messages=[{"role": "user", "content": "Review this function"}],
)
print(message.content[0].text)
```

### Async Client (use this in FastAPI / any async app)

```python
from anthropic import AsyncAnthropic

client = AsyncAnthropic()  # reads ANTHROPIC_API_KEY from env

async def ask(prompt: str) -> str:
    message = await client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}],
    )
    return message.content[0].text
```

### Streaming

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a haiku about coding"}],
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

# Or collect the final result
final_message = stream.get_final_message()
```

### Multi-Turn Conversations

The API is stateless — you maintain history by appending messages:

```python
messages = []

# Turn 1
messages.append({"role": "user", "content": "What is a closure in Python?"})
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=messages,
)
messages.append({"role": "assistant", "content": response.content})

# Turn 2 — Claude has full context
messages.append({"role": "user", "content": "Show me an example"})
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=messages,
)
```

---

## TypeScript SDK

### Installation

```bash
npm install @anthropic-ai/sdk
```

### Basic Message

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic(); // reads ANTHROPIC_API_KEY from env

const message = await client.messages.create({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Explain async/await in TypeScript" }],
});
console.log(message.content[0].text);
```

### Streaming

```typescript
// High-level helper — prefer this over the raw event loop
const stream = client.messages.stream({
  model: "claude-sonnet-4-6",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Write a haiku" }],
});

for await (const text of stream.text_stream) {
  process.stdout.write(text);
}

const finalMessage = await stream.finalMessage();
```

---

## Tool Use

Define tools and handle the response loop:

```python
tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
            },
            "required": ["location"],
        },
    }
]

messages = [{"role": "user", "content": "What's the weather in SF?"}]

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=messages,
)

# If Claude wants to use a tool
if response.stop_reason == "tool_use":
    # Append Claude's full response (including tool_use block)
    messages.append({"role": "assistant", "content": response.content})

    # Execute each tool call and collect results
    tool_results = []
    for block in response.content:
        if block.type == "tool_use":
            result = get_weather(**block.input)  # your implementation
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": str(result),
            })

    # Send results back
    messages.append({"role": "user", "content": tool_results})
    final = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        tools=tools,
        messages=messages,
    )
    print(final.content[0].text)
```

For a complete multi-tool agentic loop, see `references/patterns.md`.

---

## Vision and PDF

### Image — URL (simpler for web content)

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "url", "url": "https://example.com/diagram.png"}},
            {"type": "text", "text": "Describe this diagram"},
        ],
    }],
)
```

### Image — Base64 (for local files)

```python
import base64

with open("diagram.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": image_data}},
            {"type": "text", "text": "Describe this diagram"},
        ],
    }],
)
```

### PDF Input

```python
with open("report.pdf", "rb") as f:
    pdf_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    messages=[{
        "role": "user",
        "content": [
            {"type": "document", "source": {"type": "base64", "media_type": "application/pdf", "data": pdf_data}},
            {"type": "text", "text": "Summarize the key findings"},
        ],
    }],
)
```

---

## Extended Thinking

For complex reasoning — `max_tokens` must exceed `budget_tokens` to leave room for the actual response:

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=20000,          # must be > budget_tokens
    thinking={
        "type": "enabled",
        "budget_tokens": 10000,  # reasoning budget; remainder is for output
    },
    messages=[{"role": "user", "content": "Solve this complex optimization problem..."}],
)

for block in message.content:
    if block.type == "thinking":
        print(f"[Thinking]\n{block.thinking}\n")
    elif block.type == "text":
        print(f"[Answer]\n{block.text}")
```

---

## Prompt Caching

Cache system prompts or large context to reduce costs by up to 90% on repeated requests:

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": large_context,               # must be ≥ 1024 tokens to cache
            "cache_control": {"type": "ephemeral"},
        }
    ],
    messages=[{"role": "user", "content": "Question about the context"}],
)

# Inspect cache hits on the response
usage = message.usage
print(f"Cache read tokens:     {usage.cache_read_input_tokens}")
print(f"Cache creation tokens: {usage.cache_creation_input_tokens}")
print(f"Uncached input tokens: {usage.input_tokens}")
```

---

## Token Counting

Count tokens before sending to avoid surprises — free to call, subject to rate limits:

```python
# Pre-flight check before sending a large payload
token_response = client.messages.count_tokens(
    model="claude-sonnet-4-6",
    system="You are a helpful assistant",
    messages=[{"role": "user", "content": large_input}],
)
print(f"Estimated input tokens: {token_response.input_tokens}")

# Supports tools, images, PDFs — same structure as messages.create()
```

---

## Batches API

Process large volumes asynchronously at 50% cost reduction — for non-time-sensitive jobs:

```python
import time

# Submit batch
batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"item-{i}",
            "params": {
                "model": "claude-sonnet-4-6",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": prompt}],
            },
        }
        for i, prompt in enumerate(prompts)
    ]
)

# Poll with timeout
max_wait_seconds = 3600  # 1 hour
deadline = time.time() + max_wait_seconds

while time.time() < deadline:
    status = client.messages.batches.retrieve(batch.id)
    if status.processing_status == "ended":
        break
    print(f"[{status.processing_status}] {status.request_counts}")
    time.sleep(30)
else:
    raise TimeoutError(f"Batch {batch.id} did not complete within {max_wait_seconds}s")

# Retrieve results
for result in client.messages.batches.results(batch.id):
    if result.result.type == "succeeded":
        print(result.custom_id, result.result.message.content[0].text)
    else:
        print(f"Failed: {result.custom_id} — {result.result.error}")
```

---

## Error Handling

The SDK auto-retries 2× with exponential backoff for: connection errors, 429 Rate Limit, 409 Conflict, and ≥500 server errors. You don't need to implement this yourself.

```python
from anthropic import (
    Anthropic,
    APIConnectionError,   # network-level failure (retried automatically)
    AuthenticationError,  # 401 — bad API key
    RateLimitError,       # 429 — retried automatically; fires if retries exhausted
    APIStatusError,       # all other non-2xx responses
)

client = Anthropic(
    max_retries=3,       # override default of 2 (set to 0 to disable retries)
    timeout=30.0,        # seconds; default is 600
)

try:
    message = client.messages.create(...)
except AuthenticationError:
    # Bad or expired API key — fix before retrying
    raise
except RateLimitError as e:
    # Retries exhausted — back off and try again later
    print(f"Rate limit hit: {e.response.headers.get('retry-after')} seconds")
except APIConnectionError as e:
    # Retries exhausted — network is unavailable
    print(f"Connection failed: {e.__cause__}")
except APIStatusError as e:
    # Non-retriable 4xx error
    print(f"API error {e.status_code}: {e.message}")
```

---

## Cost Optimization

| Strategy | Savings | Notes |
|----------|---------|-------|
| Prompt caching | Up to 90% on cached tokens | Cache anything ≥ 1024 tokens repeated across requests |
| Batches API | 50% on all tokens | For bulk processing that can wait up to 24h |
| Haiku instead of Sonnet | ~75% | Suitable for classification, extraction, simple Q&A |
| Token counting preflight | Variable | Catch oversized prompts before they fail or charge |
| Shorter `max_tokens` | Variable | Set conservatively when output length is predictable |
| Streaming | No savings | Better UX only — same cost as non-streaming |

---

## Environment Setup

```bash
# Required
export ANTHROPIC_API_KEY="sk-ant-..."

# Verify with a quick test
python -c "import anthropic; print(anthropic.Anthropic().models.list().data[0].id)"
```

Never hardcode API keys. Use environment variables or a secrets manager (`python-dotenv`, AWS Secrets Manager, etc.).
