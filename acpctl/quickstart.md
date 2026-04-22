---
title: Quickstart · acpctl
---

# Quickstart

This assumes an ACP-capable agent is installed. Examples use Google's
`gemini-cli` with `--experimental-acp`; any ACP agent works (codex, goose,
claude-acp, etc.).

## 30-second one-shot

```sh
# 1. Register the agent (JSON spec avoids shell-quoting pain).
acpctl config add gemini --spec '{"command":"gemini","args":["--experimental-acp"]}'

# 2. Introspect — returns the agent's declared capabilities.
acpctl inspect gemini

# 3. One-shot prompt. Default format auto-switches to JSON when piped.
acpctl call gemini "say pong"
acpctl call gemini "summarise" --file README.md
acpctl --format json call gemini "reply ok" | jq
```

A `call` spawns the agent fresh, runs `initialize` → `new_session` →
`prompt`, then exits. Good for stateless tool use.

## Two-minute cross-CLI conversation

For turn-by-turn conversations that survive between CLI invocations, use
the session daemon.

```sh
# 1. Start the daemon (idempotent; auto-starts on first session prompt too).
acpctl daemon start

# 2. Register a persistent session.
acpctl session new gemini --name chat

# 3. First prompt. The agent subprocess launches and stays alive.
acpctl session prompt chat "remember: password=hunter2"

# 4. Second prompt, NEW CLI invocation. Daemon routes to the same agent.
acpctl session prompt chat "what was the password?"
# -> hunter2

# 5. Observe daemon + session state.
acpctl daemon status
acpctl session list

# 6. Clean up.
acpctl session remove chat
acpctl daemon stop
```

> **Why a daemon?** Some ACP agents (notably `gemini-cli`) only remember
> `sessionId`s within a single process. Without the daemon, each CLI call
> starts over. With it, the agent subprocess lives exactly as long as the
> session — cross-invocation context is automatic.

## Agent-facing consumption

Everything below is how an AI agent calling `acpctl` should use it.

### Self-introspection

```sh
acpctl schema | jq
```

Returns every command, its args, stream / single-result type, output
shape, error codes, exit codes, env vars. Parse once, never read `--help`
again.

### Structured errors

All errors are stable JSON on stderr:

```sh
$ acpctl session prompt nonexistent "hi"
# stderr:
{"error":{"code":"unknown_session","message":"unknown session: nonexistent","hint":"run `acpctl session list` to see known sessions"}}
# exit 1
```

Branch on `.error.code`, not on prose.

### Stream contract

`call`, `session prompt`, `session tail` emit NDJSON:

```json
{"type":"notification","data":{ /* ACP SessionNotification */ }}
{"type":"notification","data":{ /* ... */ }}
{"type":"result","data":{"sessionId":"...","stopReason":"end_turn"}}
```

Parse until you see `"type":"result"` or `"type":"error"` — that's your
terminal. `session tail` streams indefinitely until the session closes.

### Token-lean by default

The default filter drops `agent_thought_chunk` and
`available_commands_update`. Real-world Gemini numbers:

| Mode              | Lines | Bytes |
|-------------------|------:|------:|
| Default           |     3 |   516 |
| `--verbose-events`|     7 |  3830 |

That's a **7.4× byte reduction** with no loss of user-visible content.

### Piping structured prompts

Agents rarely have clean multi-line prompts to pass on the command line.
Three options:

```sh
# stdin
printf 'Review the diff below:\n%s' "$DIFF" | acpctl call gemini --prompt-file -

# file
acpctl call gemini --prompt-file /tmp/prompt.txt

# structured ContentBlock array
acpctl call gemini --prompt-json '[
  {"type":"text","text":"Summarise:"},
  {"type":"resource","resource":{"type":"text_resource_contents","uri":"file:///tmp/x.md","text":"..."}}
]'
```

## Next

- **[Configuration](/acpctl/configuration/)** — where config and state live.
- **[Commands](/acpctl/commands/)** — the full reference.
