---
title: mcpctl
---

# mcpctl

`mcpctl` is the **MCP control utility for AI agents** — a single-binary CLI that lets agents discover, inspect, and invoke any MCP server configured in their environment, without restarts, without extra daemons, without extra config.

```sh
# Agent perspective: enumerate servers, inspect capabilities, call a tool
mcpctl server list --json
mcpctl introspect github --json
mcpctl github/search_repos --args-json '{"query":"mcp","language":"rust"}'
```

## Why agent-first

`mcpctl` is infrastructure for agents, not a developer GUI:

- **Discovery** — `mcpctl server list` lets an agent enumerate all available MCP servers at runtime.
- **Introspection** — `mcpctl introspect <server> --json` returns tools + prompts + resources in one round-trip, giving the agent a full capability map before deciding what to call.
- **Invocation** — `mcpctl <server>/<tool> --args-json '...'` invokes any tool with a single shell command — no SDK, no handshake boilerplate.
- **Machine-readable output** — `--json` on any command returns structured JSON; errors go to stderr as `{"error":"...","code":N}` so agents can parse and branch without string matching.
- **Config reuse** — reads existing Claude Code, Claude Desktop, Cursor, Windsurf, Gemini CLI, and Zed configs — zero extra setup for the agent.

## Design principles

- **No daemon, no state.** Each invocation connects, calls, disconnects. Stateless by design — like `curl`, not like `ssh`. Per-call cold start (~100–500ms) is acceptable for agent tool use frequency.
- **Agent output contracts.** `--json` always emits valid JSON; exit codes are stable and documented; all human-readable text goes to stdout, errors to stderr.
- **Config reuse over new config.** `mcpctl` reads the MCP server registries you've already set up in any supported agent environment. Its own writable file is for additions and overrides only.

## Get started

1. **[Install](/mcpctl/install/)** — cargo or prebuilt tarball.
2. **[Quickstart](/mcpctl/quickstart/)** — discover and call your first MCP tool in under a minute.
3. **[Configuration](/mcpctl/configuration/)** — where mcpctl looks for servers, and how to add your own.
4. **[Commands](/mcpctl/commands/)** — full reference for all subcommands.
5. **[Troubleshooting](/mcpctl/troubleshooting/)** — stdio protocol gotchas, exit codes, and common failures.

## Links

- Source: [github.com/agent-rt/mcpctl](https://github.com/agent-rt/mcpctl)
- Releases: [github.com/agent-rt/mcpctl/releases](https://github.com/agent-rt/mcpctl/releases)
- Issues: [github.com/agent-rt/mcpctl/issues](https://github.com/agent-rt/mcpctl/issues)
