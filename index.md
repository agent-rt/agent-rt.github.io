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

### [skillctl](/skillctl/)

*Profile-driven Agent skill manager and protocol gateway.* Lets users curate
skill bundles once and apply them to projects with `skillctl init <profile>`.
Exposes a stable shell-level protocol any agent can consume through four
commands documented in `AGENTS.md`. Mandatory `namespace/name` IDs eliminate
collisions; the managed `AGENTS.md` block is byte-stable across skill churn,
keeping prompt caches warm.

```sh
brew tap agent-rt/tap
brew install skillctl
skillctl init rust-cli                         # apply a profile to a project
skillctl list --project --format tsv           # 4-col agent catalog
```

Status: **0.1.x preview** — macOS / Linux.

→ [Install](/skillctl/install/) · [Quickstart](/skillctl/quickstart/) · [Commands](/skillctl/commands/)

### [memctl](/memctl/)

*Persistent agent memory layer — cross-tool, cross-project, cross-session.*
Topic-based markdown stream with timestamped, attributed entries. Seven
typed memory kinds (`lesson`, `decision`, `fact`, `feedback`, `reference`,
`user`, `project`) and three scopes (`global`, `project`, `agent:<name>`).
Project-scoped memory commits with code — every team member who clones the
repo inherits the lore. Independent `AGENTS.md` block coexists with
`skillctl`.

```sh
brew tap agent-rt/tap
brew install memctl
memctl save --type decision --topic api-conventions \
  "POST /payments must include Idempotency-Key header"
memctl list --format tsv
```

Status: **0.1.x preview** — macOS / Linux.

→ [Install](/memctl/install/) · [Quickstart](/memctl/quickstart/) · [Commands](/memctl/commands/)

### [imgctl](/imgctl/)

*Agent-first image processing CLI.* Single static Rust binary with 16
commands for editing (convert, resize, crop, compose), annotation (text,
arrows, boxes, blur), analysis (compare, diff, histogram, palette), and
diagram rendering (Mermaid via headless Chrome). TSV output by default,
`--json` for structured consumption, stable error codes everywhere.

```sh
brew tap agent-rt/tap
brew install imgctl
imgctl resize -i in.png -o out.png --width 400 --fit contain --json
imgctl mermaid -i flow.mmd -o flow.png --width 1200
imgctl diff -i baseline.png -i current.png -o diff.png --threshold 0.05
```

Status: **0.1.x preview** — macOS / Linux.

→ [Install](/imgctl/install/) · [Quickstart](/imgctl/quickstart/) · [Commands](/imgctl/commands/)

---

More products coming. Source: [github.com/agent-rt](https://github.com/agent-rt).
