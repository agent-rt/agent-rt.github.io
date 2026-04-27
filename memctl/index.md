---
title: memctl
---

# memctl

`memctl` is the **persistent agent memory layer** — cross-tool, cross-project, cross-session. It fills the gap between ephemeral session context (lost at `/clear`), static `CLAUDE.md` files (no structure, no time line), and tool-private memories (Claude Code's auto-memory, Cursor's rules) that don't follow you across tools.

```sh
# Capture an observation
memctl save --type decision --topic skillctl-design \
  "core/extra naming wins over baseline/on-demand: more human-friendly"

# Read it from a different project, different agent, different session:
memctl read --topic skillctl-design --format tsv
memctl search "naming"
```

## Why agent-first

`memctl` is shared infrastructure for AI agents that need durable observations:

- **Cross-tool** — `~/.memctl/` is read via four shell commands documented in `AGENTS.md`. Any agent that can read `AGENTS.md` and execute a shell can participate. Claude Code, Codex, Cursor, Cline — all see the same memory.
- **Cross-project** — `global` scope is the default. Topics like `rust-error-patterns` or `prompt-cache-tricks` follow you across repositories.
- **Per-project commits** — `project` scope writes to `<repo>/.memctl/topics/`, which you commit. New team members get the lore on `git clone` — something tool-private memories cannot do.
- **Append-only timeline** — every entry has a timestamp and a source (`agent-name @ project-path`). Auditable, reviewable, never silently overwritten.
- **Plain markdown** — no SQLite, no vector embedding service, no proprietary format. `cat ~/.memctl/global/topics/X.md` reads the truth.

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

Agents can filter on type: `memctl read --topic X --type decision` returns only design decisions.

## Three scopes

| Scope | Path | Visible to |
|---|---|---|
| `global` (default) | `~/.memctl/global/topics/` | All projects, all agents |
| `project` | `<project>/.memctl/topics/` | Only this repo (commit it!) |
| `agent:<name>` | `~/.memctl/agents/<name>/topics/` | Only that named agent |

`memctl list` (no flag) merges all visible scopes; `--scope project` narrows to one.

## Position in the Agent-RT family

| Tool | Layer |
|---|---|
| [`skillctl`](/skillctl/) | Curated capability bundles |
| **`memctl`** | Persistent memory (this) |
| [`acpctl`](/acpctl/) | ACP agent invocation |
| [`mcpctl`](/mcpctl/) | MCP server invocation |

`skillctl` and `memctl` install independent `AGENTS.md` managed blocks; they coexist without conflict.

## Get started

1. **[Install](/memctl/install/)** — Homebrew or `cargo install`.
2. **[Quickstart](/memctl/quickstart/)** — capture and retrieve your first observation in one minute.
3. **[Commands](/memctl/commands/)** — full reference for all 12 subcommands.

## Links

- Source: [github.com/agent-rt/memctl](https://github.com/agent-rt/memctl)
- Releases: [github.com/agent-rt/memctl/releases](https://github.com/agent-rt/memctl/releases)
- Issues: [github.com/agent-rt/memctl/issues](https://github.com/agent-rt/memctl/issues)
- Spec: [`REQ.md`](https://github.com/agent-rt/memctl/blob/main/REQ.md)
