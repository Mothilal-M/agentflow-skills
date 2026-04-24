---
name: agent-basics
description: Build your first agent with 10xscale-agentflow (Agent class, StateGraph, ToolNode, streaming, checkpointing)
metadata:
  tags: agentflow, agents, react, tools, streaming, checkpointing
---

## When to use

Load this skill whenever the user is building an agent with `10xscale-agentflow` (the `agentflow` Python package). It covers the 10-30 line patterns that handle ~90% of use cases. For prebuilt workflows (RAG, Router) see `prebuilt-patterns`. For tool adapters (MCP, LangChain, Composio) see `tool-integrations`. For deployment see `production`.

## Installation

```bash
pip install 10xscale-agentflow
# or: uv pip install 10xscale-agentflow
```

Set an LLM key:

```bash
export OPENAI_API_KEY=sk-...
# or GEMINI_API_KEY, ANTHROPIC_API_KEY
```

`.env` files are auto-loaded.

## Minimal tool-calling agent (the Agent class)

This is the preferred starting point. Under 30 lines:

```python
from agentflow.graph import Agent, StateGraph, ToolNode
from agentflow.state import AgentState, Message
from agentflow.utils.constants import END


def get_weather(location: str) -> str:
    """Get weather for a location."""
    return f"The weather in {location} is sunny, 72°F"


graph = StateGraph()
graph.add_node("MAIN", Agent(
    model="gemini/gemini-2.5-flash",
    system_prompt=[{"role": "system", "content": "You are a helpful assistant."}],
    tool_node_name="TOOL",
))
graph.add_node("TOOL", ToolNode([get_weather]))


def route(state: AgentState) -> str:
    if state.context and state.context[-1].tools_calls:
        return "TOOL"
    return END


graph.add_conditional_edges("MAIN", route, {"TOOL": "TOOL", END: END})
graph.add_edge("TOOL", "MAIN")
graph.set_entry_point("MAIN")

app = graph.compile()
result = app.invoke(
    {"messages": [Message.text_message("What's the weather in NYC?")]},
    config={"thread_id": "1"},
)
for msg in result["messages"]:
    print(f"{msg.role}: {msg.content}")
```

Key pieces:

- **`Agent(...)`** — an LLM node that handles message conversion and tool-call orchestration automatically. Give it a `model` string (LiteLLM-style: `"gemini/..."`, `"openai/..."`, `"anthropic/..."`), a `system_prompt`, and the name of the tool node.
- **`ToolNode([fn1, fn2, ...])`** — turns plain Python functions into callable tools. Each function's signature/docstring becomes the tool schema.
- **`StateGraph`** — container for nodes + edges. Call `.compile()` to get a runnable app.
- **Routing function** — inspects the last message; return `"TOOL"` if the LLM requested a tool call, otherwise `END`.

## Streaming

Use `astream()` on the compiled graph to get deltas as they arrive:

```python
import asyncio

async def run():
    async for event in app.astream(
        {"messages": [Message.text_message("Plan my week")]},
        config={"thread_id": "1"},
    ):
        print(event)

asyncio.run(run())
```

## Checkpointing (persistent state)

Pass a checkpointer to `.compile()` so conversations resume across invocations:

```python
from agentflow.checkpointer import InMemoryCheckpointer

app = graph.compile(checkpointer=InMemoryCheckpointer())

# First call
app.invoke({"messages": [Message.text_message("Remember my name is Alex.")]},
           config={"thread_id": "user-42"})

# Second call — same thread_id, context carries over
app.invoke({"messages": [Message.text_message("What's my name?")]},
           config={"thread_id": "user-42"})
```

`InMemoryCheckpointer` is only for demos. For production use the PostgreSQL + Redis checkpointer:

```bash
pip install 10xscale-agentflow[pg_checkpoint]
```

## Common mistakes to avoid

- **Don't forget `set_entry_point`.** Every graph needs exactly one.
- **Routing returns the edge key, not the node name.** The key maps through the dict you pass to `add_conditional_edges`.
- **`state.context[-1].tools_calls`** — note the plural `tools_calls`, not `tool_calls`.
- **Tool functions must have type hints and a docstring.** That's how the tool schema is generated.

## Typical follow-ups

- **"How do I add multiple tools?"** → Pass a longer list to `ToolNode([...])`.
- **"How do I stream to a WebSocket?"** → Use `astream()` and forward each event.
- **"How do I pause for human approval?"** → See the `production` skill (interrupts + checkpointer resume).
- **"I need RAG / multi-agent routing."** → Load the `prebuilt-patterns` skill.
- **"Integrate with MCP / LangChain / Composio."** → Load the `tool-integrations` skill.
