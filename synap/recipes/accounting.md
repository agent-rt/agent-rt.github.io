---
title: Accounting Recipe · Synap
---

# Recipe: Accounting

Track expenses and income with categories. Ask natural-language questions
like "how much did I spend on food last month?" and let the agent
translate to structured queries.

## 1. Design

> Make a synap app "ledger" with entity_types "expense" and "income".
> Both have: amount (number, required), currency (text, default "JPY"),
> category (text), description (text, vectorized), occurred_at (date,
> required), account (text).

```json
{"tool": "init_app", "args": {
  "name": "ledger",
  "description": "Personal ledger — expenses and income",
  "types": {
    "expense": {
      "fields": {
        "amount":      {"type": "number", "required": true},
        "currency":    {"type": "text",   "default":  "JPY"},
        "category":    {"type": "text"},
        "description": {"type": "text",   "vectorized": true},
        "occurred_at": {"type": "date",   "required": true},
        "account":     {"type": "text"}
      }
    },
    "income": {
      "fields": {
        "amount":      {"type": "number", "required": true},
        "currency":    {"type": "text",   "default":  "JPY"},
        "source":      {"type": "text"},
        "description": {"type": "text",   "vectorized": true},
        "occurred_at": {"type": "date",   "required": true},
        "account":     {"type": "text"}
      }
    }
  }
}}
```

One app, two entity types. Each type has its own schema, vec table, and
FTS table — but they share an `app_id`, so cross-type queries work.

## 2. Record transactions

> Log: spent 1200 yen on lunch at Ichiran today, paid with SMBC card.

```json
{"tool": "write", "args": {
  "app_id": "ledger",
  "ops": [{
    "op": "create",
    "entity_type": "expense",
    "entity": {
      "amount": 1200, "currency": "JPY",
      "category": "food", "description": "Lunch at Ichiran",
      "occurred_at": "2026-04-19", "account": "SMBC card"
    }
  }]
}}
```

> Log: received 500,000 yen salary from Acme Corp on 2026-04-25.

```json
{"tool": "write", "args": {
  "app_id": "ledger",
  "ops": [{
    "op": "create",
    "entity_type": "income",
    "entity": {
      "amount": 500000, "currency": "JPY",
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
  "app_id": "ledger",
  "entity_type": "expense",
  "filters": [
    {"field": "category",    "op": "eq", "value": "food"},
    {"field": "occurred_at", "op": "gte", "value": "2026-04-01"},
    {"field": "occurred_at", "op": "lt",  "value": "2026-05-01"}
  ],
  "select": ["amount", "description", "occurred_at"]
}}
```

Agent sums `amount` client-side from TSV. Synap doesn't do SQL aggregates
in 0.1 — projection + iteration is the pattern.

## 4. Cross-type net position

> What's my net for April?

Two queries (expenses + income), agent subtracts totals. Or a single
query over both types by omitting `entity_type`:

```json
{"tool": "query", "args": {
  "app_id": "ledger",
  "filters": [
    {"field": "occurred_at", "op": "gte", "value": "2026-04-01"},
    {"field": "occurred_at", "op": "lt",  "value": "2026-05-01"}
  ]
}}
```

Result includes an `entity_type` column — partition client-side.

## 5. Fuzzy description search

> Did I ever pay for that music subscription thing?

```json
{"tool": "search", "args": {
  "app_id":      "ledger",
  "entity_type": "expense",
  "query":       "music streaming subscription",
  "rank":        "relevance"
}}
```

Finds a "Spotify Family" line item even if you wrote "Spotify" in the
description without the word "music" or "subscription".

## 6. Monthly summary as a prompt

> Give me a one-paragraph summary of my April finances.

The agent orchestrates:

1. `query` all April expenses + incomes
2. Group by category client-side
3. Compose prose ("spent ¥48,200 on food (13 meals out), ¥9,400 on
   transport, ¥6,500 on streaming…")

## Design tips

- Use `default` on currency so the agent doesn't need to specify every
  time.
- Vectorize `description` but not `category` — categories are discrete
  labels; fuzzy matching them produces noise.
- For recurring transactions, consider an `is_recurring: bool` field and
  a separate "templates" entity_type you copy from.
