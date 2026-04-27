---
title: Quickstart · memctl
---

# Quickstart

Goal: capture an observation, retrieve it from a different working directory, and see how project-scoped memory commits with code.

Prereq: `memctl` installed (see [Install](/memctl/install/)).

## 1. Capture a global observation

```sh
export MEMORYCTL_AGENT=claude-code   # so source attribution is real

memctl save --type decision --topic skillctl-design \
  --no-confirm \
  "TSV beats JSON for agent list output: 3x token savings on the highest-frequency call"

memctl save --type lesson --topic skillctl-design \
  --no-confirm \
  "AGENTS.md byte-stable block is the core protocol invariant; any churn invalidates prompt cache"
```

`--no-confirm` skips the interactive prompt; without it, an agent or script would have to confirm each write. (Recommended setting for human shells: leave the prompt on for safety.)

## 2. List topics

```sh
memctl list --format tsv
# TOPIC             ENTRIES  LAST_UPDATED      TYPES             SCOPE
# skillctl-design   2        2026-04-27T15:40  decision,lesson   global
```

The 5 columns are exactly what an agent needs to decide what to load.

## 3. Read full content

```sh
memctl read --topic skillctl-design
# # skillctl-design
#
# ## 2026-04-27T15:40 [type=decision source=claude-code @ ai-workspace/skillctl]
# TSV beats JSON for agent list output...
#
# ## 2026-04-27T15:40 [type=lesson source=claude-code @ ai-workspace/skillctl]
# AGENTS.md byte-stable block is the core protocol invariant...
```

Filter by type or time:

```sh
memctl read --topic skillctl-design --type decision
memctl read --topic skillctl-design --since 7d
memctl read --topic skillctl-design --format tsv     # one entry per row, agent-optimized
```

## 4. Search across topics

```sh
memctl search "prompt cache"
# TOPIC             TIMESTAMP         TYPE     SCOPE   MATCH
# skillctl-design   2026-04-27T15:40  lesson   global  …prompt cache. any churn invalidates…
```

Search is byte-wise on entry content; CJK is safe.

## 5. Project-scoped memory (commits with code)

For lore that's only relevant to one repository:

```sh
cd ~/work/elepay
memctl init                            # creates .memctl/topics/ + AGENTS.md block
memctl save --type fact --topic api-conventions \
  --scope project --no-confirm \
  "POST /payments must include Idempotency-Key header"

git add .memctl AGENTS.md
git commit -m "memctl: capture API conventions"
```

Now any team member who clones this repo:

```sh
cd elepay
memctl read --topic api-conventions --scope project
# Stripe → us → merchant clearing path...
```

picks up the lore immediately. **This is the core differentiator vs Claude Code / Cursor private memory** — those don't travel with code.

## 6. Coexistence with skillctl

If you use both tools, your `AGENTS.md` will hold two independent managed blocks:

```
<!-- skillctl:start version=1 ... -->
... skillctl protocol rules ...
<!-- skillctl:end -->

<!-- memctl:start version=1 -->
... memctl protocol rules ...
<!-- memctl:end -->
```

`memctl disable` removes only its own block, never touching skillctl's. Same for the reverse.

## What's next

- [Commands](/memctl/commands/) — full subcommand reference.
- Pair with [`skillctl`](/skillctl/) — capability bundles + persistent memory in one `AGENTS.md`.
