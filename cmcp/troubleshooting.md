---
title: Troubleshooting · cmcp
---

# Troubleshooting

## `stdio handshake failed: connection closed: initialize response`

cmcp connected the stdio pipes, spoke `initialize`, and the child closed
its stdout before responding — or responded with something that wasn't
valid JSON-RPC.

Run with `-v` to surface the child's stderr:

```sh
cmcp -v tool list <server>
```

Common causes:

1. **The server writes logs to stdout.** This is the big one. stdio MCP
   servers must emit **only JSON-RPC on stdout**. Any stray text —
   tracing output, banners, `println!` debugging — poisons the channel.

   If you control the server (Rust / `tracing_subscriber`):

   ```rust
   tracing_subscriber::fmt()
       .with_writer(std::io::stderr)   // ← not stdout
       .with_ansi(false)
       .init();
   ```

   Node servers should `console.error` instead of `console.log`.

2. **The command isn't on PATH.** GUI-launched processes sometimes have
   a different PATH than your shell. Use absolute paths in the config:
   `"command": "/usr/local/bin/my-server"`.

3. **Auth / env missing.** HTTP servers often need an `Authorization`
   header; stdio servers often need an API key in `env`. cmcp inherits
   the parent env but only passes along explicit `env` entries from the
   config.

## `server 'x' not found`

```sh
cmcp config sources
```

Confirms cmcp is reading the config file you think it is. If your server
lives in a source marked `exists: no`, that file doesn't exist — the
Agent hasn't actually saved it there yet.

Names are case-sensitive and must match `[A-Za-z0-9_-]+`.

## `tool 'x' not found on server 'y'`

```sh
cmcp tool list y
```

Shows the exact names the server exposes. Some servers expose dynamic
tool sets — reconnect after the server state changes.

## Timeouts

Defaults to 30s for the entire connect + call. For slow-starting stdio
servers (model downloads, DB migrations, etc.):

```sh
cmcp --timeout 120 my-slow-server/initialize_things
```

The per-operation timeout also covers `cmcp tool list` — increase it if
your server takes a while to become ready.

## cmcp can't find Claude Desktop's config on Linux

cmcp uses the macOS path for Claude Desktop
(`~/Library/Application Support/Claude/...`). Linux is handled as a
best-effort fallback to `~/.config/Claude/claude_desktop_config.json` —
only exists if you've manually created it.

Keep your dev servers in either Claude Code (`~/.claude.json`) or cmcp's
own config (`~/.config/cmcp/mcp.json`).

## HTTP server returns 401

cmcp passes through the `Authorization` header verbatim. Double-check:

```sh
cmcp server show my-api --reveal
```

Ensure the header value is the full `Bearer xxxx` (or equivalent scheme),
not just the token. For OAuth flows with automatic refresh: not yet
supported — that's on the roadmap.

## I want cmcp to connect once and reuse the session

Not in 0.1.x. cmcp is one-shot by design, matching the curl mental
model. A long-lived daemon mode is on the roadmap; if you need
sub-100ms repeated calls today, embed `rmcp` directly in your own
tool.

## Reporting bugs

Open an issue at
[github.com/agent-rt/cmcp/issues](https://github.com/agent-rt/cmcp/issues).
Include:

- `cmcp --version`
- The command you ran, with `-v`
- The full error message
- If stdio: the relevant snippet from `cmcp server show <name> --reveal`
  (redact tokens first)
