---
name: models
description: LiteLLM-style model strings - provider mapping, choosing models, environment variables
metadata:
  tags: models, llm, litellm, openai, anthropic, gemini
---

The `model` parameter on `Agent(...)` is a LiteLLM-style string: `<provider>/<model-id>`. AgentFlow routes through LiteLLM, so any provider/model LiteLLM supports works.

## Format

```
"openai/gpt-4o"
"openai/gpt-4o-mini"
"anthropic/claude-3-5-sonnet-latest"
"anthropic/claude-3-7-sonnet-20250219"
"gemini/gemini-2.5-flash"
"gemini/gemini-2.5-pro"
"groq/llama-3.1-70b-versatile"
"bedrock/anthropic.claude-3-5-sonnet-20241022-v2:0"
"ollama/llama3.1"  # local
"vertex_ai/gemini-2.5-pro"
```

## API key environment variables

Set the key for your provider before calling the agent. AgentFlow auto-loads `.env` if present.

| Provider prefix | Env var |
|------------------|---------|
| `openai/` | `OPENAI_API_KEY` |
| `anthropic/` | `ANTHROPIC_API_KEY` |
| `gemini/` | `GEMINI_API_KEY` |
| `groq/` | `GROQ_API_KEY` |
| `bedrock/` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION_NAME` |
| `vertex_ai/` | `GOOGLE_APPLICATION_CREDENTIALS` (service account JSON) + `VERTEXAI_PROJECT`, `VERTEXAI_LOCATION` |
| `ollama/` | `OLLAMA_API_BASE` (default `http://localhost:11434`) |
| `azure/` | `AZURE_API_KEY`, `AZURE_API_BASE`, `AZURE_API_VERSION` |

## Picking a model

Rough guidance:

| Need | Recommended |
|------|-------------|
| Fast + cheap default | `openai/gpt-4o-mini` or `gemini/gemini-2.5-flash` |
| Highest reasoning quality | `openai/gpt-4o`, `anthropic/claude-3-7-sonnet-latest` |
| Long context (1M+ tokens) | `gemini/gemini-2.5-pro` |
| Local / offline | `ollama/llama3.1` (or whatever you've pulled) |
| Strict structured output | `openai/gpt-4o` (best JSON mode), `gemini/gemini-2.5-flash` |
| Tool calling reliability | `openai/gpt-4o`, `anthropic/claude-3-5-sonnet-latest` |

## Fallback chains

Pass alternates to ride out provider outages:

```python
agent = Agent(
    model="anthropic/claude-3-5-sonnet-latest",
    fallback_models=[
        "openai/gpt-4o",
        "openai/gpt-4o-mini",
    ],
)
```

If the primary returns a 5xx / rate limit / timeout, AgentFlow tries the next entry. Each fallback respects the same `retry_config`.

## Model-specific config

### Reasoning tokens (Anthropic extended thinking, etc.)

```python
from agentflow.core.graph import Agent
# Provider-specific reasoning_config shape; consult AgentFlow docs.
agent = Agent(
    model="anthropic/claude-3-7-sonnet-20250219",
    reasoning_config={"max_thinking_tokens": 8000},
)
```

### Temperature, max_tokens, etc.

These are LiteLLM standard params; they're passed through via the `extra_messages` / per-call config. For most agents, defaults are fine — adjust at the system-prompt level instead of fiddling with sampling.

## Model parity checks

Some agents only work well on tool-calling-strong models. If you swap to a weaker one and tools stop firing, the model is the regression source — not your code. Confirm with:

```python
result = app.invoke({"messages": [...]}, config={...})
print(result["messages"][-1].tools_calls)  # should be non-empty when expected
```

## See also

- [agent-class.md](./agent-class.md) — the `Agent(model=...)` parameter.
- [tools.md](./tools.md) — tool calling, which depends heavily on model choice.
