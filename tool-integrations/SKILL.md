---
name: tool-integrations
description: Plug MCP, LangChain, and Composio tools into 10xscale-agentflow agents
metadata:
  tags: agentflow, tools, mcp, langchain, composio
---

## When to use

Load this when the user is connecting external tools into an AgentFlow agent — MCP servers, LangChain's tool catalog, or Composio integrations. For plain local Python tools, use `agent-basics`.

## Install the extras you need

```bash
pip install 10xscale-agentflow[mcp]        # MCP tools
pip install 10xscale-agentflow[langchain]  # LangChain tools
pip install 10xscale-agentflow[composio]   # Composio tools
# combined:
pip install 10xscale-agentflow[mcp,langchain,composio]
```

## Pattern: `ToolNode` is the integration point

Every tool source funnels through `agentflow.graph.ToolNode`. You pass local Python callables, an MCP client, LangChain tools, or a Composio registry — the node normalizes them so the LLM sees a uniform tool schema.

## MCP (Model Context Protocol)

MCP lets you connect to any MCP server — local or remote — that exposes tools. Use the `fastmcp` client.

```python
from fastmcp import Client
from agentflow.graph import ToolNode

mcp_config = {
    "mcpServers": {
        "weather": {
            "url": "http://127.0.0.1:8000/mcp",
            "transport": "streamable-http",
        },
    }
}
mcp_client = Client(mcp_config)

tool_node = ToolNode(functions=[], client=mcp_client)
```

Wire `tool_node` into your graph the same way as any other `ToolNode`. You can also mix local functions with MCP by passing both:

```python
tool_node = ToolNode(functions=[my_local_tool], client=mcp_client)
```

See [`examples/react-mcp/`](https://github.com/10xhub/agentflow/tree/main/examples/react-mcp) in the 10xscale-agentflow repo for a runnable server + client.

## LangChain tools

Use LangChain's community tools directly. AgentFlow adapts them into native tool calls.

```python
from langchain_community.tools import DuckDuckGoSearchRun
from agentflow.graph import ToolNode
from agentflow.prebuilt.tools.langchain_adapter import from_langchain

search = DuckDuckGoSearchRun()
tool_node = ToolNode([from_langchain(search)])
```

Any LangChain `BaseTool` works — including your own subclasses.

## Composio tools (parallel execution)

Composio gives you pre-built integrations (Gmail, Slack, GitHub, Notion, etc.). AgentFlow runs Composio tool calls in parallel out of the box.

```python
from composio import ComposioToolSet, App
from agentflow.graph import ToolNode
from agentflow.prebuilt.tools.composio_adapter import from_composio

tools = ComposioToolSet().get_tools(apps=[App.GITHUB])
tool_node = ToolNode([from_composio(t) for t in tools])
```

Set the relevant API keys in your environment (e.g. `COMPOSIO_API_KEY`, `GITHUB_TOKEN`) before running.

## Dependency injection in tools

AgentFlow tools can request DI-provided kwargs via their signature. Commonly injected values:

- `tool_call_id: str | None` — the current call's ID, required if you return a `Message.tool_message(...)`.
- `state: AgentState | None` — full conversation state.

```python
from agentflow.state import AgentState, Message


def audit_log(action: str, tool_call_id: str | None = None, state: AgentState | None = None) -> Message:
    """Write an audit entry."""
    # ... persist to DB ...
    return Message.tool_message(content=f"Logged: {action}", tool_call_id=tool_call_id)
```

## Combining multiple sources

You can register a single `ToolNode` that blends local, MCP, LangChain, and Composio tools:

```python
tool_node = ToolNode(
    functions=[audit_log, *[from_langchain(t) for t in lc_tools]],
    client=mcp_client,
)
```

The LLM sees one unified tool list.

## Common mistakes

- **Forgetting the extra.** `pip install 10xscale-agentflow[mcp]` is required before importing any MCP code.
- **Blocking the event loop.** If your tool does HTTP I/O, make it `async def` so the graph can run calls in parallel.
- **Swallowing tool errors silently.** Let exceptions bubble — AgentFlow surfaces them as tool messages so the LLM can retry.

## Where to go next

- Want an out-of-the-box RAG or router on top of these tools? Load `prebuilt-patterns`.
- Deploying as an API server? Load `production`.
