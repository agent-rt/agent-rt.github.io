---
title: Commands · memctl
---

# Commands

`memctl` ships 12 subcommands. Use `memctl <cmd> --help` for full flag listings.

## Global options

```
--format <human|json|tsv>   default: human
                             json    → structured wire schema
                             tsv     → compact rows, agent-optimized
                             human   → markdown (read) / table (list)
```

When `--format json` is set, errors emit a stable envelope to stdout:

```json
{
  "protocol": 1,
  "success": false,
  "error": "topic_not_found",
  "hint": "topic not found: nonexistent-topic"
}
```

When `--format tsv` is set, errors emit a single line: `ERROR\t<code>\t<hint>`.

## Project lifecycle (human-facing)

| Command | Purpose |
|---|---|
| `memctl init [--force]` | Create `.memctl/topics/` + inject `AGENTS.md` managed block |
| `memctl enable / disable [--target AGENTS.md]` | Manage just the `AGENTS.md` block (independent of `.memctl/`) |

## Capture

| Command | Purpose |
|---|---|
| `memctl save --type T --topic N [content / --from-stdin] [--scope S] [--source ...] [--no-confirm]` | Append a new entry to a topic; auto-creates the topic file |

`--type` must be one of: `lesson`, `decision`, `fact`, `feedback`, `reference`, `user`, `project`.
`--scope` defaults to `global`; other values are `project` and `agent:<name>`.

## Read

| Command | Purpose |
|---|---|
| `memctl list [--scope S] [--type T] [--recent <Nd/Nh>] [--format tsv]` | Topic catalog: name, entry count, last updated, types, scope |
| `memctl topics` | Alias for `list` |
| `memctl read --topic N [--scope S] [--type T] [--since Nd] [--limit N] [--reverse]` | Read entries; default human output is markdown, `--format tsv` for agent consumption |
| `memctl search <query> [--scope S] [--type T] [--max-per-topic N]` | Full-text search across topics; CJK-safe; returns matches with timestamp + snippet |

## Maintain

| Command | Purpose |
|---|---|
| `memctl forget --topic N [--entry N \| --before Nd \| --yes]` | Remove a specific entry (1-indexed by time), all entries before a cutoff, or the entire topic |
| `memctl edit --topic N [--scope S] [--new]` | Open the topic file in `$EDITOR`; auto-validates format on exit |
| `memctl move --topic N --from-scope A --to-scope B` | Migrate a topic between scopes (e.g. `global` → `project` once it stabilizes) |
| `memctl validate [--topic N] [--scope S] [--strict]` | Check topic file format; warns on cosmetic issues, errors on parser-fatal ones |
| `memctl export --topic N [--scope S]` | Print a clean markdown digest grouped by type — useful as the seed for a `SKILL.md` once a topic graduates |

## Wire schemas

The full JSON / TSV schemas are documented in [`REQ.md` §6 and §7](https://github.com/agent-rt/memctl/blob/main/REQ.md).

Topic files on disk are plain markdown (entries delimited by `## <ISO timestamp> [type=X source=Y @ Z]` H2 headers). You can `cat` them, `git diff` them, or edit by hand without going through the CLI.

## Environment variables

| Var | Effect |
|---|---|
| `MEMORYCTL_AGENT` | Agent name recorded in `source=` of every saved entry. Default: `unknown`. Set this in your agent harness so cross-tool attribution works. |
| `EDITOR` | Used by `memctl edit`. Default: `vi`. |
| `RUST_LOG` | Standard `tracing-subscriber` log level. |

## Type taxonomy in detail

| Type | Use when |
|---|---|
| `lesson` | A recurring gotcha, hard-won experience, "next time we'll..." |
| `decision` | A design decision plus its rationale |
| `fact` | Domain knowledge that's not opinion |
| `feedback` | User collaboration preferences ("prefer real DB over mocks") |
| `reference` | A pointer to an external system ("bugs in Linear INGEST") |
| `user` | Identity / role / strengths of the user |
| `project` | Current project state, deadlines, freeze windows |

Agents can filter on any of these via `--type` on `list`, `read`, and `search`.
