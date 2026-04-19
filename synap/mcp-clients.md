---
title: MCP Clients · Synap
---

# MCP Client Configuration

Synap speaks MCP over stdio. Any MCP-compatible client can drive it. Tool
names appear to the model prefixed with `mcp__synap__` (e.g.,
`mcp__synap__query`).

## Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "synap": {
      "command": "synap",
      "args": ["mcp"]
    }
  }
}
```

Restart Claude Desktop.

## Cursor

Settings → MCP → Add new MCP server, or edit `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "synap": {
      "command": "synap",
      "args": ["mcp"]
    }
  }
}
```

## Zed

`~/.config/zed/settings.json`:

```json
{
  "context_servers": {
    "synap": {
      "command": {
        "path": "synap",
        "args": ["mcp"]
      }
    }
  }
}
```

## Claude Code (CLI)

Project-level `.mcp.json`, or global via `~/.claude.json`:

```json
{
  "mcpServers": {
    "synap": {
      "command": "synap",
      "args": ["mcp"]
    }
  }
}
```

Then `claude mcp list` should show `synap ✓ Connected`.

## Verifying a connection

Ask the model:

> List the synap apps available.

It should call `mcp__synap__list_apps` and return at least the built-ins
(`synap.memories`, `synap.sessions`, `synap.views`).

## Common issues

- **"command not found: synap"** — the client's PATH may not include
  `/usr/local/bin`. Use an absolute path: `"command": "/usr/local/bin/synap"`.
- **Daemon not starting** — check `~/.synap/synap.log`. See
  [Troubleshooting](/synap/troubleshooting/).
- **Tools don't show up** — make sure the client fully restarted (Claude
  Desktop: quit from the menu bar, not just the window).
