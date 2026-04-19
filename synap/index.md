---
title: Synap
---

# Synap

A local-first, Rust-powered structured memory engine for AI agents. Synap
exposes an MCP (Model Context Protocol) server that lets agents dynamically
create app namespaces — tasks, notes, accounting, knowledge bases — and
perform hybrid keyword + semantic search across them. Everything lives in a
single SQLite file under `~/.synap/`. No cloud, no accounts, no API keys.

## Why

Most "agent memory" products are cloud SaaS with opaque vector stores. Synap
flips that:

- **Local-first** — one SQLite file, your disk, offline by default.
- **Structured** — EAV schema with field types, filters, and projection.
  Your agent can reason about data, not just retrieve blobs.
- **Hybrid search** — sqlite-vec embeddings (Qwen3-0.6B, runs on CPU) +
  FTS5 keyword. Ranks by relevance, recency, or time-decay.
- **Agent-native** — designed for MCP from day one. Every tool returns
  compact TSV/JSON to fit in context windows.
- **Single binary** — `brew install synap`, done.

## Get started

1. **[Install](/synap/install/)** — Homebrew or one-liner.
2. **[Quickstart](/synap/quickstart/)** — 5 minutes, Claude Desktop walkthrough.
3. **[MCP clients](/synap/mcp-clients/)** — Cursor, Zed, Claude Code.
4. **[Recipes](/synap/recipes/)** — real examples (knowledge base, task
   board, accounting, personal memory).
5. **[Reference](/synap/reference/)** — all MCP tools, one page.

## Concepts at a glance

Synap organizes data into **apps**. An app is a namespace with a schema.
Schemas are EAV under the hood, so agents can add fields without a migration.

Two apps are built in:

- `synap.memories` — long-term memory about the user (profile attributes +
  timestamped events). Pre-injected into every turn.
- `synap.sessions` — conversation history with LCM-style lossless compaction.

User-facing apps (tasks, recipes, expenses, …) are created on demand by the
agent calling `init_app`. See [Concepts](/synap/concepts/) for the full
mental model.

## Configuration

Optional config at `~/.synap/agent.toml` (only needed if you run `synap
agent` for the web UI or Slack). See [Configuration](/synap/config/).

## Links

- Source: [github.com/agent-rt/synap](https://github.com/agent-rt/synap) (private preview)
- Releases: [github.com/agent-rt/homebrew-tap](https://github.com/agent-rt/homebrew-tap)
- Agent Skill: [github.com/agent-rt/skills](https://github.com/agent-rt/skills/tree/main/skills/synap)
