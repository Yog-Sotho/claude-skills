# Claude API — Extended Patterns

Full agentic loop, structured outputs, TypeScript multi-turn, and advanced patterns.

---

## Complete Agentic Loop (Python)

Handles multiple tool calls, tool failures, and graceful termination:

```python
import anthropic
from typing import Any

client = anthropic.Anthropic()

tools = [
    {
        "name": "search_docs",
        "description": "Search technical documentation",
        "input_schema": {
            "type": "object",
            "properties": {"query": {"type": "string"}},
            "required": ["query"],
        },
    },
    {
        "name": "run_code",
        "description": "Execute a Python code snippet and return output",
        "input_schema": {
            "type": "object",
            "properties": {"code": {"type": "string"}},
            "required": ["code"],
        },
    },
]

def dispatch_tool(name: str, inputs: dict[str, Any]) -> str:
    """Route tool calls to your implementations."""
    if name == "search_docs":
        return search_docs(inputs["query"])  # your implementation
    elif name == "run_code":
        return run_code(inputs["code"])       # your implementation
    return f"Unknown tool: {name}"

def run_agent(user_message: str, max_iterations: int = 10) -> str:
    messages = [{"role": "user", "content": user_message}]

    for iteration in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages,
        )

        if response.stop_reason == "end_turn":
            # Extract final text response
            for block in response.content:
                if block.type == "text":
                    return block.text
            return ""

        if response.stop_reason != "tool_use":
            # Unexpected stop reason
            break

        # Append Claude's response (contains tool_use blocks)
        messages.append({"role": "assistant", "content": response.content})

        # Execute all tool calls in this response
        tool_results = []
        for block in response.content:
            if block.type != "tool_use":
                continue
            try:
                result = dispatch_tool(block.name, block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": result,
                })
            except Exception as e:
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": f"Error: {e}",
                    "is_error": True,
                })

        messages.append({"role": "user", "content": tool_results})

    raise RuntimeError(f"Agent exceeded {max_iterations} iterations without finishing")
```

---

## Structured Outputs (Python)

Force a specific JSON schema using tools with `tool_choice`:

```python
import json

extraction_tool = {
    "name": "extract_data",
    "description": "Extract structured data from the text",
    "input_schema": {
        "type": "object",
        "properties": {
            "company": {"type": "string"},
            "revenue": {"type": "number", "description": "Annual revenue in USD"},
            "employees": {"type": "integer"},
            "founded": {"type": "integer", "description": "Year founded"},
        },
        "required": ["company", "revenue", "employees", "founded"],
    },
}

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=[extraction_tool],
    tool_choice={"type": "tool", "name": "extract_data"},  # force this tool
    messages=[{
        "role": "user",
        "content": "Extract company info: Acme Corp was founded in 1985, has 2,300 employees, and generated $450M in revenue last year.",
    }],
)

# The response is always the tool call — no need to check stop_reason
for block in response.content:
    if block.type == "tool_use":
        data = block.input  # already a dict matching the schema
        print(json.dumps(data, indent=2))
```

---

## TypeScript: Multi-Turn Conversation

```typescript
import Anthropic from "@anthropic-ai/sdk";
import * as readline from "readline";

const client = new Anthropic();

type Message = { role: "user" | "assistant"; content: string };

async function chat(messages: Message[], userInput: string): Promise<string> {
  messages.push({ role: "user", content: userInput });

  const response = await client.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    messages,
  });

  const assistantText = response.content[0].type === "text"
    ? response.content[0].text
    : "";

  messages.push({ role: "assistant", content: assistantText });
  return assistantText;
}

// Usage
const history: Message[] = [];
const reply1 = await chat(history, "What is a closure?");
const reply2 = await chat(history, "Show me an example in TypeScript");
```

---

## TypeScript: Agentic Loop

```typescript
import Anthropic from "@anthropic-ai/sdk";
import type { Tool, MessageParam } from "@anthropic-ai/sdk/resources";

const client = new Anthropic();

const tools: Tool[] = [
  {
    name: "get_weather",
    description: "Get weather for a city",
    input_schema: {
      type: "object" as const,
      properties: { city: { type: "string" } },
      required: ["city"],
    },
  },
];

async function dispatchTool(name: string, input: Record<string, unknown>): Promise<string> {
  if (name === "get_weather") return `Sunny, 22°C in ${input.city}`;
  throw new Error(`Unknown tool: ${name}`);
}

async function runAgent(userMessage: string): Promise<string> {
  const messages: MessageParam[] = [{ role: "user", content: userMessage }];

  for (let i = 0; i < 10; i++) {
    const response = await client.messages.create({
      model: "claude-sonnet-4-6",
      max_tokens: 2048,
      tools,
      messages,
    });

    if (response.stop_reason === "end_turn") {
      return response.content.find(b => b.type === "text")?.text ?? "";
    }

    messages.push({ role: "assistant", content: response.content });

    const toolResults = await Promise.all(
      response.content
        .filter(b => b.type === "tool_use")
        .map(async (b) => {
          if (b.type !== "tool_use") return null;
          try {
            const result = await dispatchTool(b.name, b.input as Record<string, unknown>);
            return { type: "tool_result" as const, tool_use_id: b.id, content: result };
          } catch (e) {
            return { type: "tool_result" as const, tool_use_id: b.id, content: String(e), is_error: true };
          }
        }),
    );

    messages.push({ role: "user", content: toolResults.filter(Boolean) });
  }

  throw new Error("Agent exceeded maximum iterations");
}
```

---

## Async Streaming with Tool Use (Python)

```python
from anthropic import AsyncAnthropic

client = AsyncAnthropic()

async def stream_with_tools():
    async with client.messages.stream(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        tools=tools,
        messages=[{"role": "user", "content": "Search for Python async patterns"}],
    ) as stream:
        async for text in stream.text_stream:
            print(text, end="", flush=True)

    final = await stream.get_final_message()
    return final
```

---

## Prompt Caching with Conversation History

Cache the system prompt; do not mark individual conversation turns as cached:

```python
# System prompt cached (large, stable, repeated across requests)
# Conversation messages uncached (change every turn)
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system=[
        {
            "type": "text",
            "text": large_system_context,          # e.g., full codebase, documentation
            "cache_control": {"type": "ephemeral"},
        }
    ],
    messages=conversation_history,                 # no cache_control on these
)
```

Minimum cacheable size: 1024 tokens. Cache TTL: 5 minutes by default. Check `usage.cache_read_input_tokens` to confirm hits.
