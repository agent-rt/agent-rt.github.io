---
title: Personal Memory Recipe · Synap
---

# Recipe: Personal memory

Teach your agent enduring facts about you *and* log moments in time.
`synap.memories` is built in — no `init_app` step.

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

`op: "set"` overwrites. `op: "append"` adds to a list (creates it if
missing). `op: "clear"` with `value` removes one element; without
`value` unsets the attribute.

## Events (timestamped observations)

> Log that I shipped Synap 0.1.0 today.

```json
{"tool": "remember", "args": {
  "type": "event",
  "description": "User shipped Synap 0.1.0"
}}
```

Omit `when` for "now". To backdate, pass unix seconds:

```json
{"tool": "remember", "args": {
  "type": "event",
  "description": "User shipped Synap 0.1.0",
  "when": 1744934400
}}
```

Events are append-only. `description` is embedded so semantic search
finds events by meaning.

## Reading back

> What do you know about me?

Agent calls `query` (structured, not fuzzy):

```json
{"tool": "query", "args": {
  "app":  "synap.memories",
  "type": "profile",
  "filters": [{"field": "subject", "op": "eq", "value": "user"}]
}}
```

Returns TSV with all profile attributes.

## Fuzzy recall with time decay

> What have I been working on lately?

```json
{"tool": "search", "args": {
  "app":        "synap.memories",
  "type":       "event",
  "query_text": "working on",
  "rank":       "time_decay"
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

To drop a single event, first `query` / `search` to get its id, then:

```json
{"tool": "forget", "args": {
  "type": "event", "id": "<entity_id>"
}}
```

## Pre-turn injection

You don't need to explicitly ask "what do you know about me?" — the
daemon injects `[memory: profile]` and `[memory: events]` blocks into
every turn. Config lives under `[agent.memory]` in `agent.toml`:
`event_digest_max`, `decay_half_life_days`, `decay_weight`.

## Tracking someone else

Memory defaults to the current user. For multi-subject tracking:

```json
{"tool": "remember", "args": {
  "subject": "alice",
  "type": "attr", "attribute": "prefers",
  "op": "set", "value": "tea"
}}
```

Query with `filters: [{field: "subject", op: "eq", value: "alice"}]`.
