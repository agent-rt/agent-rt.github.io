---
title: cmcp
---

# cmcp

`cmcp` is **curl for MCP** — a one-shot CLI that calls any
[Model Context Protocol](https://modelcontextprotocol.io/) tool from the
terminal, reusing the MCP server configs you've already set up in Claude
Code, Claude Desktop, or Cursor.

```sh
cmcp cortex/list_projects
cmcp github/search_repos --arg query=rust --arg limit=5
cmcp mcp://my-dev-server/hello --args-json '{"name":"world"}'
```

## Why

When you're building an MCP server, every rebuild means restarting your
Agent to pick up the new binary — which blows away the prompt cache, kicks
you out of your flow, and is flat-out slow. `cmcp` sidesteps the Agent
entirely: one CLI invocation, one session, one tool call, done.

It's also useful outside dev:

- **Glue** — pipe MCP tool output into `jq`, `grep`, or a shell script.
- **CI** — run an MCP tool deterministically, check exit codes.
- **Exploration** — list tools and schemas without a chat UI.

## Design principles

- **No daemon, no state.** Each invocation connects, calls, disconnects.
  Like `curl`, not like `ssh`.
- **Reuse your existing config.** cmcp reads Agent configs read-only, plus
  its own writable file for local overrides.
- **stdio-first.** The primary transport is subprocess stdio — the most
  common shape for local MCP servers under development.

## Get started

1. **[Install](/cmcp/install/)** — cargo or prebuilt tarball.
2. **[Quickstart](/cmcp/quickstart/)** — call your first MCP tool in under a minute.
3. **[Configuration](/cmcp/configuration/)** — where cmcp looks for servers, and how to add your own.
4. **[Commands](/cmcp/commands/)** — full reference for `server`, `tool`, `config`, and `mcp://` calls.
5. **[Troubleshooting](/cmcp/troubleshooting/)** — stdio protocol gotchas, exit codes, and common failures.

## Links

- Source: [github.com/agent-rt/cmcp](https://github.com/agent-rt/cmcp)
- Releases: [github.com/agent-rt/cmcp/releases](https://github.com/agent-rt/cmcp/releases)
- Issues: [github.com/agent-rt/cmcp/issues](https://github.com/agent-rt/cmcp/issues)
