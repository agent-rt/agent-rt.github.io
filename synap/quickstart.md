---
title: Quickstart · Synap
---

# Quickstart

Five minutes from zero to an agent with persistent memory and a task app.
Uses Claude Desktop; other MCP clients work identically — see
[MCP clients](/synap/mcp-clients/).

## 1. Install

```sh
brew tap agent-rt/tap
brew install synap
```

More options: [Install](/synap/install/).

## 2. Register with Claude Desktop

Edit `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "synap": {
      "command": "synap",
      "args": ["mcp"]
    }
  }
}
```

Restart Claude Desktop. A 🔌 icon in the message composer means Synap is
connected.

## 3. Teach it something

> Use synap to remember that I prefer Rust over Python for backend work.

Claude calls `mcp__synap__remember` — a memory attribute set on subject
`"user"`. Then:

> What do you know about me?

Claude calls `mcp__synap__query` on `synap.memories` and reads back
`preferred_languages = ["Rust"]`.

## 4. Create an app

> Make a synap app called `tasks` with fields: title (text), status
> (enum: todo/doing/done), due (date).

Claude calls `mcp__synap__init_app`. The app is ready immediately — no
migration step.

```
Add a task: "Ship 0.1.0" due 2026-05-01.
```

Claude calls `mcp__synap__write`.

```
Show me my todos.
```

Claude calls `mcp__synap__query` with filters. Results come back in dense
TSV.

## 5. Semantic search

> Remember: Rust's `Arc<Mutex<T>>` is for shared mutable state across threads.
> Remember: Go channels are preferred over shared memory for communication.

Later:

> What notes do I have on concurrency in systems languages?

Claude calls `mcp__synap__search`. Both entries come back, ranked by embedding
similarity. Long text is auto-chunked; results dedupe by entity.

## 6. (Optional) Local agent UI

A local ChatUI that wires Synap tools into OpenAI-compatible or ACP
(Codex / Zed) backends:

```sh
synap agent --port 3000
```

Browse <http://localhost:3000>. Config lives at `~/.synap/agent.toml` —
see [Configuration](/synap/config/).

## What's next

- [Recipes](/synap/recipes/) — build a knowledge base, task board, or
  accounting app end-to-end.
- [Concepts](/synap/concepts/) — the mental model behind apps, memory,
  and search.
- [Reference](/synap/reference/) — every MCP tool, one page.
