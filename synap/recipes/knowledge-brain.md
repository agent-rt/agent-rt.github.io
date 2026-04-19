---
title: Knowledge Brain Recipe · Synap
---

# Recipe: Knowledge brain

A typed-graph knowledge store — pages (concepts / people / companies /
papers) connected by labelled edges (`works_at`, `attended`,
`cites`, `invested_in`). Each page carries a **compiled truth** (the
current understanding, edited as evidence changes) plus an
append-only stream of events. Query by meaning, by structure, or by
graph traversal.

Inspired by [GBrain](https://github.com/garrytan/gbrain)'s compiled-
truth + timeline pattern; adapted to Synap 0.2's `expand_refs`, `date`
fields, and `types` multi-entity apps.

## 1. Design the schemas

Three apps: `brain.pages` for nodes, `brain.links` for edges,
`brain.events` for the append-only timeline. Separating them lets the
vectorizer run only on the long-text fields (`compiled_truth`, event
`summary`) and keeps edge writes cheap.

### `brain.pages` — multi-type nodes

> Create brain.pages with types concept / person / company / paper.
> Shared fields: title, slug, compiled_truth, tags, updated_at.
> People also have org (ref). Companies have founded_at.

```json
{"tool": "init_app", "args": {
  "app": "brain.pages",
  "types": {
    "concept": [
      {"name": "slug",           "value_type": "string", "indexed": true, "required": true},
      {"name": "title",          "value_type": "string", "indexed": true, "vectorized": true, "required": true},
      {"name": "compiled_truth", "value_type": "string", "vectorized": true},
      {"name": "tags",           "value_type": "array"},
      {"name": "updated_at",     "value_type": "date",   "indexed": true, "default": "now"}
    ],
    "person": [
      {"name": "slug",           "value_type": "string", "indexed": true, "required": true},
      {"name": "title",          "value_type": "string", "indexed": true, "vectorized": true, "required": true},
      {"name": "compiled_truth", "value_type": "string", "vectorized": true},
      {"name": "org",            "value_type": "ref"},
      {"name": "tags",           "value_type": "array"},
      {"name": "updated_at",     "value_type": "date",   "indexed": true, "default": "now"}
    ],
    "company": [
      {"name": "slug",           "value_type": "string", "indexed": true, "required": true},
      {"name": "title",          "value_type": "string", "indexed": true, "vectorized": true, "required": true},
      {"name": "compiled_truth", "value_type": "string", "vectorized": true},
      {"name": "founded_at",     "value_type": "date",   "indexed": true},
      {"name": "tags",           "value_type": "array"},
      {"name": "updated_at",     "value_type": "date",   "indexed": true, "default": "now"}
    ],
    "paper": [
      {"name": "slug",           "value_type": "string", "indexed": true, "required": true},
      {"name": "title",          "value_type": "string", "indexed": true, "vectorized": true, "required": true},
      {"name": "compiled_truth", "value_type": "string", "vectorized": true},
      {"name": "authors",        "value_type": "ref"},
      {"name": "published_at",   "value_type": "date",   "indexed": true},
      {"name": "tags",           "value_type": "array"},
      {"name": "updated_at",     "value_type": "date",   "indexed": true, "default": "now"}
    ]
  }
}}
```

`updated_at` uses `default: "now"` — the agent doesn't have to set it
on write, and `value_type: date` rejects garbage like `"yesterday"`
at the boundary. `authors` and `org` are `ref` fields — Synap 0.2's
`expand_refs` can follow them in a single query (see §4).

### `brain.links` — typed edges

```json
{"tool": "init_app", "args": {
  "app": "brain.links",
  "fields": [
    {"name": "from",     "value_type": "ref", "indexed": true, "required": true},
    {"name": "to",       "value_type": "ref", "indexed": true, "required": true},
    {"name": "rel_type", "value_type": "enum",
     "enum_values": ["works_at", "attended", "cites", "invested_in", "founded", "advises", "mentions"],
     "indexed": true, "required": true},
    {"name": "since",    "value_type": "date",  "indexed": true},
    {"name": "note",     "value_type": "string"}
  ]
}}
```

Keeping edges in their own app (rather than inline arrays on each
page) means a single `query brain.links filter=[rel_type eq works_at]
expand_refs=1` gives you every employment relationship in the graph
with both endpoints hydrated — no N+1.

### `brain.events` — append-only timeline

```json
{"tool": "init_app", "args": {
  "app": "brain.events",
  "fields": [
    {"name": "page",    "value_type": "ref",    "indexed": true, "required": true},
    {"name": "when",    "value_type": "date",   "indexed": true, "required": true, "default": "now"},
    {"name": "source",  "value_type": "string"},
    {"name": "summary", "value_type": "string", "vectorized": true, "required": true}
  ]
}}
```

`summary` is vectorized so `search rank=time_decay` naturally
surfaces *recent and relevant* events — the default mode for "what's
new about X?" questions.

## 2. Ingest

> Read ~/notes/transformers.md, decide it's a concept page, and add
> it. Mention any papers or people it references.

```json
{"tool": "read_file", "args": {"path": "~/notes/transformers.md"}}
// agent parses the markdown, extracts title/tags/body, then:
{"tool": "write", "args": {
  "app": "brain.pages",
  "ops": [
    {"op": "create", "type": "concept",
     "data": {
       "slug": "transformers",
       "title": "Transformers",
       "compiled_truth": "Sequence-to-sequence architecture using self-attention…",
       "tags": ["deep-learning", "nlp", "architecture"]
     },
     "returning": true}
  ]
}}
// → {"results": [{"entity_ids": ["pg_xyz"],
//                 "entities": [{"entity_id": "pg_xyz", "slug": "transformers", ...}]}]}
```

`returning: true` echoes the persisted row including the server-filled
`updated_at`, saving a follow-up `query`. Don't set it reflexively —
only when you actually need fields back (`default`-filled or partial
update).

### Batching related nodes + edges

When one source yields multiple entities, batch them so they commit
atomically:

```json
{"tool": "write", "args": {
  "app": "brain.pages",
  "ops": [
    {"op": "create", "type": "paper",  "data": {"slug": "attention-is-all-you-need",
                                                "title": "Attention Is All You Need",
                                                "published_at": "2017-06-12"}},
    {"op": "create", "type": "person", "data": {"slug": "vaswani", "title": "Ashish Vaswani"}}
  ]
}}
```

Then wire the edge in a second call (after capturing both entity_ids
from the first response). Cross-app writes can't batch atomically, but
each app's ops are all-or-nothing internally — good enough for idea
ingestion.

## 3. Query by structure

> What concepts are tagged with "nlp", most recently updated first?

```json
{"tool": "query", "args": {
  "app":     "brain.pages",
  "type":    "concept",
  "filters": [{"field": "tags", "op": "array_contains", "value": "nlp"}],
  "order_by":[{"field": "updated_at", "direction": "desc"}],
  "select":  ["slug", "title", "updated_at"]
}}
```

### Cross-type catalog (Karpathy `index.md` pattern)

One `query` per type surfaces the whole brain at a glance:

```json
{"tool": "query", "args": {
  "app":      "brain.pages",
  "select":   ["slug", "title", "type", "updated_at"],
  "order_by": [{"field": "updated_at", "direction": "desc"}],
  "limit":    100
}}
```

For a per-type tally (how much does the brain know?) use `aggregate`:

```json
{"tool": "query", "args": {
  "app":       "brain.pages",
  "aggregate": [{"fn": "count", "as": "pages"}],
  "group_by":  "type"
}}
// TSV: key       pages
//      concept   128
//      person    42
//      paper     71
```

## 4. Graph traversal

### One hop: who works at Acme?

```json
{"tool": "query", "args": {
  "app":     "brain.links",
  "filters": [
    {"field": "rel_type", "op": "eq",     "value": "works_at"},
    {"field": "to",       "op": "ref_to", "value": "pg_acme"}
  ],
  "expand_refs": 1,
  "select": ["rel_type", "from.title", "from.slug"]
}}
```

`expand_refs: 1` hydrates both `from` and `to` in one SQL round-trip;
`select` with dot-paths keeps only the fields you need and drops the
`_ref` / `_entity_id` sentinels. `ref_to` uses Synap's entity-id index
rather than a JSON scan.

### Two hops: who did Alice attend meetings with?

Two simple queries are cleaner than a recursive CTE when the agent is
in the loop anyway:

```json
// Step 1: meetings Alice attended.
{"tool": "query", "args": {
  "app": "brain.links",
  "filters": [
    {"field": "rel_type", "op": "eq",       "value": "attended"},
    {"field": "from",     "op": "ref_from", "value": "pg_alice"}
  ],
  "select": ["to"]
}}
// → meeting_ids = [m1, m2, ...]

// Step 2: everyone else who attended those same meetings.
{"tool": "query", "args": {
  "app": "brain.links",
  "filters": [
    {"field": "rel_type", "op": "eq", "value": "attended"},
    {"field": "to",       "op": "in", "value": ["m1", "m2"]},
    {"field": "from",     "op": "ne", "value": "pg_alice"}
  ],
  "expand_refs": 1,
  "select": ["from.title"]
}}
```

### Orphan detection

Pages that no edge points at — likely candidates for pruning or
stronger cross-references:

```json
{"tool": "query", "args": {
  "app":     "brain.pages",
  "filters": [{"field": "", "op": "orphan", "value": true}],
  "select":  ["slug", "title", "type", "updated_at"]
}}
```

## 5. Semantic & temporal search

### "What does the brain think about agent frameworks?"

```json
{"tool": "search", "args": {
  "app":        "brain.pages",
  "query_text": "agent framework design patterns",
  "rank":       "relevance",
  "limit":      10
}}
```

### "What's new about Alice?"

Events were modelled separately precisely so freshness works
naturally — `time_decay` blends similarity with recency:

```json
{"tool": "search", "args": {
  "app":        "brain.events",
  "query_text": "Alice",
  "filters":    [{"field": "page", "op": "ref_to", "value": "pg_alice"}],
  "rank":       "time_decay",
  "limit":      10
}}
```

## 6. Evolve the compiled truth

As new evidence arrives, *update* the page's `compiled_truth` (the
current understanding) and *append* an event explaining what changed.
The page reflects the latest belief; events preserve the why.

