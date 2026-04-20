---
title: Configuration · cmcp
---

# Configuration

cmcp is stateless — no daemon, no database — but it does read a handful
of JSON files to know what MCP servers exist.

## Sources and precedence

cmcp auto-discovers up to four sources. On a name collision, the higher
priority wins.

| Priority | Source         | Path                                                                        | Writable   |
|---------:|----------------|-----------------------------------------------------------------------------|:----------:|
| 1        | cmcp           | `$XDG_CONFIG_HOME/cmcp/mcp.json` (fallback `~/.config/cmcp/mcp.json`)       | ✅         |
| 2        | Claude Code    | `~/.claude.json`                                                            | read-only  |
| 3        | Claude Desktop | `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS)   | read-only  |
| 4        | Cursor         | `~/.cursor/mcp.json`                                                         | read-only  |

Only cmcp's own file is writable by `cmcp server add` / `remove` / `edit`.
The Agent configs stay untouched — cmcp never mutates them. To change
a server that lives in Claude Desktop, edit Claude Desktop's config.

List exactly what cmcp sees:

```sh
cmcp config sources
```

## Schema

All four sources use the same shape as Claude Code's `.claude.json`:

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

Keys:

- **`command`** + optional **`args`** + optional **`env`** — stdio
  transport. cmcp spawns the command as a child process and speaks
  JSON-RPC over stdin/stdout.
- **`url`** + optional **`headers`** — streamable-HTTP transport. Any
  header named `Authorization` is lifted to the rmcp `auth_header`
  config; others are sent verbatim.

The `command` and `url` fields are mutually exclusive per server.

## cmcp's own config

cmcp writes to `~/.config/cmcp/mcp.json` (or `$XDG_CONFIG_HOME/cmcp/mcp.json`).
Use it to:

- **Register dev servers** that you don't want polluting your Agent's
  config.
- **Override** a server that an Agent defines — same name, different
  command. Handy for `cmcp`-only overrides without touching Claude.
- **Stage** a server before adding it to an Agent (copy the JSON entry
  across once you're happy with it).

### Managed via CLI

```sh
cmcp server add my-stdio  --command node --arg server.js --env DEBUG=1
cmcp server add my-api    --url https://mcp.example.com/mcp \
                          --header 'Authorization: Bearer xxx'
cmcp server add my-stdio  --command ./new-binary --force       # overwrite
cmcp server remove my-stdio
cmcp server edit                                                # opens $EDITOR
```

Writes are atomic (tmp file + rename). Missing parent directories are
created on first write.

### Manual edit

Hand-editing is fine; `cmcp server edit` just launches `$EDITOR` on the
file. Since the schema matches Claude Code's, you can copy-paste entries
between `~/.claude.json` and `~/.config/cmcp/mcp.json`.

## Test / sandbox override

For CI or scripted testing, `CMCP_CONFIG_DIR=/path/to/dir` makes cmcp
look at *only* that directory — both `mcp.json` (own config) and
`.claude.json` (Claude Code emulation) — and skip Desktop / Cursor
entirely.

```sh
CMCP_CONFIG_DIR=/tmp/test cmcp server list
```

This is how cmcp's own integration tests isolate themselves from your
real user config.

## Secrets handling

- `cmcp server show <name>` redacts any env key or header name matching
  `TOKEN` / `SECRET` / `KEY` / `PASSWORD` / `AUTH` / `BEARER`
  (case-insensitive).
- `cmcp server show <name> --reveal` prints them verbatim.
- `cmcp server list` never prints env / headers.

cmcp does not encrypt the on-disk config — it's plain JSON, same as
Agent configs. If that's a concern, use a secret manager wrapper in the
`command` field instead of inline `env`.
