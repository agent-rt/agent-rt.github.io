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

### [mcpctl](/mcpctl/)

*The MCP control utility for AI agents.* Discover, inspect, and invoke any
MCP server configured in an agent's environment — without restarts, without
extra daemons, without extra config. Reads Claude Code, Claude Desktop,
Cursor, Windsurf, Gemini CLI, and Zed configs automatically.

```sh
cargo install mcpctl
mcpctl server list --json
mcpctl introspect github --json
mcpctl github/search_repos --args-json '{"query":"mcp"}'
```

Status: **0.1.x preview** — macOS / Linux.

→ [Install](/mcpctl/install/) · [Quickstart](/mcpctl/quickstart/) · [Configuration](/mcpctl/configuration/) · [Commands](/mcpctl/commands/)

---

More products coming. Source: [github.com/agent-rt](https://github.com/agent-rt).
