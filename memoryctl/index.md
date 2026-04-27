---
title: memoryctl
---

# memoryctl

`memoryctl` is the **persistent agent memory layer** — cross-tool, cross-project, cross-session. It fills the gap between ephemeral session context (lost at `/clear`), static `CLAUDE.md` files (no structure, no time line), and tool-private memories (Claude Code's auto-memory, Cursor's rules) that don't follow you across tools.

```sh
# Capture an observation
memoryctl save --type decision --topic skillctl-design \
  "core/extra naming wins over baseline/on-demand: more human-friendly"

# Read it from a different project, different agent, different session:
memoryctl read --topic skillctl-design --format tsv
memoryctl search "naming"
```

## Why agent-first

`memoryctl` is shared infrastructure for AI agents that need durable observations:

- **Cross-tool** — `~/.memoryctl/` is read via four shell commands documented in `AGENTS.md`. Any agent that can read `AGENTS.md` and execute a shell can participate. Claude Code, Codex, Cursor, Cline — all see the same memory.
- **Cross-project** — `global` scope is the default. Topics like `rust-error-patterns` or `prompt-cache-tricks` follow you across repositories.
- **Per-project commits** — `project` scope writes to `<repo>/.memoryctl/topics/`, which you commit. New team members get the lore on `git clone` — something tool-private memories cannot do.
- **Append-only timeline** — every entry has a timestamp and a source (`agent-name @ project-path`). Auditable, reviewable, never silently overwritten.
- **Plain markdown** — no SQLite, no vector embedding service, no proprietary format. `cat ~/.memoryctl/global/topics/X.md` reads the truth.

## Seven types of memory

| Type | Example |
|---|---|
| `lesson` | "thiserror in libs, anyhow in main" |
| `decision` | "core/extra naming wins over baseline/on-demand" |
| `fact` | "elepay clearing flow: Stripe → us → merchant" |
| `feedback` | "user prefers integration tests against real DB, not mocks" |
| `reference` | "INC-tagged bugs live in Linear INGEST project" |
| `user` | "data scientist, prioritizes observability" |
| `project` | "merge freeze starts 2026-03-05, mobile release branch cut" |

Agents can filter on type: `memoryctl read --topic X --type decision` returns only design decisions.

## Three scopes

| Scope | Path | Visible to |
|---|---|---|
| `global` (default) | `~/.memoryctl/global/topics/` | All projects, all agents |
| `project` | `<project>/.memoryctl/topics/` | Only this repo (commit it!) |
| `agent:<name>` | `~/.memoryctl/agents/<name>/topics/` | Only that named agent |

`memoryctl list` (no flag) merges all visible scopes; `--scope project` narrows to one.

## Position in the Agent-RT family

| Tool | Layer |
|---|---|
| [`skillctl`](/skillctl/) | Curated capability bundles |
| **`memoryctl`** | Persistent memory (this) |
| [`acpctl`](/acpctl/) | ACP agent invocation |
| [`mcpctl`](/mcpctl/) | MCP server invocation |

`skillctl` and `memoryctl` install independent `AGENTS.md` managed blocks; they coexist without conflict.

## Get started

1. **[Install](/memoryctl/install/)** — Homebrew or `cargo install`.
2. **[Quickstart](/memoryctl/quickstart/)** — capture and retrieve your first observation in one minute.
3. **[Commands](/memoryctl/commands/)** — full reference for all 12 subcommands.

## Links

- Source: [github.com/agent-rt/memoryctl](https://github.com/agent-rt/memoryctl)
- Releases: [github.com/agent-rt/memoryctl/releases](https://github.com/agent-rt/memoryctl/releases)
- Issues: [github.com/agent-rt/memoryctl/issues](https://github.com/agent-rt/memoryctl/issues)
- Spec: [`REQ.md`](https://github.com/agent-rt/memoryctl/blob/main/REQ.md)