```json
{"tool": "write", "args": {
  "app": "brain.pages",
  "ops": [{
    "op": "update",
    "entity_id": "pg_transformers",
    "data": {"compiled_truth": "…updated narrative with new findings…"}
  }]
}}
{"tool": "write", "args": {
  "app": "brain.events",
  "ops": [{
    "op": "create",
    "data": {
      "page":    "brain.pages/pg_transformers",
      "source":  "paper:arxiv/2406.12345",
      "summary": "Revised understanding: FlashAttention-2 supplants the naïve attention kernel for production inference."
    }
  }]
}}
```

The update is server-stamped (`default: "now"` on `updated_at`);
event `when` is auto-set the same way.

## Design tips

- **Edges > arrays**: store relationships in `brain.links`, not as
  inline `related: [ref1, ref2]` arrays. Typed edges (`rel_type`)
  let you ask *why* they're connected; inline arrays don't.
- **Vectorize selectively**: `compiled_truth` and event `summary` are
  worth embedding; `slug`, `tags`, `rel_type` are not — FTS5 keyword
  matching handles short strings better than dense vectors.
- **`expand_refs` + dot-path `select` is the win**: on a 100-edge
  query, one round-trip returns target titles; without it you'd fire
  100 follow-up queries and pay token cost for fields you didn't want.
- **`synap.memories` vs `brain.events`**: memories are *about the
  user* (preferences, biographical facts). Brain events are *about
  pages* (observations about external entities). Don't conflate them
  — the subject field is different and the retention policy is
  different.
