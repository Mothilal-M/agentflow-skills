---
name: imports
description: Canonical Python import paths for the public 10xscale-agentflow API surface
metadata:
  tags: imports, api, gotchas
---

The package's top-level `agentflow.graph` / `agentflow.state` / `agentflow.checkpointer` paths shown in some older docs **do not exist** at runtime. The real public surface lives under `agentflow.core.*` and `agentflow.storage.*`. Always use the imports below.

## Graph & agent

```python
from agentflow.core.graph import (
    Agent,
    StateGraph,
    ToolNode,
    CompiledGraph,
    Edge,
    Node,
    BaseAgent,
    RetryConfig,
)
```

## State & messages

```python
from agentflow.core.state import (
    AgentState,
    Message,
    BaseContextManager,
    ContentBlock,
    TextBlock,
    ImageBlock,
    AudioBlock,
    DocumentBlock,
    VideoBlock,
    ToolCallBlock,
    ToolResultBlock,
    ReasoningBlock,
)
```

`Message` factories: `Message.text_message(...)`, `Message.tool_message(...)`, `Message.image_message(...)`, `Message.multimodal_message(...)`, `Message.from_file(...)`. There is **no** `Message.from_text(...)` — older snippets that use it are wrong.

## Constants

```python
from agentflow.utils.constants import END, START
```

`END` is the value `"__end__"` returned from routing functions to terminate the graph.

## Checkpointers

```python
from agentflow.storage.checkpointer import (
    BaseCheckpointer,
    InMemoryCheckpointer,
    PgCheckpointer,  # NOT "PgRedisCheckpointer"
)
```

For Postgres checkpointing install the extra: `pip install 10xscale-agentflow[pg_checkpoint]`.

## Prebuilt agents

```python
from agentflow.prebuilt import ReactAgent, RAGAgent, RouterAgent
```

## Quick gotcha cheat-sheet

| Wrong (do not use) | Right |
|--------------------|-------|
| `from agentflow.graph import StateGraph` | `from agentflow.core.graph import StateGraph` |
| `from agentflow.state import AgentState, Message` | `from agentflow.core.state import AgentState, Message` |
| `from agentflow.checkpointer import InMemoryCheckpointer` | `from agentflow.storage.checkpointer import InMemoryCheckpointer` |
| `PgRedisCheckpointer` | `PgCheckpointer` |
| `Message.from_text("hi")` | `Message.text_message("hi")` |
