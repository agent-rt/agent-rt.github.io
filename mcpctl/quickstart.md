---
title: Quickstart · mcpctl
---

# Quickstart

One minute from install to your first MCP tool call. Assumes you've already
configured at least one MCP server in Claude Code, Claude Desktop, Cursor,
Windsurf, Gemini CLI, or Zed — mcpctl reuses those configs automatically.

## 1. Install

```sh
cargo install mcpctl
# or: curl … | tar -xz (see Install)
```

See [Install](/mcpctl/install/) for all options.

## 2. See what mcpctl found

```sh
mcpctl config sources
```

```
SOURCE           EXISTS   PATH
mcpctl           no       /Users/you/.config/mcpctl/mcp.json
claude-code      yes      /Users/you/.claude.json
claude-desktop   yes      /Users/you/Library/Application Support/Claude/claude_desktop_config.json
cursor           no       /Users/you/.cursor/mcp.json
windsurf         no       /Users/you/.codeium/windsurf/mcp_config.json
gemini           no       /Users/you/.gemini/settings.json
zed              no       /Users/you/Library/Application Support/Zed/settings.json
```

```sh
mcpctl server list
```

```
SERVER                   SOURCE           TRANSPORT
cortex                   claude-code      stdio: cortex mcp
github                   claude-desktop   stdio: npx -y @modelcontextprotocol/server-github
```

## 3. Introspect a server (agent-optimized)

Get tools + prompts + resources in a single handshake — the most efficient
way for an agent to build a capability map:

```sh
mcpctl introspect cortex --json
```

Or list tools only:

```sh
mcpctl tool list cortex
```

```
create_nodes             Batch create nodes with optional dependency edges…
get_node_context         Get a node with its dependency subgraph…
list_nodes               List nodes by type and/or status filter.
…
```

## 4. Call a tool

```sh
mcpctl cortex/list_projects
# or, with the explicit scheme:
mcpctl mcp://cortex/list_projects
```

Pass arguments:

```sh
mcpctl github/search_repos --arg query=rust --arg limit=5

# or a full JSON object
mcpctl github/search_repos --args-json '{"query":"rust","limit":5}'
```

Machine-readable output with `--json`:

```sh
mcpctl --json cortex/list_projects \
  | jq '.content[0].text' -r
```

## 5. Add your own dev server

Developing a server locally? Register it in mcpctl's own config once and
you're done — no agent restart needed between rebuilds.

```sh
mcpctl server add my-dev \
  --command /path/to/target/debug/my-mcp-server \
  --env DEBUG=1
```

Then iterate:

```sh
cargo build                           # rebuild your server
mcpctl tool list my-dev               # hits the fresh binary
mcpctl my-dev/some_tool --arg k=v     # test the tool
```

Remove when done:

```sh
mcpctl server remove my-dev
```

## What's next

- [Configuration](/mcpctl/configuration/) — precedence between mcpctl's own
  config and each agent's.
- [Commands](/mcpctl/commands/) — every flag, every subcommand.
- [Troubleshooting](/mcpctl/troubleshooting/) — when `stdio handshake
  failed`, look here.
