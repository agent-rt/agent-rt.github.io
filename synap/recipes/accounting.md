---
title: Accounting Recipe · Synap
---

# Recipe: Accounting

Track expenses and income with categories. Ask natural-language questions
like "how much did I spend on food last month?" and let the agent
translate to structured queries.

## 1. Design

> Make a synap app "ledger" with two entity types — "expense" and
> "income". Both have amount, currency, description (vectorized),
> occurred_at, account.

```json
{"tool": "init_app", "args": {
  "app": "ledger",
  "types": {
    "expense": [
      {"name": "amount",      "value_type": "number", "indexed": true, "required": true},
      {"name": "currency",    "value_type": "string", "default":   "JPY"},
      {"name": "category",    "value_type": "string", "indexed":   true},
      {"name": "description", "value_type": "string", "vectorized": true},
      {"name": "occurred_at", "value_type": "string", "indexed":   true, "required": true},
      {"name": "account",     "value_type": "string"}
    ],
    "income": [
      {"name": "amount",      "value_type": "number", "indexed": true, "required": true},
      {"name": "currency",    "value_type": "string", "default":   "JPY"},
      {"name": "source",      "value_type": "string", "indexed":   true},
      {"name": "description", "value_type": "string", "vectorized": true},
      {"name": "occurred_at", "value_type": "string", "indexed":   true, "required": true},
      {"name": "account",     "value_type": "string"}
    ]
  }
}}
```

Multi-type app, so we use `types` (not the `fields` shorthand). `amount`
and `occurred_at` are required — half-populated ledger rows are a bug
magnet. `currency` defaults to `"JPY"` server-side, so each transaction
skips it unless the agent means a different currency.

## 2. Record transactions

> Log: spent 1200 yen on lunch at Ichiran today, paid with SMBC card.

```json
{"tool": "write", "args": {
  "app": "ledger",
  "ops": [{
    "op": "create",
    "type": "expense",
    "data": {
      "amount": 1200,
      "category": "food", "description": "Lunch at Ichiran",
      "occurred_at": "2026-04-19", "account": "SMBC card"
    },
    "returning": true
  }]
}}
```

`returning: true` echoes the persisted row so the agent can confirm
`currency` defaulted to `"JPY"` and the `entity_id` without a second
call. Response:

```json
[{
  "op": "create",
  "entity_ids": ["eid_abc"],
  "entities": [{
    "entity_id": "eid_abc",
    "amount": 1200,
    "currency": "JPY",
    "category": "food",
    "description": "Lunch at Ichiran",
    "occurred_at": "2026-04-19",
    "account": "SMBC card"
  }]
}]
```

> Log: received 500,000 yen salary from Acme Corp on 2026-04-25.

```json
{"tool": "write", "args": {
  "app": "ledger",
  "ops": [{
    "op": "create",
    "type": "income",
    "data": {
      "amount": 500000,
      "source": "Acme Corp", "description": "Salary April 2026",
      "occurred_at": "2026-04-25", "account": "SMBC main"
    }
  }]
}}
```

## 3. Category rollups

> How much did I spend on food in April?

```json
{"tool": "query", "args": {
  "app":  "ledger",
  "type": "expense",
  "filters": [
    {"field": "category",    "op": "eq",      "value": "food"},
    {"field": "occurred_at", "op": "between", "value": ["2026-04-01", "2026-04-30"]}
  ],
  "select": ["amount", "description", "occurred_at"]
}}
```

The agent sums `amount` client-side from TSV. Synap doesn't do SQL
aggregates in 0.2 — projection + iteration is the pattern.

Or, if you just want the count and a grouped breakdown:

```json
{"tool": "query", "args": {
  "app":  "ledger",
  "type": "expense",
  "filters": [{"field": "occurred_at", "op": "between",
               "value": ["2026-04-01", "2026-04-30"]}],
  "count_only": true,
  "group_by":   "category"
}}
```

## 4. Cross-type net position

> What's my net for April?

Two queries (expenses + income), agent subtracts totals. Or a single
cross-type query by omitting `type`:

```json
{"tool": "query", "args": {
  "app": "ledger",
  "filters": [{"field": "occurred_at", "op": "between",
               "value": ["2026-04-01", "2026-04-30"]}]
}}
```

Result includes a `type` column — partition client-side.

## 5. Fuzzy description search

> Did I ever pay for that music subscription thing?

```json
{"tool": "search", "args": {
  "app":        "ledger",
  "type":       "expense",
  "query_text": "music streaming subscription",
  "mode":       "semantic"
}}
```

Finds a "Spotify Family" line even if the description says only
"Spotify" — embeddings capture meaning, not just tokens.

## 6. Monthly summary as a prompt

> Give me a one-paragraph summary of my April finances.

The agent orchestrates:

1. `query` all April expenses + incomes
2. Group by category client-side (or use `count_only + group_by` for
   counts, then one query per category for totals)
3. Compose prose ("spent ¥48,200 on food across 13 meals out, ¥9,400 on
   transport, ¥6,500 on streaming…")

## Design tips

- Vectorize `description` but not `category` — categories are discrete
  labels; fuzzy matching them produces noise. Filter exactly with
  `op: "eq"`.
- `indexed: true` on `amount` / `category` / `occurred_at` — the fields
  you sort and filter on all the time.
- For recurring transactions, a `templates` type alongside `expense` /
  `income` lets the agent copy from saved shapes.
