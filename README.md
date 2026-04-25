# agentflow-skills

Agent skills for users of [10xscale-agentflow](https://pypi.org/project/10xscale-agentflow/). These teach coding agents (Claude Code, Cursor, Copilot, etc.) how to build real AgentFlow applications.

## Install

```bash
pip install 10xscale-agentflow-cli
agentflow skills add agentflow
agentflow skills add prebuilt-patterns
agentflow skills add tool-integrations
agentflow skills add production
```

Short names resolve against this repo by default. You can also target a specific subpath or a different repo:

```bash
agentflow skills add Mothilal-M/agentflow-skills/agentflow
agentflow skills add https://github.com/someone/other-skills
```

## Available skills

| Skill | Teaches |
|-------|---------|
| `agentflow` | Build your first agent in 10–30 lines: `Agent`, `StateGraph`, `ToolNode`, streaming, checkpointing |
| `prebuilt-patterns` | When to pick `ReactAgent`, `RAGAgent`, `RouterAgent` — with a concrete example per pattern |
| `tool-integrations` | MCP, LangChain, Composio tool integration recipes |
| `production` | `agentflow init` / `api` / `build`, FastAPI deploy, human-in-the-loop, callbacks, publishers |

## Layout

Each skill folder has a `SKILL.md` (frontmatter + guidance) and an optional `rules/` sub-folder for longer references.

## License

MIT.
