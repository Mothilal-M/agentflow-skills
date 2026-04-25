---
name: agentflow
description: Build agents with 10xscale-agentflow - Agent class, StateGraph, ToolNode, streaming, checkpointing
metadata:
  tags: agentflow, agents, react, tools, streaming, checkpointing
---

## When to use

Load this skill whenever the user is building an agent with `10xscale-agentflow` (the `agentflow` Python package). It teaches the 10-30 line patterns that handle ~90% of use cases. For prebuilt workflows (RAG, Router) see the `prebuilt-patterns` skill. For tool adapters (MCP, LangChain, Composio) see `tool-integrations`. For deployment see `production`.

## Installation

```bash
pip install 10xscale-agentflow
# optional extras:
pip install 10xscale-agentflow[pg_checkpoint,mcp,langchain,composio]
```

Set an LLM key:

```bash
export OPENAI_API_KEY=sk-...     # or GEMINI_API_KEY / ANTHROPIC_API_KEY
```

`.env` files are auto-loaded.

## Canonical imports

The most common mistake from older snippets is using `agentflow.graph` / `agentflow.state` / `agentflow.checkpointer` — those paths **do not exist**. Always use:

```python
from agentflow.core.graph import Agent, StateGraph, ToolNode
from agentflow.core.state import AgentState, Message
from agentflow.utils.constants import END
from agentflow.storage.checkpointer import InMemoryCheckpointer
```

For the full list see [./reference/imports.md](./reference/imports.md).

## Minimal tool-calling agent

```python
from agentflow.core.graph import Agent, StateGraph, ToolNode
from agentflow.core.state import AgentState, Message
from agentflow.utils.constants import END


def get_weather(location: str) -> str:
    """Get weather for a location."""
    return f"The weather in {location} is sunny, 72°F"


tool_node = ToolNode(tools=[get_weather])

graph = StateGraph()
graph.add_node("MAIN", Agent(
    model="gemini/gemini-2.5-flash",
    system_prompt=[{"role": "system", "content": "You are a helpful assistant."}],
    tool_node=tool_node,
))
graph.add_node("TOOL", tool_node)


def route(state: AgentState) -> str:
    if state.context and state.context[-1].tools_calls:  # note: tools_calls (plural)
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

That's the full template. From here, look up the focused references below for whichever piece the user wants to extend.

## Reference index

Load the targeted file when working on a specific topic:

- [./reference/imports.md](./reference/imports.md) — Canonical Python import paths and the gotchas the README gets wrong.
- [./reference/graph.md](./reference/graph.md) — `StateGraph` API: `add_node`, `add_edge`, `add_conditional_edges`, `set_entry_point`, `compile`, recursion limits, common topologies.
- [./reference/agent-class.md](./reference/agent-class.md) — `Agent(...)` constructor: `model`, `tool_node`, `output_type`, fallback models, retry config, structured output, tag-gated tools.
- [./reference/tools.md](./reference/tools.md) — `ToolNode`: local Python tools, MCP, LangChain, Composio, dependency injection (`tool_call_id`, `state`, `config`), parallel execution.
- [./reference/state.md](./reference/state.md) — `AgentState` fields, the `Message` class (`text_message`, `tool_message`, `image_message`, multimodal blocks), and routing on `state.context`.
- [./reference/streaming.md](./reference/streaming.md) — `invoke` vs `astream`, event types (`LLM_DELTA`, `NODE_END`, …), SSE / WebSocket forwarding, cancellation.
- [./reference/checkpointing.md](./reference/checkpointing.md) — `InMemoryCheckpointer` for demos, `PgCheckpointer` for production, `thread_id` semantics, custom checkpointers.
- [./reference/models.md](./reference/models.md) — LiteLLM model-string format, provider mapping, env vars, fallback chains, picking a model.

## Common mistakes to avoid

- **`tools_calls` not `tool_calls`.** The attribute on a `Message` is plural.
- **`Message.from_text(...)` doesn't exist.** Use `Message.text_message(...)`.
- **Tool functions need type hints + a docstring.** That's how the JSON schema is generated.
- **`thread_id` is required** in the `config` dict for any `invoke` / `astream` call.
- **`InMemoryCheckpointer` is demo-only** — use `PgCheckpointer` for anything serving multiple workers.
- **Wrong import paths** — see [./reference/imports.md](./reference/imports.md).

## Sibling skills

- `prebuilt-patterns` — `ReactAgent`, `RAGAgent`, `RouterAgent` for ready-made topologies.
- `tool-integrations` — MCP, LangChain, Composio recipes.
- `production` — `agentflow init` / `api` / `build`, FastAPI deploy, observability, human-in-the-loop.
