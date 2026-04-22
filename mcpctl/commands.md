---
title: Commands · mcpctl
---

# Command Reference

```text
mcpctl [--json] [--verbose] <URI> [--arg k=v]... [--args-json JSON] [--timeout N]
mcpctl mcp://<server>/<tool> ...             # explicit URI form
mcpctl call mcp://<server>/<tool> ...        # subcommand form

mcpctl server list [--probe] [--json]
mcpctl server show <name> [--reveal] [--json]
mcpctl server check <name> [--json]
mcpctl server add <name> ( --command <c> [--arg <a>]... [--env K=V]...
                         | --url <u> [--header 'K: V']... ) [--force]
mcpctl server remove <name>
mcpctl server edit

mcpctl tool list <server> [--json] [--timeout N]
mcpctl tool show <server>/<tool> [--signature] [--json] [--timeout N]

mcpctl prompt list <server> [--json] [--timeout N]
mcpctl prompt get <server> <prompt> [--arg k=v]... [--json] [--timeout N]

mcpctl resource list <server> [--json] [--timeout N]
mcpctl resource read <server> <uri> [--json] [--timeout N]

mcpctl introspect <server> [--signature] [--json] [--timeout N]

mcpctl tail <server> [--filter <kind>] [--count N] [--json] [--timeout N]

mcpctl config sources [--json]
```

Global flags (apply to all subcommands):

- `--json` — all output as structured JSON; errors on stderr as `{"error":"...","code":N}`.
- `-v, --verbose` — forward server stderr to the terminal with a `[mcp-stderr]` prefix.
- `--override-cmd <path>` — replace the server's command with a local binary for testing, without editing config.

## Calling a tool

Two equivalent forms:

```sh
mcpctl cortex/list_projects
mcpctl mcp://cortex/list_projects
```

Any other URI scheme is an error:

```sh
mcpctl http://a/b
# mcpctl: invalid mcp uri: only mcp:// scheme is supported, got 'http://a/b'
```

### Passing arguments

```sh
# --arg key=value (repeatable). Values are parsed as JSON when possible, falling back to string.
mcpctl github/search_repos --arg query=rust --arg limit=5 --arg public=true

# --args-json as base; --arg entries override individual keys.
mcpctl github/search_repos \
  --args-json '{"query":"base","limit":1}' \
  --arg limit=5

# Read args from a file or stdin.
mcpctl github/search_repos --args-json @params.json
mcpctl github/search_repos --args-json -    # reads from stdin
```

Arguments are validated by the server, not by mcpctl. If the server rejects
them, the error comes back in the response content.

### Output

Default: human-friendly text. With `--json`: raw `CallToolResult` JSON.

```sh
mcpctl --json github/search_repos --arg query=rust \
  | jq '.content[0].text' -r
```

## Server commands

### `mcpctl server list`

Lists every MCP server across all discovered sources after precedence resolution.

```
SERVER                   SOURCE           TRANSPORT
cortex                   claude-code      stdio: cortex mcp
github                   claude-desktop   stdio: npx -y …
my-dev                   mcpctl           stdio: node server.js
```

`--probe` runs a parallel health check (handshake + list_tools) and adds a
status column. Useful for agents to verify liveness before calling.

### `mcpctl server show <name>`

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

### `mcpctl server check <name>`

Per-stage diagnostic with timing — connect, initialize, list_tools. Useful for
agents to diagnose a failing server before retrying a tool call.

### `mcpctl server add`

Write a new entry into `~/.config/mcpctl/mcp.json`.

```sh
# stdio
mcpctl server add local-dev \
  --command /path/to/my-server \
  --arg --port --arg 9001 \
  --env RUST_LOG=debug

# http
mcpctl server add remote-api \
  --url https://mcp.example.com/mcp \
  --header 'Authorization: Bearer xxxxxxxx'
```

- `--command` and `--url` are mutually exclusive.
- Duplicate names error unless you pass `--force`.
- Only mcpctl's own file is ever modified.

### `mcpctl server remove <name>`

Removes the entry from mcpctl's own config. If the name only exists in a
read-only source, mcpctl tells you which file to edit and exits non-zero.

### `mcpctl server edit`

Opens `$EDITOR` on `~/.config/mcpctl/mcp.json`. Creates the file with an
empty `{"mcpServers":{}}` skeleton if it didn't exist.

## Tool commands

### `mcpctl tool list <server>`

Connects, lists all tools, disconnects. `--json` returns the full JSON Schemas.

### `mcpctl tool show <server>/<tool>`

Full schema for one tool.

```sh
mcpctl tool show cortex/list_nodes
```

`--signature` prints a compact agent-readable one-liner instead of the full schema:

```
list_nodes(type?: enum, status?: enum) → content
```

## Prompt commands

### `mcpctl prompt list <server>`

Lists all prompts the server exposes.

### `mcpctl prompt get <server> <prompt>`

Retrieves a rendered prompt, optionally with arguments:

```sh
mcpctl prompt get my-server summarize --arg topic=rust --arg length=short
```

With `--json`: returns the full `GetPromptResult` including role and content blocks.

## Resource commands

### `mcpctl resource list <server>`

Lists all resources the server exposes (URIs + metadata).

### `mcpctl resource read <server> <uri>`

Reads one resource by URI:

```sh
mcpctl resource read my-server file:///path/to/doc.md
```

## Introspect

### `mcpctl introspect <server>`

Tools + prompts + resources in a single handshake — the most token-efficient
way for an agent to map a server's full surface before deciding what to call.

```sh
mcpctl introspect cortex --json
mcpctl introspect cortex --signature   # compact tool signatures only
```

## Tail

### `mcpctl tail <server>`

Streams server-push notifications in real time. Useful for monitoring
progress on long-running tool calls.

```sh
mcpctl tail my-server                        # stream all notifications
mcpctl tail my-server --filter progress      # filter by kind
mcpctl tail my-server --count 10 --json      # stop after 10, emit ndjson
```

## Config

### `mcpctl config sources`

Prints every path mcpctl looks at and whether it exists. First thing to run
when mcpctl "doesn't see" a server you expect.

```sh
mcpctl config sources --json | jq
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
if mcpctl my-server/health --timeout 5 >/dev/null 2>&1; then
  echo "up"
else
  echo "down (exit $?)"
fi
```

## Timeouts

Default: 30 seconds for the entire connect + handshake + call. Override
with `--timeout N` (seconds). Applies to all operations including
`tool list`, `introspect`, and `tail`.
