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

### [acpctl](/acpctl/)

*The ACP control utility for AI agents.* Discover, inspect, and invoke any
[ACP (Agent Client Protocol)][acp] agent with tagged NDJSON streams,
structured errors, and token-lean defaults. A built-in session daemon
keeps agent subprocesses alive across CLI invocations — so conversations
survive between agent turns even when the underlying agent (e.g.
`gemini-cli`) doesn't support cross-process `session/load`.

```sh
brew tap agent-rt/tap
brew install acpctl
acpctl schema                               # self-describing JSON contract
acpctl call gemini --prompt-file - < p.txt  # tagged NDJSON stream
```

Status: **pre-release** — no tagged version yet; install from source.

→ [Install](/acpctl/install/) · [Quickstart](/acpctl/quickstart/) · [Configuration](/acpctl/configuration/) · [Commands](/acpctl/commands/) · [Troubleshooting](/acpctl/troubleshooting/)

[acp]: https://agentclientprotocol.com/

---

More products coming. Source: [github.com/agent-rt](https://github.com/agent-rt).
