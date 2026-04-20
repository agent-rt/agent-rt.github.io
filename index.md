---
title: agent-rt
---

# agent-rt

Local-first, agent-native developer tools. Built in Rust, distributed as single
binaries, designed to be driven by AI agents through MCP.

## Products

### [Synap](/synap/)

Structured memory engine for AI agents. One SQLite file under `~/.synap/`,
hybrid keyword + semantic search, dynamic app namespaces (tasks, notes,
accounting, knowledge bases), long-term user memory — all exposed via MCP.

```sh
brew tap agent-rt/tap
brew install synap
```

Status: **0.1.x preview** — macOS arm64 only.

→ [Install](/synap/install/) · [Quickstart](/synap/quickstart/) · [Recipes](/synap/recipes/) · [Reference](/synap/reference/)

### [cmcp](/cmcp/)

*curl for MCP.* One-shot CLI for calling any Model Context Protocol tool
from the terminal, reusing your existing Claude / Cursor configs. Built
for the edge case that happens every ten minutes — iterating on an MCP
server without restarting your Agent.

```sh
cargo install cmcp
cmcp cortex/list_projects
```

Status: **0.1.x preview** — macOS / Linux.

→ [Install](/cmcp/install/) · [Quickstart](/cmcp/quickstart/) · [Configuration](/cmcp/configuration/) · [Commands](/cmcp/commands/)

---

More products coming. Source: [github.com/agent-rt](https://github.com/agent-rt).
