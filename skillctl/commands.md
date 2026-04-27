---
title: Commands · skillctl
---

# Commands

`skillctl` ships 18 subcommands. Use `skillctl <cmd> --help` for full flag listings.

## Global options

```
--format <human|json|tsv>   default: human
                             json   → full wire schema, scripts/tools
                             tsv    → 4-col compact, agent-optimized
```

When `--format json` is set, errors emit a stable envelope to stdout:

```json
{
  "protocol": 1,
  "success": false,
  "error": "ambiguous_skill_name",
  "hint": "Use full id, e.g. namespace/name",
  "matches": ["rust/review", "acme/review"]
}
```

When `--format tsv` is set, errors emit a single line: `ERROR\t<code>\t<hint>`.

## Project lifecycle (human-facing)

| Command | Purpose |
|---|---|
| `skillctl init [profile…] [--link/--force/--suggest]` | Apply one or more profiles to the current directory; writes `skillctl.toml`, `skillctl.lock`, and the `AGENTS.md` block |
| `skillctl use [ids…] [--add/--remove/--clear/--core]` | Tweak the project's enabled skill list; without args, opens a TUI multi-select |
| `skillctl enable / disable [--target AGENTS.md]` | Manage the `AGENTS.md` managed block independently of project init |
| `skillctl trust [path] [--list/--remove]` | Manage the project trust list (only consulted when `SKILLCTL_TRUST_GATE=1`) |

## Global skill management

| Command | Purpose |
|---|---|
| `skillctl add <source> [--as ns/name]` | Install a skill from a local path or `github:user/repo[/sub][@ref]` |
| `skillctl remove <id> [--all-versions/--force]` | Drop a skill from the global store |
| `skillctl update [<id>] [--breaking]` | Refetch from the recorded source and refresh version + checksum |
| `skillctl restore [--global/--project]` | Verify checksums; refetch any skill whose bytes were tampered with or are missing |

## Profile management

| Command | Purpose |
|---|---|
| `skillctl profile add <source> [--as <name>]` | Install a profile (`.toml`) from local path or `github:` URL |
| `skillctl profile list` | List globally installed profiles |
| `skillctl profile show <name>` | Print profile contents |
| `skillctl profile remove <name>` | Drop profile + sidecar |
| `skillctl profile update [<name>]` | Refetch profile from its recorded source |

## Protocol surface (agent-facing)

These four commands are what `AGENTS.md` instructs agents to call. They are **read-only** from the project's perspective and never modify `AGENTS.md`.

| Command | Tier | Token target |
|---|---|---|
| `skillctl list --project --format tsv` | catalog | ≤ 30/skill |
| `skillctl describe <id> --format json` | structured metadata | ≤ 200/skill |
| `skillctl show <id>` | full `SKILL.md` | ≤ 5000/skill |
| `skillctl path <id>` | absolute path to `SKILL.md` | ~80 |

`describe` and `show` resolve short names within the project's manifest. Bare names that are ambiguous return error code `ambiguous_skill_name` with the candidate list — agents can re-call with the full `namespace/name`.

## Diagnostics

| Command | Purpose |
|---|---|
| `skillctl doctor [<id>] [--project]` | Verify external dependencies (binaries via `which`, env vars) declared by skills |
| `skillctl validate [<path>] [--strict]` | Check `SKILL.md` format; warns on cosmetic issues, errors on missing required fields |
| `skillctl verify [<id>]` | Re-compute SHA-256 of installed skills and compare against the registry |
| `skillctl inspect <id>` | Audit a skill: declared deps, bundled scripts, env vars, risk class |

## Wire schemas

The full JSON / TSV schemas for every command's `--format` output are documented in [`PROTOCOL.md` §3](https://github.com/agent-rt/skillctl/blob/main/PROTOCOL.md). Any tool implementing those four protocol commands byte-compatibly is a conformant gateway — `skillctl` is just the reference implementation.

## Environment variables

| Var | Effect |
|---|---|
| `SKILLCTL_TRUST_GATE=1` | Enables the project trust check on read-only agent commands. Default off — `init` / `use` / `enable` / `trust` auto-add. Recommended for agent harnesses that consume third-party repositories. |
| `RUST_LOG` | Standard `tracing-subscriber` log level. `RUST_LOG=skillctl=debug` for verbose tracing. |
