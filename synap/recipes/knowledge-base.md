---
title: Knowledge Base Recipe · Synap
---

# Recipe: Knowledge base

A searchable archive of notes, articles, and references. Long text is
auto-chunked and embedded, so semantic search works out of the box.

## 1. Design the schema

> Make a synap app "kb" for my personal knowledge base. Fields: title,
> source, tags (array), body (vectorized), captured_at.

```json
{"tool": "init_app", "args": {
  "app": "kb",
  "fields": [
    {"name": "title",       "value_type": "string", "indexed": true, "required": true},
    {"name": "source",      "value_type": "string"},
    {"name": "tags",        "value_type": "array"},
    {"name": "body",        "value_type": "string", "vectorized": true},
    {"name": "captured_at", "value_type": "date",   "indexed": true, "default": "now"}
  ]
}}
```

`fields` is the single-entity-type shorthand — equivalent to `types:
{kb: [fields]}`. Use it whenever an app holds one shape of entity. For
multi-type apps (see the [accounting recipe](/synap/recipes/accounting/)),
drop back to `types`. `required: true` makes the server reject create
ops missing `title`.

`vectorized: true` on `body` is the magic. Synap auto-embeds that field
on every write and chunks >1024-token bodies with 128-token overlap.
`captured_at` is a real `date` field — ISO 8601 validated on write —
with `default: "now"` so the agent never has to supply it by hand.

## 2. Add entries

> Save this note:
> **Title**: Arc<Mutex<T>> vs. channels
> **Tags**: rust, concurrency
> **Source**: personal
> **Body**: `Arc<Mutex<T>>` is for shared mutable state between threads.
> Prefer channels when the pattern is producer/consumer — they move
> ownership rather than share it...

```json
{"tool": "write", "args": {
  "app": "kb",
  "ops": [{
    "op": "create",
    "data": {
      "title": "Arc<Mutex<T>> vs. channels",
      "tags":  ["rust", "concurrency"],
      "source": "personal",
      "body":  "Arc<Mutex<T>> is for shared mutable state between threads. Prefer channels when the pattern is producer/consumer — they move ownership rather than share it...",
      "captured_at": "2026-04-19"
    }
  }]
}}
```

With the `fields` shorthand the implicit type name is empty, so `type`
can be omitted on writes.

**Batching**: pass multiple ops in one array for atomic writes (all-or-
nothing):

```json
{"tool": "write", "args": {
  "app": "kb",
  "ops": [
    {"op": "create", "data": {"title": "Entry A", "body": "..."}},
    {"op": "create", "data": {"title": "Entry B", "body": "..."}},
    {"op": "create", "data": {"title": "Entry C", "body": "..."}}
  ]
}}
```

## 3. Semantic search

> Find my notes on concurrency in systems languages.

```json
{"tool": "search", "args": {
  "app":        "kb",
  "query_text": "concurrency in systems languages",
  "mode":       "semantic",
  "rank":       "relevance",
  "limit":      10
}}
```

Returns TSV: `entity_id  title  body_snippet  score`. Snippets come from
the best-matching chunk with surrounding context.

## 4. Filter + search combo

> Find concurrency notes, but only ones tagged `rust`.

```json
{"tool": "search", "args": {
  "app":        "kb",
  "query_text": "concurrency",
  "filters":    [{"field": "tags", "op": "array_contains", "value": "rust"}]
}}
```

Filters run before ranking, dramatically narrowing the candidate set.
`array_contains` tests membership in an `array`-typed field.

## 5. Browse by date

> Show me everything I saved this week.

```json
{"tool": "query", "args": {
  "app":      "kb",
  "filters":  [{"field": "captured_at", "op": "gte", "value": "2026-04-14"}],
  "order_by": [{"field": "captured_at", "direction": "desc"}],
  "select":   ["title", "tags", "captured_at"]
}}
```

`select` projects only the listed attributes — essential when bodies
are long and you just want a list.

## 6. Evolve the schema later

Decide you also want a `rating` field? Just start writing it:

```json
{"op": "update", "entity_id": "...", "data": {"rating": 5}}
```

EAV means old rows simply lack the attribute. No migration, no downtime.
The schema registry updates transparently on first write.

## Design tips

- `vectorized: true` on long strings (bodies, descriptions). Don't
  vectorize short labels (titles, tags) — FTS5 keyword match handles
  them.
- `indexed: true` on fields you filter or sort by frequently — B-tree
  lookups instead of full scans.
- For multiple shapes in one app (articles vs. code snippets), use
  `types` (e.g., `{types: {article: [...], snippet: [...]}}`) — each
  type gets its own schema and vec table.
