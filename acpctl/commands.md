---
title: Commands · acpctl
---

# Commands

Full reference. For a machine-readable version run:

```sh
acpctl schema | jq
```

## Global flags

| Flag              | Meaning                                                      |
|-------------------|--------------------------------------------------------------|
| `--format <F>`    | `text` / `json` / `quiet`. Default: text on TTY, json when piped. |
| `-v`, `-vv`       | Bump tracing level (warn → info → debug → trace).            |
| `--version`       | Print version and exit.                                      |

Env: `ACPCTL_FORMAT` overrides `--format`.

## Registry

### `acpctl list`

Enumerate agents in the ACP public registry.

```sh
acpctl list --refresh           # force re-fetch
acpctl --format json list
```

### `acpctl search <query>`

```sh
acpctl search review
```

### `acpctl info <agent_id>`

```sh
acpctl info claude-acp
```

## Direct agent calls

### `acpctl inspect [<agent>] [--command "..."]`

Runs `initialize` against an agent and prints its declared capabilities.
Does not create a session. 15s timeout.

```sh
acpctl inspect gemini
acpctl inspect --command "goose run --acp"
```

Output (JSON mode): the agent's full `InitializeResponse`.

### `acpctl call [<agent>] [<prompt>]`

One-shot: spawn → initialize → new_session → prompt → exit. Streams events
to stdout.

```sh
# Positional prompt.
acpctl call gemini "explain this function"

# Stdin or file.
printf "review this:\n%s" "$DIFF" | acpctl call gemini --prompt-file -
acpctl call gemini --prompt-file /tmp/prompt.txt

# Structured ContentBlock array.
acpctl call gemini --prompt-json '[{"type":"text","text":"hi"}]'

# Attach files as text_resource_contents blocks.
acpctl call gemini "summarise" --file README.md --file CHANGELOG.md

# Permissions + events.
acpctl call gemini --approve allow-fs --verbose-events --max-chunk-bytes 0 "..."
```

Flags:

| Flag                   | Default      | Meaning                                          |
|------------------------|--------------|--------------------------------------------------|
| `--prompt-file <PATH>` | —            | Read prompt text from file. `-` = stdin.         |
| `--prompt-json <STR>`  | —            | JSON `ContentBlock[]`. `-` = stdin.              |
| `--file <PATH>`        | —            | Attach file (repeatable).                        |
| `--command <STR>`      | —            | Ad-hoc command; overrides `<agent>` lookup.      |
| `--cwd <PATH>`         | `$PWD`       | Working directory passed to `session/new`.       |
| `--approve <POLICY>`   | `read-only`  | See [Configuration](/acpctl/configuration/).     |
| `--verbose-events`     | off          | Include thought + available_commands.            |
| `--max-chunk-bytes N`  | `4096`       | Truncation cap per event. `0` disables.          |

Stream (JSON):

```json
{"type":"notification","data":{ /* SessionNotification */ }}
...
{"type":"result","data":{"sessionId":"...","stopReason":"end_turn"}}
```

Exit codes: 0 success, 1 user error, 2 protocol/agent, 3 daemon/io, 4 other.

## Sessions

### `acpctl session new <agent> --name <NAME> [--cwd <PATH>]`

Register a persistent session.

```sh
acpctl session new gemini --name research
# -> {"name":"research","agent":"gemini","sessionId":"...","cwd":"...","createdAt":"..."}
```

### `acpctl session list`

```sh
acpctl --format json session list    # {"sessions":[...]}
acpctl session list                  # TTY table
```

### `acpctl session show <name>`

Metadata + JSONL event log.

```sh
acpctl session show research
```

### `acpctl session prompt <name> [<prompt>]`

Same prompt-resolution rules as `call`. Routes through the daemon (auto-starts
if needed) unless `--no-daemon`.

```sh
acpctl session prompt research "what did we decide last time?"
acpctl session prompt research --prompt-file next.md
acpctl session prompt research --no-daemon "..."   # skip daemon (rarely useful)
```

Extra flags: `--file`, `--approve`, `--verbose-events`, `--max-chunk-bytes`.

### `acpctl session tail <name>`

Subscribe to another prompt's stream without sending one of your own. Runs
until the session is closed or Ctrl-C.

```sh
# Terminal A: long-running prompt.
acpctl session prompt research "generate 2000-word essay"

# Terminal B: observe events in real time.
acpctl --format json session tail research | jq .data.update.sessionUpdate
```

### `acpctl session cancel <name>`

Propagate `session/cancel` notification to the agent. Current prompt
returns `stopReason: "cancelled"`.

### `acpctl session close <name>`

Stop the agent subprocess; keep session metadata for later re-open.
Next `session prompt` will re-open (context may be lost, agent-dependent).

### `acpctl session remove <name>`

Close + delete metadata + delete `.jsonl`. Idempotent on a closed session.

### `--no-daemon`

`session prompt --no-daemon` spawns a fresh subprocess per call. Fine for
agents with true cross-process `session/load`; most (including
`gemini-cli`) will error with "invalid session identifier".

## Config

### `acpctl config add <name> [options]`

```sh
# Agent-friendly: JSON spec in one shot.
acpctl config add gemini --spec '{"command":"gemini","args":["--experimental-acp"]}'

# Human-friendly: flags.
acpctl config add gemini --command gemini --arg --experimental-acp
```

| Flag                   | Meaning                                                        |
|------------------------|----------------------------------------------------------------|
| `--spec <JSON>`        | Full `AgentSpec` JSON. Wins over all other flags.              |
| `--command <STR>`      | Executable path.                                               |
| `--arg <STR>`          | Single arg; repeatable.                                        |
| `--env <KEY=VAL>`      | Repeatable.                                                    |
| `--cwd <PATH>`         | Working directory hint.                                        |
| `--description <STR>`  | Human note.                                                    |

### `acpctl config list`

```sh
acpctl --format json config list    # {"agents":{...}}
```

### `acpctl config remove <name>`

### `acpctl config path`

```sh
acpctl config path
# -> /Users/.../dev.acpctl.acpctl/config.json (text)
# -> {"path":"/Users/.../config.json"}       (json)
```

## Daemon

### `acpctl daemon start`

Starts a detached daemon if none is running. Idempotent.

Output states: `started`, `already_running`.

### `acpctl daemon stop`

Output states: `stopped`, `sigterm`, `not_running`.

### `acpctl daemon status`

```json
{"pid":12345,"socket":"...","uptime_secs":42,"sessions":[ ... ]}
```

### `acpctl daemon restart`

Graceful stop + wait for socket cleanup + start.

### `acpctl daemon serve [--detached]`

Runs the daemon in the current process. Used by `start` internally; also
useful for `systemd` / `launchd` / foreground debugging.

## Schema

### `acpctl schema`

Emits the complete machine-readable contract: commands, arg lists, stream
vs single, output shape, error codes, exit code classes, env vars, event
filter policy, truncation policy.

```sh
acpctl schema | jq '.exit_codes'
acpctl schema | jq '.error_codes'
acpctl schema | jq '.commands | keys'
```

Schema is additive; existing fields never change meaning.
