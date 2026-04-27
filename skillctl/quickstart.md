---
title: Quickstart · skillctl
---

# Quickstart

Goal: in under two minutes, install a skill, create a profile, apply it to a project, and watch an agent consume the result through the protocol surface.

Prereq: `skillctl` installed (see [Install](/skillctl/install/)).

## 1. Add a skill to the global store

A skill is a directory with a `SKILL.md` file. For this walkthrough we'll author one inline:

```sh
mkdir -p /tmp/rust-review/
cat > /tmp/rust-review/SKILL.md <<'EOF'
---
name: review
version: "1.0.0"
description: Idiomatic Rust review checklist.
metadata:
  namespace: rust
  summary: Rust review — ownership, error handling, unsafe audit
  tags: [rust, review]
  languages: [rust]
  triggers: [rust, cargo, clippy, unsafe]
  requires:
    binaries:
      - { name: cargo, version: ">=1.75" }
---

# Rust Review

- No `unwrap()` / `expect()` / `panic!()` in production code.
- Prefer `&str` / `&[u8]` over `String` / `Vec<u8>` when ownership isn't needed.
- Newtype basic types (`UserId(u64)`) to prevent misuse.
- `unsafe` requires `// SAFETY:` comment.
EOF

skillctl add /tmp/rust-review --as rust/review
```

This copies the skill into `~/.skillctl/skills/rust/review/1.0.0/`, records its checksum, and indexes it in `~/.skillctl/registry.lock`.

## 2. Create a profile

A profile is a TOML file declaring which skills are `core` (always-on) vs `extra` (loaded on demand):

```sh
cat > /tmp/rust-cli.toml <<'EOF'
[profile]
name = "rust-cli"
description = "Rust CLI starter"

[skills.core]
"rust/review" = "1.0.0"

[skills.extra]
EOF

skillctl profile add /tmp/rust-cli.toml
```

## 3. Apply the profile to a project

```sh
mkdir -p ~/work/my-cli && cd ~/work/my-cli
skillctl init rust-cli
```

Three files appear:

- `skillctl.toml` — manifest with `started_from = ["rust-cli"]` and `[skills.core]` populated
- `skillctl.lock` — pinned versions and checksums
- `AGENTS.md` — managed block telling agents which four commands to use

Open `AGENTS.md` and you'll see:

```
<!-- skillctl:start version=1 started_from=rust-cli -->
## skillctl

This project uses skillctl to manage Agent skills.

Rules:
- Use only project-enabled skills. Do not scan the filesystem for skills.
- Before non-trivial work, list available skills (TSV: tier / id / summary / triggers):
  `skillctl list --project --format tsv`
...
```

These bytes are stable — adding more skills with `skillctl use` will not change them.

## 4. Consume as an agent

Run the four commands from the agent's perspective:

```sh
skillctl list --project --format tsv
# TIER  ID            SUMMARY                                          TRIGGERS
# core  rust/review   Rust review — ownership, error handling, ...    rust,cargo,clippy,unsafe

skillctl describe rust/review --format json | jq '{id, tier, deps: [.requires.binaries[].name]}'
# {"id":"rust/review","tier":"core","deps":["cargo"]}

skillctl show rust/review | head -5
# ---
# name: review
# version: "1.0.0"
# ...
```

That's the full agent-side protocol. An agent walking into this project starts with `list`, decides which skills are relevant, and `show`s their full instructions only when needed.

## 5. Verify reproducibility

```sh
sha256sum AGENTS.md
skillctl use --add rust/review --core   # reorder, no-op since it's already core
sha256sum AGENTS.md                     # unchanged
```

`AGENTS.md` is byte-stable across `add` / `use` / `update` / `restore`. Only `enable` / `disable` / `init` (when called explicitly) modify it. This keeps prompt caches warm.

## What's next

- Browse [Commands](/skillctl/commands/) for the full subcommand reference.
- Read the [protocol spec](https://github.com/agent-rt/skillctl/blob/main/PROTOCOL.md) for the wire-level contract any conformant gateway must implement.
- Pair with [`memoryctl`](/memoryctl/) — both tools coexist in the same `AGENTS.md` with independent managed blocks.
