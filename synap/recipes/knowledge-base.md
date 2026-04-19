---
title: Knowledge Base Recipe · Synap
---

# Recipe: Knowledge base

A searchable archive of notes, articles, and references. Long text is
auto-chunked and embedded, so semantic search works out of the box.

## 1. Design the schema

> Make a synap app called "kb" for my personal knowledge base. Fields:
> title (required text), source (text, optional), tags (text, multi),
> body (text, vectorized), captured_at (date).

```json
{"tool": "init_app", "args": {
  "name": "kb",
  "description": "Personal knowledge base: notes, articles, references",
  "fields": {
    "title":       {"type": "text", "required": true},
    "source":      {"type": "text"},
    "tags":        {"type": "text", "multi": true},
    "body":        {"type": "text", "vectorized": true},
    "captured_at": {"type": "date"}
  }
}}
```

`vectorized: true` on `body` is the magic. It makes Synap auto-embed that
field on every write, chunking >1024-token bodies with 128-token overlap.

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
    "entity": {
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
    {"op": "create", "entity": { "title": "Entry A", "body": "..." }},
    {"op": "create", "entity": { "title": "Entry B", "body": "..." }},
    {"op": "create", "entity": { "title": "Entry C", "body": "..." }}
  ]
}}
```

## 3. Semantic search

> Find my notes on concurrency in systems languages.

```json
{"tool": "search", "args": {
  "app_id": "kb",
  "query": "concurrency in systems languages",
  "rank":  "relevance",
  "limit": 10
}}
```

Returns TSV: `entity_id  title  body_snippet  score`. Snippets are extracted
from the best-matching chunk with surrounding context.

## 4. Filter + search combo

> Find concurrency notes, but only ones tagged `rust`.

```json
{"tool": "search", "args": {
  "app_id": "kb",
  "query": "concurrency",
  "filters": [{"field": "tags", "op": "contains", "value": "rust"}]
}}
```

Filters run before ranking — dramatically narrows the candidate set.

## 5. Browse by date

> Show me everything I saved this week.

```json
{"tool": "query", "args": {
  "app_id": "kb",
  "filters": [{"field": "captured_at", "op": "gte", "value": "2026-04-14"}],
  "sort":    [{"field": "captured_at", "dir": "desc"}],
  "select":  ["title", "tags", "captured_at"]
}}
```

`select` projects only specific fields — useful when bodies are long and
you just want a list.

## 6. Evolve the schema later

Decide you also want a `rating` field? Just start writing it:

```json
{"op": "update", "entity_id": "...", "entity": {"rating": 5}}
```

EAV means old rows simply lack the attribute. No migration, no downtime.

## Design tips

- Mark long-text fields `vectorized: true` so `search` finds them
  semantically. Short text (titles, tags) doesn't need vectorization —
  FTS5 handles it.
- Use `multi: true` for list-valued fields (`tags`, `authors`).
- If a `kb` app accumulates different shapes (articles vs. code
  snippets), consider `entity_type` to split schemas within one app —
  see the [Reference](/synap/reference/).
