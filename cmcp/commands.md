---
title: Commands · cmcp
---

# Command Reference

```text
cmcp <URI> [--arg k=v]... [--args-json JSON] [--json] [--timeout N]
cmcp mcp://<server>/<tool> ...    # equivalent
cmcp server list [--json]
cmcp server show <name> [--reveal] [--json]
cmcp server add <name> ( --command <c> [--arg <a>]... [--env K=V]...
                       | --url <u> [--header 'K: V']... ) [--force]
cmcp server remove <name>
cmcp server edit
cmcp tool list <server> [--json] [--timeout N]
cmcp tool show <server> <tool> [--json] [--timeout N]
cmcp config sources [--json]
```

Global flags:

- `-v, --verbose` — forward the child's stderr to the terminal with a
  `[mcp-stderr]` prefix. Off by default (cmcp's stdout stays clean).
- `-h, --help`, `-V, --version`

## Calling a tool

Two equivalent forms (mirrors `curl google.com` vs `curl https://google.com`):

```sh
cmcp cortex/list_projects
cmcp mcp://cortex/list_projects
```

Any other scheme is an error:

```sh
cmcp http://a/b
# cmcp: invalid mcp uri: only mcp:// scheme is supported, got 'http://a/b'
```

### Passing arguments

Two ways, composable.

```sh
# --arg key=value (repeatable). Values are parsed as JSON when possible,
# falling back to string:
cmcp github/search_repos --arg query=rust --arg limit=5 --arg public=true

# --args-json as a base; per-key --arg entries override:
cmcp github/search_repos \
  --args-json '{"query":"base","limit":1}' \
  --arg limit=5
```

Arguments are validated against the tool's `input_schema` by the server,
not by cmcp. If the server rejects them, you'll see the error come back
in the response content.

### Output modes

Default: human-friendly text. Text content is printed verbatim; image /
embedded-resource content prints a one-line placeholder.

`--json`: raw `CallToolResult` JSON — suitable for piping into `jq`.

```sh
cmcp github/search_repos --arg query=rust --json \
  | jq '.content[0].text' -r
```

## Managing servers

### `cmcp server list`

Lists every MCP server across all discovered sources, after precedence
resolution.

```
SERVER                   SOURCE           TRANSPORT
cortex                   claude-code      stdio: cortex mcp
github                   claude-desktop   stdio: npx -y …
my-dev                   cmcp             stdio: node server.js
```

### `cmcp server show <name>`

Full resolved config for one server. Sensitive env / header values are
redacted by default:

```
server:      github
source:      claude-desktop
source_path: /Users/you/Library/Application Support/Claude/claude_desktop_config.json
transport:   stdio
command:     npx
args:        -y @modelcontextprotocol/server-github
env:
  GITHUB_TOKEN=gh…(redacted, 40b)
```

Pass `--reveal` to print the raw values.

### `cmcp server add`

Write a new entry into `~/.config/cmcp/mcp.json`.

```sh
# stdio
cmcp server add local-dev \
  --command /path/to/my-server \
  --arg --port --arg 9001 \
  --env RUST_LOG=debug

# http
cmcp server add remote-api \
  --url https://mcp.example.com/mcp \
  --header 'Authorization: Bearer xxxxxxxx' \
  --header 'X-Region: us-east-1'
```

- `--command` and `--url` are mutually exclusive — each entry is exactly
  one transport.
- Duplicate names error unless you pass `--force`.
- Only cmcp's own file is touched; read-only sources are never modified.

### `cmcp server remove <name>`

Removes the entry from cmcp's own config. If the name only exists in a
read-only source (Claude Code, Desktop, Cursor), cmcp tells you which
file to edit and exits non-zero instead of silently doing nothing.

### `cmcp server edit`

Opens `$EDITOR` on `~/.config/cmcp/mcp.json`. Creates the file with an
empty `{"mcpServers":{}}` skeleton if it didn't exist.

## Inspecting tools

### `cmcp tool list <server>`

Connects, sends `initialize`, lists all tools, disconnects. `--json`
prints the full JSON Schemas.

### `cmcp tool show <server> <tool>`

Full schema for one tool — useful when you're figuring out what args to
pass:

```sh
cmcp tool show cortex list_nodes
```

```
tool:        list_nodes
description: List nodes by type and/or status filter.
input_schema:
{
  "type": "object",
  "properties": {
    "type":   {"enum": ["REQ","SPEC","TASK","BUG"]},
    "status": {"enum": ["PENDING","IN_PROGRESS","DONE", ...]}
  }
}
```

## Config

### `cmcp config sources`

Prints every path cmcp looks at and whether it exists. First thing to
run when cmcp "doesn't see" a server you expect.

```sh
cmcp config sources --json | jq
```

## Exit codes

| Code | Meaning                              |
|-----:|--------------------------------------|
| 0    | success                              |
| 1    | generic IO / JSON error              |
| 2    | config or invalid-URI error          |
| 3    | server not found / transport failure |
| 4    | tool not found / tool reported error |
| 5    | timeout                              |

Handy for shell scripts:

```sh
if cmcp my-server/health --timeout 5 >/dev/null 2>&1; then
  echo "up"
else
  echo "down (exit $?)"
fi
```

## Timeouts

Default: 30 seconds for both connect and call. Override with `--timeout N`
(seconds). Applies to the whole operation — connect + handshake + call.

Long-running tools should return early and expose their own progress
channel; cmcp is not designed for multi-minute calls.
