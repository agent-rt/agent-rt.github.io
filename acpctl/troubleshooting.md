---
title: Troubleshooting · acpctl
---

# Troubleshooting

## Exit code reference

| Code | Class              | Typical cause                                                              |
|-----:|--------------------|----------------------------------------------------------------------------|
| 0    | success            | —                                                                          |
| 1    | user error         | `unknown_agent`, `unknown_session`, `session_exists`, `config_error`, bad JSON |
| 2    | protocol/agent     | `sdk_error` — agent rejected a request, protocol mismatch                  |
| 3    | daemon/io          | `io_error`, `http_error`, daemon not reachable, agent spawn failed         |
| 4    | unexpected         | catch-all internal                                                         |

Parse `.error.code` from the stderr envelope to branch on the exact cause.

## Error codes

From `acpctl schema | jq .error_codes`:

- `io_error`, `json_error`, `http_error`, `sdk_error`
- `unknown_agent`, `unknown_session`, `session_exists`
- `no_distribution`, `config_error`, `other`

Each error carries a stable `code`, a human `message`, and sometimes an
actionable `hint`:

```json
{"error":{"code":"unknown_session","message":"unknown session: foo","hint":"run `acpctl session list` to see known sessions"}}
```

## "daemon not reachable"

```sh
acpctl session prompt chat "hi"
# error: daemon not reachable at .../runtime/acpctl.sock: No such file or directory
```

Causes + fixes:

1. Daemon hasn't started yet — `acpctl daemon start`. (Or just run
   `session prompt` again; it auto-starts.)
2. Daemon crashed — check `runtime/acpctl.log` for the panic / SDK error.
3. Stale socket file — `acpctl daemon start` deletes stale sockets on
   bind; if it still fails, `rm runtime/acpctl.sock && acpctl daemon start`.

## "invalid session identifier"

```sh
acpctl session prompt chat --no-daemon "hi"
# error: ACP SDK error: Internal error: {"details":"Invalid session identifier \"...\""}
```

You're running `--no-daemon` against an agent that only scopes `sessionId`
to a single process (gemini-cli is the canonical example). Drop
`--no-daemon` and let the daemon hold the subprocess alive — that's what
it's for.

## Gemini prints EPIPE on shutdown

```
error: write EPIPE
```

Cosmetic only. `gemini-cli` writes a Node.js unhandled-rejection trace to
its own stderr when `acpctl` closes the stdin pipe. `acpctl` inherits
`stderr` from the child so that noise reaches your terminal. The daemon
and session state are unaffected. Workaround: send `stderr` to `/dev/null`
via your shell if it bothers you.

## `agent_thought_chunk` flood in output

You're running with `--verbose-events`. By default `acpctl` drops thought
chunks and `available_commands_update` — typical savings are 7×–8× bytes.
Remove `--verbose-events` (or check `ACPCTL_FORMAT` / environment flags).

## Oversize events / truncation

```json
{"type":"notification","data":{"update":{...},"_meta":{"truncated":true}}}
```

Event exceeded `--max-chunk-bytes` (default 4096). Raise the cap if you
need full payloads: `--max-chunk-bytes 65536`, or `--max-chunk-bytes 0`
to disable. The `_meta.truncated` marker tells agents to re-fetch or back
off.

## Session context lost after idle

After ~30 minutes of inactivity, the idle sweeper closes the session's
agent subprocess (`"idle timeout reached, closing session"` in
`acpctl.log`). Next `session prompt` auto-reopens — for agents without
true `session/load` this is equivalent to starting a new conversation.

Either bump `RUST_LOG=info` and check the log, or just prompt more often.
A `session new` with a different name and a fresh context is usually
cleaner than trying to stretch one.

## Homebrew formula out of date

The tap [`agent-rt/homebrew-tap`](https://github.com/agent-rt/homebrew-tap)
is updated by the release workflow automatically. If a new GitHub release
exists but `brew upgrade acpctl` still pulls the old version:

```sh
brew update
brew upgrade acpctl
```

If the formula itself is stale, file an issue at
[github.com/agent-rt/acpctl/issues](https://github.com/agent-rt/acpctl/issues).

## Getting help

- `acpctl schema` — machine-readable contract.
- `acpctl --help`, `acpctl <command> --help` — human prose.
- `runtime/acpctl.log` — daemon tracing.
- `sessions/<name>.jsonl` — full event log for any session.
