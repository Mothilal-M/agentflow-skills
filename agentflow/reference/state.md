---
name: state
description: AgentState fields, the Message class, content blocks, and how state flows between nodes
metadata:
  tags: state, agentstate, message, content-blocks, multimodal
---

`AgentState` is the dictionary-like object that flows between nodes in a graph. Every node receives it, can read from it, and returns updates that are merged in.

## AgentState fields

```python
from agentflow.core.state import AgentState

# What the runtime guarantees:
state.context           # list[Message] - conversation messages so far
state.context_summary   # str | None    - optional rolling summary
state.execution_meta    # internal      - run metadata (don't write to)
```

Custom subclasses can add more fields:

```python
from agentflow.core.state import AgentState

class MyState(AgentState):
    user_id: str
    cart_items: list[str] = []

graph = StateGraph(state=MyState(user_id="u-1"))
```

The graph engine merges your subclass's fields turn-by-turn using reducers (lists append by default; scalars overwrite).

## Returning state updates from a node

A node returns a `dict`; matching keys are merged into state.

```python
async def main_agent(state: AgentState):
    response = await llm_call(...)
    return {"messages": [Message.text_message(response.content, role="assistant")]}
```

Note: returning `{"messages": [...]}` appends to `state.context`. The list reducer handles concatenation.

## The Message class

`Message` is a Pydantic model. Use the factories — don't construct it field-by-field.

```python
from agentflow.core.state import Message
```

### Factories

| Factory | Use |
|---------|-----|
| `Message.text_message(content, role="user")` | Plain text. Most common. |
| `Message.tool_message(content, tool_call_id)` | Result of a tool call. `tool_call_id` is **required** — get it from the DI-injected `tool_call_id` kwarg in your tool. |
| `Message.image_message(...)` | Image content (URL or base64). |
| `Message.multimodal_message(...)` | Mixed content blocks in one message. |
| `Message.from_file(path)` | Auto-detect mime type and create the right message. |

There is **no** `Message.from_text(...)`. Older snippets that use it are wrong; substitute `text_message`.

### Common fields on a Message

```python
m.role            # "user" | "assistant" | "system" | "tool"
m.content         # str | list[ContentBlock]
m.tools_calls     # list[ToolCallBlock] | None  (note plural "tools_calls")
m.tool_call_id    # str | None  (set on tool messages)
```

**Routing gotcha:** the attribute is `tools_calls` (plural), **not** `tool_calls`. Routers commonly check it:

```python
def route(state):
    last = state.context[-1]
    if last.tools_calls:
        return "TOOL"
    return END
```

## Content blocks (multimodal)

`Message.content` can be a string OR a list of `ContentBlock` subclasses for richer payloads:

```python
from agentflow.core.state import (
    TextBlock, ImageBlock, AudioBlock, DocumentBlock, VideoBlock,
    ReasoningBlock, ToolCallBlock, ToolResultBlock,
)
```

Use `Message.multimodal_message(blocks=[...])` to assemble:

```python
m = Message.multimodal_message(
    role="user",
    blocks=[
        TextBlock(text="What's in this picture?"),
        ImageBlock(url="https://example.com/img.png"),
    ],
)
```

## Inspecting state in routing

Routers run after every node. Typical patterns:

```python
def route(state):
    if not state.context:
        return END  # nothing to process

    last = state.context[-1]

    if last.role == "tool":
        return "MAIN"            # tool output -> let the LLM read it
    if last.tools_calls:
        return "TOOL"            # LLM asked for a tool
    return END                    # plain assistant text -> done
```

## See also

- [graph.md](./graph.md) — how state flows through nodes and edges.
- [tools.md](./tools.md) — `Message.tool_message` and DI-injected state.
- [streaming.md](./streaming.md) — observing state updates as deltas.
