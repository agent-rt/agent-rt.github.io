---
title: Troubleshooting · mcpctl
---

# Troubleshooting

## `stdio handshake failed: connection closed: initialize response`

mcpctl connected the stdio pipes, spoke `initialize`, and the child closed
its stdout before responding — or responded with something that wasn't
valid JSON-RPC.

Run with `-v` to surface the child's stderr:

```sh
mcpctl -v tool list <server>
```

Common causes:

1. **The server writes logs to stdout.** stdio MCP servers must emit
   **only JSON-RPC on stdout**. Any stray text — tracing output, banners,
   `println!` debugging — poisons the channel.

   Rust / `tracing_subscriber`:

   ```rust
   tracing_subscriber::fmt()
       .with_writer(std::io::stderr)   // ← not stdout
       .with_ansi(false)
       .init();
   ```

   Node servers should `console.error` instead of `console.log`.

2. **The command isn't on PATH.** GUI-launched processes may have a
   different PATH than your shell. Use absolute paths in the config:
   `"command": "/usr/local/bin/my-server"`.

3. **Auth / env missing.** HTTP servers often need an `Authorization`
   header; stdio servers often need an API key in `env`. mcpctl passes
   only the explicit `env` entries from the config.

## `server 'x' not found`

```sh
mcpctl config sources
```

Confirms mcpctl is reading the config file you expect. If your server
lives in a source marked `exists: no`, that file doesn't exist yet.

Names are case-sensitive and must match the name in the config file exactly.

## `tool 'x' not found on server 'y'`

```sh
mcpctl tool list y
```

Shows the exact tool names the server exposes. Some servers expose dynamic
tool sets — reconnect after server state changes.

## Timeouts

Default is 30s for the entire connect + call. For slow-starting servers:

```sh
mcpctl --timeout 120 my-slow-server/initialize_things
```

The timeout also applies to `tool list`, `introspect`, and `tail`.

## mcpctl can't find Claude Desktop's config on Linux

mcpctl uses the macOS path for Claude Desktop
(`~/Library/Application Support/Claude/...`). On Linux it falls back to
`~/.config/Claude/claude_desktop_config.json` — only present if you've
created it manually.

Keep dev servers in Claude Code (`~/.claude.json`) or mcpctl's own config
(`~/.config/mcpctl/mcp.json`).

## Zed servers aren't appearing

Zed stores its MCP servers under `context_servers` in `settings.json`, not
under `mcpServers`. mcpctl parses this automatically. Check that:

1. The Zed settings file exists (`mcpctl config sources`).
2. Your server entry has a `settings.command` field — servers without a
   `command` are skipped (Zed supports non-stdio servers mcpctl can't launch).

## HTTP server returns 401

mcpctl passes the `Authorization` header verbatim. Verify with:

```sh
mcpctl server show my-api --reveal
```

Ensure the value is the full `Bearer xxxx`, not just the raw token. OAuth
flows with automatic refresh are not yet supported.

## I want mcpctl to connect once and reuse the session

Not in 0.1.x. mcpctl is one-shot by design — stateless, like `curl`. A
daemon mode is on the roadmap; if you need sub-100ms repeated calls today,
embed `rmcp` directly in your own tool.

## Reporting bugs

Open an issue at
[github.com/agent-rt/mcpctl/issues](https://github.com/agent-rt/mcpctl/issues).
Include:

- `mcpctl --version`
- The command you ran (with `-v`)
- The full error message
- If stdio: the relevant snippet from `mcpctl server show <name> --reveal`
  (redact tokens first)
