---
title: Configuration · mcpctl
---

# Configuration

mcpctl is stateless — no daemon, no database — but it reads a handful of
JSON files to know what MCP servers exist.

## Sources and precedence

mcpctl auto-discovers up to seven sources. On a name collision, the higher
priority (lower number) wins.

| Priority | Source         | Path                                                                                    | Writable   |
|---------:|----------------|-----------------------------------------------------------------------------------------|:----------:|
| 1        | mcpctl         | `$XDG_CONFIG_HOME/mcpctl/mcp.json` (fallback `~/.config/mcpctl/mcp.json`)              | ✅         |
| 2        | Claude Code    | `~/.claude.json`                                                                        | read-only  |
| 3        | Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS)               | read-only  |
| 4        | Cursor         | `~/.cursor/mcp.json`                                                                    | read-only  |
| 5        | Windsurf       | `~/.codeium/windsurf/mcp_config.json`                                                   | read-only  |
| 6        | Gemini CLI     | `~/.gemini/settings.json`                                                               | read-only  |
| 7        | Zed            | `~/Library/Application Support/Zed/settings.json` (macOS) / `~/.config/zed/settings.json` (Linux) | read-only |

Only mcpctl's own file is writable by `mcpctl server add` / `remove` / `edit`.
The agent configs stay untouched — mcpctl never mutates them.

List exactly what mcpctl sees:

```sh
mcpctl config sources
```

## Schema

Most sources use the standard `mcpServers` shape:

```json
{
  "mcpServers": {
    "my-stdio-server": {
      "command": "node",
      "args": ["server.js"],
      "env": {
        "API_TOKEN": "sk-xxx"
      }
    },
    "my-http-server": {
      "url": "https://mcp.example.com/mcp",
      "headers": {
        "Authorization": "Bearer xxx"
      }
    }
  }
}
```

### Zed format

Zed uses a different top-level key (`context_servers`) and nests command
info under a `settings` object:

```json
{
  "context_servers": {
    "my-server": {
      "settings": {
        "command": "node",
        "args": ["server.js"],
        "env": { "KEY": "val" }
      }
    }
  }
}
```

mcpctl parses both formats transparently.

## mcpctl's own config

mcpctl writes to `~/.config/mcpctl/mcp.json`. Use it to:

- **Register dev servers** without touching your agent's config.
- **Override** a server defined by an agent — same name, different command.
- **Stage** a server before promoting it to a shared agent config.

### Managed via CLI

```sh
mcpctl server add my-stdio  --command node --arg server.js --env DEBUG=1
mcpctl server add my-api    --url https://mcp.example.com/mcp \
                            --header 'Authorization: Bearer xxx'
mcpctl server add my-stdio  --command ./new-binary --force   # overwrite
mcpctl server remove my-stdio
mcpctl server edit                                           # opens $EDITOR
```

Writes are atomic (tmp file + rename). Missing parent directories are
created on first write.

### Manual edit

Hand-editing is fine — `mcpctl server edit` just opens `$EDITOR` on the file.
The schema matches Claude Code's `.claude.json`, so entries can be copied
directly between them.

## Migration from cmcp

On first run, if `~/.config/cmcp/mcp.json` exists (the old name), it is
silently copied to `~/.config/mcpctl/mcp.json`. Nothing is deleted from
the old location.

## Test / sandbox override

For CI or scripted testing, `MCPCTL_CONFIG_DIR=/path/to/dir` makes mcpctl
look only in that directory — both `mcp.json` (own config) and `.claude.json`
(Claude Code emulation) — and skip all other sources.

```sh
MCPCTL_CONFIG_DIR=/tmp/test mcpctl server list
```

This is how mcpctl's own integration tests isolate themselves from real user config.

## Secrets handling

- `mcpctl server show <name>` redacts any env key or header matching
  `TOKEN` / `SECRET` / `KEY` / `PASSWORD` / `AUTH` / `BEARER` (case-insensitive).
- `mcpctl server show <name> --reveal` prints values verbatim.
- `mcpctl server list` never prints env or headers.

mcpctl does not encrypt the on-disk config — it's plain JSON. If that's a
concern, use a secret manager wrapper in the `command` field instead of
inline `env`.
