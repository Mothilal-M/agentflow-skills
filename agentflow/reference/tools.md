---
name: tools
description: ToolNode - registering local tools, MCP, LangChain, Composio, and dependency injection
metadata:
  tags: tools, toolnode, mcp, langchain, composio, dependency-injection
---

A `ToolNode` is a graph node that executes tool calls produced by the LLM. It accepts plain Python callables, MCP servers, LangChain tools, and Composio tools — all surfaced to the LLM as one unified tool list.

## Construction

```python
from agentflow.core.graph import ToolNode

tool_node = ToolNode(
    tools=[get_weather, get_stock_price],
)
```

Constructor params (all keyword-only):

| Param | Purpose |
|-------|---------|
| `tools` | Iterable of callables. Each must have type hints + a docstring (used to derive the tool schema). |
| `client` | An MCP `Client` instance (from `fastmcp`). Tools exposed by the MCP server are auto-registered. |
| `composio_adapter` | Adapter for Composio toolsets. |
| `langchain_adapter` | Adapter that registers LangChain `BaseTool` instances. |
| `pass_user_info_to_mcp` | If `True`, forwards `user_id` / `thread_id` from runtime config to MCP calls. |

## Methods

- `add_tool(fn)` — register an extra tool after construction.
- `all_tools()` — async; returns the merged tool list as the LLM sees it (use when wiring tools into a manual `async def` agent that calls the LLM directly).
- `invoke(...)` — graph engine calls this internally; you rarely call it yourself.

## Local Python tools

```python
def get_weather(location: str) -> str:
    """Get weather for a location."""
    return f"Sunny in {location}"
```

Requirements:
- Type-annotated parameters (used to build the JSON schema).
- A docstring (becomes the tool description).
- Either sync or `async def` — both work. Async is preferred for I/O.

## MCP

```python
from fastmcp import Client

mcp = Client({
    "mcpServers": {
        "weather": {"url": "http://127.0.0.1:8000/mcp", "transport": "streamable-http"},
    }
})
tool_node = ToolNode(tools=[], client=mcp)
```

Install the extra: `pip install 10xscale-agentflow[mcp]`. You can mix local tools with MCP — pass both `tools=` and `client=`.

## LangChain

```python
from langchain_community.tools import DuckDuckGoSearchRun
from agentflow.prebuilt.tools.langchain_adapter import from_langchain

search = DuckDuckGoSearchRun()
tool_node = ToolNode(tools=[from_langchain(search)])
```

Install: `pip install 10xscale-agentflow[langchain]`. Any `BaseTool` subclass works.

## Composio

```python
from composio import ComposioToolSet, App
from agentflow.prebuilt.tools.composio_adapter import from_composio

tools = ComposioToolSet().get_tools(apps=[App.GITHUB])
tool_node = ToolNode(tools=[from_composio(t) for t in tools])
```

Install: `pip install 10xscale-agentflow[composio]`. Set `COMPOSIO_API_KEY` and any provider tokens (e.g. `GITHUB_TOKEN`).

## Dependency injection in tools

Tools can accept extra DI-provided kwargs. AgentFlow inspects the signature and only passes what you ask for.

| Injected param | Type | What it is |
|----------------|------|------------|
| `tool_call_id` | `str \| None` | The current tool call's ID. **Required** if you return a `Message.tool_message(...)`. |
| `state` | `AgentState \| None` | Full conversation state. |
| `config` | `dict \| None` | The `config` dict passed to `invoke()` / `astream()`. |

```python
from agentflow.core.state import Message, AgentState

def audit_log(
    action: str,
    tool_call_id: str | None = None,
    state: AgentState | None = None,
) -> Message:
    """Write an audit entry."""
    # ... persist ...
    return Message.tool_message(content=f"Logged {action}", tool_call_id=tool_call_id)
```

## Returning from a tool

A tool may return:
- A plain string → wrapped automatically into a `Message.tool_message(...)`.
- A `Message` (typically `Message.tool_message(content, tool_call_id=...)`) — full control over the message.
- Any JSON-serializable value → stringified.
- Raise an exception → AgentFlow surfaces it as a tool error message; the LLM can retry.

## Parallel execution

When the LLM emits multiple tool calls in one turn, the `ToolNode` runs them concurrently. Make I/O tools `async` so the event loop can interleave them.

## See also

- [agent-class.md](./agent-class.md) — wiring `tool_node` into an `Agent`.
- [state.md](./state.md) — `Message.tool_message(...)` factory and DI-injected `state`.
- [graph.md](./graph.md) — adding `ToolNode` as a graph node.
