---
name: graph
description: StateGraph internals - nodes, edges, conditional routing, compile, recursion limits
metadata:
  tags: graph, stategraph, nodes, edges, routing
---

The graph engine is the orchestration core. A graph is a directed structure: nodes do work, edges decide where to go next. Nodes are async functions or `ToolNode` / `Agent` instances; edges are static (always go to X) or conditional (a function picks the next key).

## Construction

```python
from agentflow.core.graph import StateGraph

graph = StateGraph()
```

Optional constructor args (all keyword-only, all optional):
- `state` — a custom `AgentState` subclass instance to seed default state.
- `context_manager` — a `BaseContextManager` for token-budget control between turns.
- `publisher` — a `BasePublisher` (Redis/Kafka/RabbitMQ) for event emission.
- `id_generator` — override the default ID scheme (string/int/bigint).
- `container` — an `InjectQ` container for dependency injection.

## Methods

### `add_node(name, runnable)`

Register a worker. `runnable` can be:
- An `async def fn(state, config=None, ...)` returning a dict, a `Message`, or `None`.
- An `Agent(...)` instance (LLM call + auto tool routing).
- A `ToolNode([...])` instance (tool execution).

```python
graph.add_node("MAIN", main_agent)
graph.add_node("TOOL", ToolNode([get_weather]))
```

### `add_edge(from_node, to_node)`

Unconditional: when `from_node` finishes, always go to `to_node`.

```python
graph.add_edge("TOOL", "MAIN")  # tools always loop back to the main LLM
```

### `add_conditional_edges(from_node, route_fn, mapping)`

The router function inspects state and returns a key; the mapping resolves the key to a node name (or `END` to terminate).

```python
from agentflow.utils.constants import END

def route(state):
    if state.context and state.context[-1].tools_calls:
        return "TOOL"
    return END

graph.add_conditional_edges("MAIN", route, {"TOOL": "TOOL", END: END})
```

The mapping must include every key your router can return. If a key is missing, execution raises.

### `set_entry_point(name)`

Required. Marks the first node to run on `invoke()`.

```python
graph.set_entry_point("MAIN")
```

### `compile(checkpointer=None, callbacks=None, publisher=None) -> CompiledGraph`

Returns the runnable. After this point, structural changes are frozen.

```python
app = graph.compile(checkpointer=InMemoryCheckpointer())
```

### `override_node(name, new_runnable)`

Replace a node's runnable post-construction (useful for testing, A/B swapping models). Only valid before `compile()`.

## Running the compiled graph

```python
config = {"thread_id": "user-42", "recursion_limit": 25}

# Sync
result = app.invoke({"messages": [Message.text_message("hi")]}, config=config)

# Async streaming
async for event in app.astream(input, config=config):
    ...
```

Required config keys:
- `thread_id` — opaque session ID. Same value resumes the same conversation if a checkpointer is set.

Useful config keys:
- `recursion_limit` — max number of node transitions per `invoke`. Default ~25. Cap it to prevent runaway loops between an LLM and its tools.

## Common topology recipes

**Linear pipeline** (A → B → C → END):
```python
graph.add_node("A", a); graph.add_node("B", b); graph.add_node("C", c)
graph.add_edge("A", "B"); graph.add_edge("B", "C"); graph.add_edge("C", END)
graph.set_entry_point("A")
```

**ReAct loop** (LLM ↔ Tools):
```python
graph.add_conditional_edges("MAIN", route, {"TOOL": "TOOL", END: END})
graph.add_edge("TOOL", "MAIN")
```

**Fan-out / fan-in** (parallel branches): use multiple `add_edge("ROUTER", branch)` calls and a join node. The graph engine runs siblings concurrently when possible.

## See also

- [agent-class.md](./agent-class.md) — the `Agent` node that handles LLM + tool orchestration for you.
- [tools.md](./tools.md) — `ToolNode` and tool wiring.
- [state.md](./state.md) — what flows between nodes.
- [streaming.md](./streaming.md) — `astream` events.
