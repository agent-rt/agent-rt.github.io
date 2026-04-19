---
title: Configuration · Synap
---

# Configuration

Synap's core MCP server (`synap mcp`) needs no config — it auto-starts the
daemon and stores everything under `~/.synap/`. This page covers the
**Agent web UI** (`synap agent`) and the opt-in **daemon HTTP dashboard**.

Config lives at `~/.synap/agent.toml`. A fully annotated starter ships in
the distribution tarball as `agent.toml.example`.

## Environment variables

| Variable          | Effect                                           |
|-------------------|--------------------------------------------------|
| `SYNAP_DATA_DIR`  | Override `~/.synap/` (all runtime files)         |
| `SYNAP_HTTP_PORT` | Legacy dashboard port override (deprecated)      |

## Top-level sections

```toml
[agent]
backend = "api"     # "api" (OpenAI-compatible) or "acp"
port    = 3000      # web UI port

[agent.api]         # when backend = "api"
base_url = "https://api.openai.com/v1"
model    = "gpt-4o"
api_key  = "sk-..."

[agent.acp]         # when backend = "acp"
command         = "codex"
args            = ["acp"]
workspace_root  = "~/.synap/sessions"   # per-session cwd parent

[[agent.acp.mcp_servers]]
name    = "synap"
command = "synap"
args    = ["mcp"]
# REQUIRED: without this, the sub-agent has no access to Synap tools.

[agent.memory]
subject                  = "user"       # default attribute subject
event_digest_max         = 8            # events injected into [memory: events]
decay_half_life_days     = 14
decay_weight             = 1.0

[context]                               # LCM phase 1
soft_token_threshold     = 32000
hard_token_threshold     = 64000
preserve_recent_turns    = 10
compact_block_size       = 20
fold_tool_results_after_turns = 6       # phase 2
fold_tool_result_size_bytes   = 500

[compaction]                            # optional: separate backend for compaction
# backend = "api"
# [compaction.api]
# base_url = ...
# model    = "gpt-4o-mini"

[slack]                                 # requires --features slack at build
app_token   = "xapp-..."
bot_token   = "xoxb-..."

[daemon.dashboard]                      # opt-in HTTP dashboard
enabled = false
port    = 3801
bind    = "127.0.0.1"
```

## Backend selection

**API backend** (OpenAI-compatible) — streams SSE from an OpenAI-style
`/chat/completions` endpoint. Works against OpenAI, vLLM, Ollama, LM
Studio, OpenRouter, Together, Groq, etc. Synap executes tools directly
against the local core.

**ACP backend** — delegates the turn to an Agent Client Protocol sub-agent
(Codex, Zed, …). Synap advertises only Synap-specific context (a `[synap
context]` leading block carrying memory + skills + usage guide). All tool
calls route through `mcp_servers=[synap]`, so the sub-agent sees
`mcp__synap__query` / `mcp__synap__remember` / … natively. Each Synap
session maps 1:1 to an ACP session with its own cwd
`<workspace_root>/<synap_sid>/workspace/`.

## Daemon dashboard

Off by default to avoid colliding with the agent port. Set
`[daemon.dashboard] enabled = true` and restart the daemon. `synap daemon
status` prints the bound URL when it's up.

## Slack channel adapter

Requires a Slack app with Socket Mode enabled — no public HTTP endpoint
needed. Set `app_token` / `bot_token` and start `synap agent`. Multimodal
uploads (images / PDFs / text) land under the session's
`workspace/attachments/`. See the tarball's `agent.toml.example` for the
full Slack scope list.
