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
  "app_id": "kb",
  "types": {
    "note": [
      {"name": "title",       "value_type": "string", "indexed": true},
      {"name": "source",      "value_type": "string"},
      {"name": "tags",        "value_type": "array"},
      {"name": "body",        "value_type": "string", "vectorized": true},
      {"name": "captured_at", "value_type": "string", "indexed": true}
    ]
  }
}}
```

`vectorized: true` on `body` is the magic. Synap auto-embeds that field
on every write and chunks >1024-token bodies with 128-token overlap.
`captured_at` is ISO 8601 in a `string` — there's no dedicated date
type, but lexicographic compare on ISO strings gives correct range
filtering.

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
  "app_id": "kb",
  "ops": [{
    "op": "create",
    "entity_type": "note",
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

**Batching**: pass multiple ops in one array for atomic writes (all-or-
nothing):

```json
{"tool": "write", "args": {
  "app_id": "kb",
  "ops": [
    {"op": "create", "entity_type": "note", "data": {"title": "Entry A", "body": "..."}},
    {"op": "create", "entity_type": "note", "data": {"title": "Entry B", "body": "..."}},
    {"op": "create", "entity_type": "note", "data": {"title": "Entry C", "body": "..."}}
  ]
}}
```

## 3. Semantic search

> Find my notes on concurrency in systems languages.

```json
{"tool": "search", "args": {
  "app_id":     "kb",
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
  "app_id":     "kb",
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
  "app_id":   "kb",
  "filters":  [{"field": "captured_at", "op": "gte", "value": "2026-04-14"}],
  "order_by": [{"field": "captured_at", "direction": "desc"}],
  "fields":   ["title", "tags", "captured_at"]
}}
```

`fields` projects only selected attributes — essential when bodies are
long and you just want a list.

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
- For multiple shapes in one app (articles vs. code snippets), add
  more entries to `types` (e.g., `{types: {article: [...], snippet:
  [...]}}`) — each type gets its own schema and vec table.
