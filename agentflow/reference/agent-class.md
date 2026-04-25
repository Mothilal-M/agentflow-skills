---
name: agent-class
description: The Agent class - parameters, tool wiring, fallback models, retries, structured output
metadata:
  tags: agent, llm, model, structured-output, retries
---

`Agent` is the smart node that handles LLM calls, message conversion, and tool-call dispatching automatically. Use it instead of writing `async def my_node(state): ...` boilerplate when you want a standard LLM-with-tools loop.

```python
from agentflow.core.graph import Agent, ToolNode

tool_node = ToolNode([get_weather])

agent = Agent(
    model="gemini/gemini-2.5-flash",
    system_prompt=[{"role": "system", "content": "You are a helpful assistant."}],
    tool_node=tool_node,
)
```

## Constructor parameters

All keyword-only.

| Param | Type | Purpose |
|-------|------|---------|
| `model` | `str` | LiteLLM-style model string. See [models.md](./models.md). |
| `system_prompt` | `list[dict]` | Initial system messages. Each dict has `role` + `content`. |
| `tool_node` | `ToolNode \| None` | The tool execution node this agent dispatches to. **Pass the instance, not the name.** |
| `output_type` | `type[BaseModel] \| None` | A Pydantic model for structured output. The LLM is forced to return JSON matching this schema. |
| `extra_messages` | `list[dict] \| Callable` | Additional messages injected each turn (e.g. dynamic memory, RAG-retrieved context). |
| `trim_context` | `bool \| int` | Auto-trim old messages to stay under the model's context window. Pass `True` for auto, an int for max-messages. |
| `tools_tags` | `list[str] \| None` | Filter which tools from `tool_node` are advertised this turn (tag-based gating). |
| `reasoning_config` | `ReasoningConfig \| None` | Enable / configure reasoning tokens for models that support it (Anthropic extended thinking, etc.). |
| `skills` | `list[SkillConfig] \| None` | Internal AgentFlow "skills" config — distinct from coding-agent skills, this controls runtime capability scoping. |
| `memory` | `Memory \| None` | Long-term memory adapter (vector DB / KV). Replaces session memory for cross-thread recall. |
| `retry_config` | `RetryConfig \| None` | Retry behavior on LLM errors (rate limits, timeouts). |
| `fallback_models` | `list[str] \| None` | Tried in order if the primary model fails. E.g. `["openai/gpt-4o", "openai/gpt-4o-mini"]`. |
| `multimodal_config` | `MultimodalConfig \| None` | Defaults for image / audio / document handling. |

## Common patterns

### Plain tool-calling agent

```python
agent = Agent(
    model="openai/gpt-4o-mini",
    system_prompt=[{"role": "system", "content": "Be concise."}],
    tool_node=ToolNode([search, fetch]),
)
```

### Structured output (typed)

```python
from pydantic import BaseModel

class Plan(BaseModel):
    steps: list[str]
    deadline: str

agent = Agent(
    model="openai/gpt-4o",
    output_type=Plan,
    system_prompt=[{"role": "system", "content": "Return a plan as JSON."}],
)
```

The result's `parsed` attribute will be a `Plan` instance.

### Fallbacks for reliability

```python
agent = Agent(
    model="anthropic/claude-3-5-sonnet-latest",
    fallback_models=["openai/gpt-4o", "openai/gpt-4o-mini"],
    retry_config=RetryConfig(max_attempts=3, backoff_seconds=2),
    tool_node=tool_node,
)
```

If Anthropic 5xxs three times, AgentFlow rolls down the fallback list automatically.

### Tag-gated tools

```python
tool_node = ToolNode([
    fn_a,  # tagged with "general" by default
    fn_with_tag(fn_b, tags=["billing"]),
])
billing_agent = Agent(model="...", tool_node=tool_node, tools_tags=["billing"])
```

Only billing tools are exposed in this agent's turn.

## Wiring into a graph

`Agent` is a node. Add it like any other node:

```python
graph.add_node("MAIN", agent)
graph.add_node("TOOL", tool_node)  # the same tool_node you passed to Agent

graph.add_conditional_edges("MAIN", route, {"TOOL": "TOOL", END: END})
graph.add_edge("TOOL", "MAIN")
graph.set_entry_point("MAIN")
```

The router function's job is to inspect `state.context[-1].tools_calls` and decide whether the LLM asked for a tool call. The Agent itself does **not** auto-route; it produces the LLM response and stops.

## See also

- [tools.md](./tools.md) — building the `ToolNode` you pass to `Agent`.
- [graph.md](./graph.md) — wiring the Agent into a `StateGraph`.
- [state.md](./state.md) — inspecting `state.context` for routing.
- [models.md](./models.md) — model string format and provider mapping.
