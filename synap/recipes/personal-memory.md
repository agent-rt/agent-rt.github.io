---
title: Personal Memory Recipe · Synap
---

# Recipe: Personal memory

Teach your agent enduring facts about you *and* log moments in time. The
`synap.memories` app is built in — no `init_app` step.

## Profile attributes (enduring state)

> Remember that I prefer Rust, live in Tokyo, and am allergic to shellfish.

The agent calls `remember` three times:

```json
{"tool": "remember", "args": {
  "type": "attr", "attribute": "preferred_language",
  "op": "set", "value": "Rust"
}}
{"tool": "remember", "args": {
  "type": "attr", "attribute": "city",
  "op": "set", "value": "Tokyo"
}}
{"tool": "remember", "args": {
  "type": "attr", "attribute": "allergies",
  "op": "append", "value": "shellfish"
}}
```

Note `op: "append"` for list-valued attributes — appends "shellfish" to
any existing list. `op: "set"` overwrites.

## Events (timestamped observations)

> Log that I shipped Synap 0.1.0 today.

```json
{"tool": "remember", "args": {
  "type": "event",
  "description": "User shipped Synap 0.1.0",
  "when": "2026-04-19"
}}
```

Events are append-only. `description` is embedded so semantic search
finds events by meaning.

## Reading back

> What do you know about me?

Agent calls `query` (not `search` — this is a structured lookup):

```json
{"tool": "query", "args": {
  "app_id": "synap.memories",
  "filters": [{"field": "subject", "op": "eq", "value": "user"}]
}}
```

Returns TSV with all profile attributes.

## Fuzzy recall with time decay

> What have I been working on lately?

Agent calls `search` with `rank: "time_decay"`:

```json
{"tool": "search", "args": {
  "app_id": "synap.memories",
  "query": "working on",
  "rank": "time_decay"
}}
```

Blends semantic similarity with a freshness half-life — recent events
weigh more. See [Concepts · Hybrid search](/synap/concepts/#hybrid-search-and-ranking).

## Forgetting

> Forget that I live in Tokyo.

```json
{"tool": "forget", "args": {
  "type": "attr", "attribute": "city"
}}
```

Precise by key. Don't "query then write null" — `forget` is atomic.

## Pre-turn injection

You don't need to explicitly ask "what do you know about me?" — the
daemon injects a `[memory: profile]` block into every turn the agent
sees. Events are surfaced via `[memory: events]`, ranked by time-decay
when the user query is non-empty, recency otherwise.

Config lives under `[agent.memory]` in `agent.toml`:
`event_digest_max`, `decay_half_life_days`, `decay_weight`.

## Tracking someone else

Memory defaults to subject `"user"`. For multi-subject tracking:

```json
{"tool": "remember", "args": {
  "subject": "alice",
  "type": "attr", "attribute": "prefers",
  "op": "set", "value": "tea"
}}
```

Query with `filters: [{field: "subject", op: "eq", value: "alice"}]`.
