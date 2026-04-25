---
name: prebuilt-patterns
description: Use prebuilt agent patterns from 10xscale-agentflow (ReactAgent, RAGAgent, RouterAgent)
metadata:
  tags: agentflow, prebuilt, react, rag, router
---

## When to use

Load this skill when the user wants a common agent topology and doesn't want to wire it up by hand. If the user is writing basic `Agent` + `StateGraph` code from scratch, they probably want `agentflow` instead.

The prebuilt classes live in `agentflow.prebuilt`:

```python
from agentflow.prebuilt import ReactAgent, RAGAgent, RouterAgent
```

Each returns a compiled graph. You call `.compile(...)` with your nodes, then `.invoke(...)` / `.astream(...)` like any other app.

## Choosing a pattern

| Pattern | Use when |
|---------|----------|
| `ReactAgent` | LLM + tools in a loop (reason → act → observe). The default starting point for most agents. |
| `RAGAgent` | You have a retriever (vector DB, keyword search, a ToolNode) and want a retrieve → synthesize flow. |
| `RouterAgent` | You have multiple specialized sub-nodes and want an LLM (or rule) to decide which one handles the request. |

When in doubt: start with `ReactAgent`. Promote to `RAGAgent` once retrieval is the core workflow; to `RouterAgent` once you have enough distinct capabilities that one system prompt can't juggle them.

## ReactAgent

```python
from agentflow.prebuilt import ReactAgent
from agentflow.graph import ToolNode
from agentflow.state import Message


def get_weather(location: str) -> str:
    """Get the weather for a city."""
    return f"Sunny and 72°F in {location}"


react = ReactAgent()
app = react.compile(
    model="openai/gpt-4o-mini",
    system_prompt=[{"role": "system", "content": "You are a helpful assistant."}],
    tool_node=ToolNode([get_weather]),
)

result = app.invoke(
    {"messages": [Message.text_message("Weather in Tokyo?")]},
    config={"thread_id": "1"},
)
```

## RAGAgent

Two required nodes: `retriever` (fetches documents/context) and `synthesizer` (produces the answer). The optional `should_continue` condition can loop back to retrieve more.

```python
from agentflow.prebuilt import RAGAgent
from agentflow.graph import ToolNode
from agentflow.state import Message


def search_docs(query: str) -> str:
    """Look up relevant passages from the knowledge base."""
    # Replace with a real vector search, Elasticsearch call, etc.
    return f"Relevant passages for: {query}"


rag = RAGAgent()
app = rag.compile(
    retriever=ToolNode([search_docs]),
    synthesizer_model="openai/gpt-4o-mini",
    system_prompt=[{"role": "system", "content": "Answer using only retrieved context."}],
)

result = app.invoke(
    {"messages": [Message.text_message("How does our refund policy work?")]},
    config={"thread_id": "1"},
)
```

## RouterAgent

A router node picks a route key; each route maps to a node that handles the request and returns to the router (or ends).

```python
from agentflow.prebuilt import RouterAgent
from agentflow.state import AgentState, Message
from agentflow.utils.constants import END


def router_node(state: AgentState):
    """Decide which specialist should answer."""
    last = state.context[-1].content.lower()
    if "refund" in last:
        return {"route": "billing"}
    if "reset" in last:
        return {"route": "auth"}
    return {"route": END}


def billing_node(state): ...
def auth_node(state): ...


router = RouterAgent()
app = router.compile(
    router_node=router_node,
    routes={
        "billing": billing_node,
        "auth": auth_node,
    },
)
```

## Where to go next

- Add tools to any prebuilt by passing a `ToolNode` to the relevant param.
- Wrap a prebuilt with a checkpointer (see `agentflow`) so state persists across invocations.
- To integrate MCP / LangChain / Composio tools into a prebuilt's `ToolNode`, load `tool-integrations`.
- For deploying a prebuilt as an API, load `production`.
