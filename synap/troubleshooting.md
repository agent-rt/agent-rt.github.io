---
title: Troubleshooting · Synap
---

# Troubleshooting

## The daemon won't start

Check `~/.synap/synap.log`. Common causes:

- **Stale PID file** — another instance claimed the pidfile but died. Run
  `synap daemon stop`, then `synap daemon start`. If that fails, remove
  `~/.synap/synap.pid` manually.
- **Socket already exists** — `~/.synap/synap.sock` left over. Boot normally
  cleans it up, but a zombie process may still hold it. `lsof
  ~/.synap/synap.sock` to find the holder, kill it, retry.
- **Data dir unwritable** — `ls -la ~/.synap/`; the daemon needs r/w.

## First run takes forever

The embedding model (Qwen3-Embedding-0.6B, ~600 MB GGUF) downloads from
Hugging Face on first use. Subsequent runs are instant. Set
`HF_HUB_OFFLINE=1` after the first run to disable network calls.

## Claude can't see Synap tools

1. Verify the binary is on PATH as seen by the client. GUI apps on macOS
   often don't inherit `/usr/local/bin`. Use the absolute path in the MCP
   config.
2. `synap daemon status` — if not running, the MCP handshake will fail.
   `synap mcp` auto-starts it, but a crashed daemon with a stale lock can
   block that.
3. Check the MCP client's own log for the handshake. The tool list should
   include the 13 `mcp__synap__*` tools.

## Dashboard port collision

If `[daemon.dashboard] enabled = true` and `synap daemon start` fails with
`Address already in use`, change `[daemon.dashboard] port`. Default 3801
was chosen to avoid common agent ports (3000 / 8080 / 11434).

## Agent config not loaded

`synap agent` logs the loaded config path at startup. If edits aren't
applying:

- Confirm `~/.synap/agent.toml` vs `--config` flag path.
- TOML syntax — `[[agent.acp.mcp_servers]]` needs double brackets per
  entry.

## ACP sub-agent has no memory / can't see Synap

Look for this in the agent log:

```
WARN  ACP backend without mcp_servers[synap] — the sub-agent has no access
      to Synap tools.
```

Add the block. Without it the sub-agent (Codex, Zed, …) sees Synap's
injected `[synap context]` snapshot but can't call tools.

## Slack adapter silent

- The `slack` feature must be compiled in. Release tarballs include it.
- Socket Mode must be enabled in the Slack app config.
- Check the agent log for `slack: connected as @…`. If absent, the tokens
  or scopes are wrong. Required: `app_mentions:read`, `channels:history`,
  `chat:write`, `files:read`, `im:history`.

## Database corruption

SQLite + WAL is robust but not invincible. If `synap daemon start` fails
with `database disk image is malformed`:

```sh
synap daemon stop
cp ~/.synap/synap.db ~/.synap/synap.db.bak
sqlite3 ~/.synap/synap.db ".recover" | sqlite3 ~/.synap/synap.db.recovered
mv ~/.synap/synap.db.recovered ~/.synap/synap.db
```

File an issue with `~/.synap/synap.log` attached.

## Reporting issues

- Preview binary — expect rough edges.
- Include: `synap --version`, `synap daemon status`, relevant log tail from
  `~/.synap/synap.log` or `~/.synap/agent.log`.
- Don't paste `agent.toml` directly — it contains API keys. Redact first.
