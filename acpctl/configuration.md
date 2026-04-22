---
title: Configuration · acpctl
---

# Configuration

`acpctl` keeps all state in one directory. There is no magic auto-discovery
of other tools' configs (unlike [mcpctl](/mcpctl/)) — ACP agents are
registered explicitly.

## Paths

macOS:

```
~/Library/Application Support/dev.acpctl.acpctl/
├── config.json              # registered agents
├── sessions/
│   ├── <name>.json          # session metadata
│   └── <name>.jsonl         # per-session event log
└── runtime/
    ├── acpctl.sock          # UDS the daemon listens on (0600)
    ├── acpctl.pid           # daemon PID
    └── acpctl.log           # daemon tracing output
```

Linux follows the XDG Base Directory spec:

```
~/.config/acpctl/config.json
~/.local/share/acpctl/{sessions,runtime}/
```

Run `acpctl config path` to print the resolved `config.json` location.

## Agent specs

Each entry in `config.json` is an `AgentSpec`:

```json
{
  "agents": {
    "gemini": {
      "command": "gemini",
      "args": ["--experimental-acp"],
      "env": {},
      "cwd": null,
      "description": "Google Gemini CLI via experimental ACP transport"
    }
  }
}
```

### Register (agent-friendly, JSON)

```sh
acpctl config add gemini --spec '{"command":"gemini","args":["--experimental-acp"]}'
```

### Register (CLI ergonomic)

```sh
acpctl config add gemini \
  --command gemini \
  --arg --experimental-acp \
  --env GEMINI_API_KEY=sk-... \
  --description "Google Gemini CLI"
```

`--arg` is repeatable — each one becomes a single argument. This avoids
the classic `--args "--foo"` clap quoting trap.

### Inspect

```sh
acpctl config list               # table (TTY) / {"agents":{...}} (JSON)
acpctl --format json config list | jq '.agents | keys'
```

### Remove

```sh
acpctl config remove gemini
# -> {"action":"config_remove","name":"gemini","state":"removed"}
```

## Permissions (`--approve`)

Every `call` / `session prompt` takes `--approve`. Policy is locked in on
the first `session open` for daemon sessions.

| Value         | `fs/read` | `fs/write` | `terminal/*` | `session/request_permission` |
|---------------|:---------:|:----------:|:------------:|:----------------------------:|
| `read-only`   | ✓         | —          | —            | auto-cancel                  |
| `allow-fs`    | ✓         | ✓          | —            | auto-accept first option     |
| `allow-all`   | ✓         | ✓          | ✓            | auto-accept first option     |
| `deny-all`    | —         | —          | —            | auto-cancel                  |

`read-only` is the default and the safest baseline for new agents.
`allow-all` should only be used when the wrapping harness fully trusts
the ACP agent.

## Output format resolution

`acpctl` decides `text` vs `json` in this order:

1. `ACPCTL_FORMAT=text|json|quiet` env var (highest priority).
2. `--format text|json|quiet` flag.
3. TTY autodetect: stdout is a TTY → `text`, otherwise `json`.

For reproducible agent pipelines, always set `ACPCTL_FORMAT=json` in your
execution environment.

## Event filtering

By default, `acpctl` drops two high-volume event types that add noise but
no content:

- `agent_thought_chunk` — chain-of-thought reasoning
- `available_commands_update` — agent-side slash command registry

Override with `--verbose-events` on any streaming command. Cap individual
events with `--max-chunk-bytes N` (default 4096; 0 disables). When text is
truncated the outer notification gets `_meta.truncated=true`.

## The session daemon

`acpctl` uses a long-lived daemon to hold ACP agent subprocesses alive
across CLI invocations.

### Lifecycle

```sh
acpctl daemon start       # idempotent; auto-forks on first session prompt
acpctl daemon status
acpctl daemon stop
acpctl daemon restart
```

The daemon:

- Binds `runtime/acpctl.sock` with mode `0600`.
- Writes PID to `runtime/acpctl.pid`.
- Sends all tracing output to `runtime/acpctl.log`.
- Closes agent subprocesses whose `last_activity` exceeds **1800 seconds**
  (default idle timeout). The sweeper runs every 60 s.
- Handles `SIGINT` / `SIGTERM` with a graceful shutdown (closes all open
  sessions, unlinks socket + PID file).

### Bypassing the daemon

```sh
acpctl session prompt chat "..." --no-daemon
```

`--no-daemon` spawns a fresh subprocess per invocation. Only useful for
agents with a real cross-process `session/load` (most don't); gemini will
error out immediately because its `sessionId` is process-scoped.

## Environment variables

| Name              | Meaning                                                   |
|-------------------|-----------------------------------------------------------|
| `ACPCTL_FORMAT`   | `text` / `json` / `quiet`. Overrides `--format` and TTY.  |
| `RUST_LOG`        | Standard `tracing_subscriber::EnvFilter` spec. `-v`/`-vv` still work. |
