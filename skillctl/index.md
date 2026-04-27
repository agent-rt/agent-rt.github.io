---
title: skillctl
---

# skillctl

`skillctl` is a **profile-driven Agent skill manager and protocol gateway**. It lets users who already know which skills they want apply curated bundles to a project in one command, while exposing a stable shell-level protocol that any AI agent (Claude Code, Codex, Cursor, …) can consume through four commands documented in `AGENTS.md`.

```sh
# Curate once
skillctl profile add github:agent-rt/skills/profiles/rust-cli.toml --as rust-cli

# Apply to a project
cd my-rust-cli
skillctl init rust-cli       # writes skillctl.toml, skillctl.lock, AGENTS.md

# Agent perspective (per AGENTS.md):
skillctl list --project --format tsv          # 4-col catalog
skillctl describe rust/review --format json   # structured metadata
skillctl show rust/review                     # full SKILL.md
```

## Why agent-first

`skillctl` is opinionated for users who **explicitly curate** their AI agent's capabilities, not for agents that scan filesystems and guess:

- **Profile-first** — define a skill bundle once (`rust-cli`, `tauri-app`, `nextjs-saas`), apply with `skillctl init <profile>`. Sets up `skillctl.toml`, lock file, and `AGENTS.md` block in one shot.
- **Mandatory `namespace/name` IDs** — collisions are syntactically impossible. `myorg/deploy` and `acme/deploy` coexist; agents always know which they got.
- **Stable `AGENTS.md` bytes** — the managed block depends only on protocol version, not on the current skill list. Adding/removing skills never invalidates prompt cache.
- **TSV output for agents** — `list --project --format tsv` returns 4 columns (`tier / id / summary / triggers`), about 3× more compact than JSON for the same information.
- **Reproducible** — `skillctl.lock` pins exact versions and checksums; `git clone && skillctl restore` gives every team member identical agent context.

## Position in the Agent-RT family

| Tool | Layer |
|---|---|
| **`skillctl`** | Curated capability bundles (skills + profiles) |
| [`memoryctl`](/memoryctl/) | Persistent agent memory (lessons / decisions / facts) |
| [`acpctl`](/acpctl/) | ACP agent invocation |
| [`mcpctl`](/mcpctl/) | MCP server invocation |

`skillctl` and `memoryctl` are independent — same `AGENTS.md` can host both managed blocks; either works without the other.

## Get started

1. **[Install](/skillctl/install/)** — Homebrew or `cargo install`.
2. **[Quickstart](/skillctl/quickstart/)** — your first profile, your first project, in under two minutes.
3. **[Commands](/skillctl/commands/)** — reference for all 18 subcommands.

## Links

- Source: [github.com/agent-rt/skillctl](https://github.com/agent-rt/skillctl)
- Releases: [github.com/agent-rt/skillctl/releases](https://github.com/agent-rt/skillctl/releases)
- Issues: [github.com/agent-rt/skillctl/issues](https://github.com/agent-rt/skillctl/issues)
- Specs: [`REQ.md`](https://github.com/agent-rt/skillctl/blob/main/REQ.md) · [`PROTOCOL.md`](https://github.com/agent-rt/skillctl/blob/main/PROTOCOL.md)
