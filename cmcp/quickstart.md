---
title: Quickstart · cmcp
---

# Quickstart

One minute from install to your first MCP tool call. Assumes you've run at
least one MCP server through Claude Desktop, Claude Code, or Cursor
already — cmcp reuses those configs.

## 1. Install

```sh
cargo install cmcp
# or: curl … | tar -xz (see Install)
```

See [Install](/cmcp/install/) for all options.

## 2. See what cmcp found

```sh
cmcp config sources
```

```
SOURCE           EXISTS   PATH
cmcp             no       /Users/you/.config/cmcp/mcp.json
claude-code      yes      /Users/you/.claude.json
claude-desktop   yes      /Users/you/Library/Application Support/Claude/claude_desktop_config.json
cursor           no       /Users/you/.cursor/mcp.json
```

```sh
cmcp server list
```

```
SERVER                   SOURCE           TRANSPORT
cortex                   claude-code      stdio: cortex mcp
github                   claude-desktop   stdio: npx -y @modelcontextprotocol/server-github
```

## 3. List a server's tools

```sh
cmcp tool list cortex
```

```
create_nodes             Batch create nodes with optional dependency edges…
get_node_context         Get a node with its dependency subgraph…
list_nodes               List nodes by type and/or status filter.
…
```

## 4. Call a tool

```sh
cmcp cortex/list_projects
# or, with the explicit scheme (same thing):
cmcp mcp://cortex/list_projects
```

Pass arguments:

```sh
cmcp github/search_repos --arg query=rust --arg limit=5

# or a full JSON object
cmcp github/search_repos --args-json '{"query":"rust","limit":5}'
```

Pipe to `jq`:

```sh
cmcp cortex/list_projects --json \
  | jq '.content[0].text' -r \
  | head
```

## 5. Add your own dev server

Developing a server locally? Register it in cmcp's own config once and
you're done — no Agent restart needed between rebuilds.

```sh
cmcp server add my-dev \
  --command /path/to/target/debug/my-mcp-server \
  --env DEBUG=1
```

Then iterate:

```sh
cargo build                       # rebuild your server
cmcp tool list my-dev             # hits the fresh binary
cmcp my-dev/some_tool --arg k=v   # test the tool
```

Remove when done:

```sh
cmcp server remove my-dev
```

## What's next

- [Configuration](/cmcp/configuration/) — precedence between cmcp's own
  config and each Agent's.
- [Commands](/cmcp/commands/) — every flag, every subcommand.
- [Troubleshooting](/cmcp/troubleshooting/) — when `stdio handshake
  failed`, look here.
