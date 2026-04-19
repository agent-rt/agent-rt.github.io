---
title: Knowledge Brain Recipe · Synap
---

# Recipe: Knowledge brain

A typed-graph knowledge store — pages (concepts / people / companies /
papers) connected by labelled edges (`works_at`, `attended`, `cites`,
`invested_in`) plus an append-only stream of events. Each page carries
a **compiled truth** (the current understanding, edited as evidence
changes). Query by meaning, by structure, or by graph traversal.

Inspired by [GBrain](https://github.com/garrytan/gbrain)'s compiled-
truth + timeline pattern; adapted to Synap 0.2's `expand_refs`, `date`
fields, and multi-type apps.

## 1. Design the schema — one app, many types

A brain is one domain, even when it holds multiple entity shapes.
Model it as a **single `brain` app with several types**. Synap 0.2's
multi-type `init_app` gives each type its own vectorizer-aware schema
while keeping cross-type queries, atomic cross-type writes, and a
single namespace the agent has to remember.

The types: `concept`, `person`, `company`, `paper` are **nodes**;
`link` is a typed **edge**; `event` is an append-only observation.

```json
{"tool": "init_app", "args": {
  "app": "brain",
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
    ],
    "link": [
      {"name": "from",     "value_type": "ref",  "indexed": true, "required": true},
      {"name": "to",       "value_type": "ref",  "indexed": true, "required": true},
      {"name": "rel_type", "value_type": "enum",
       "enum_values": ["works_at", "attended", "cites", "invested_in", "founded", "advises", "mentions"],
       "indexed": true, "required": true},
      {"name": "since",    "value_type": "date", "indexed": true},
      {"name": "note",     "value_type": "string"}
    ],
    "event": [
      {"name": "page",    "value_type": "ref",    "indexed": true, "required": true},
      {"name": "when",    "value_type": "date",   "indexed": true, "required": true, "default": "now"},
      {"name": "source",  "value_type": "string"},
      {"name": "summary", "value_type": "string", "vectorized": true, "required": true}
    ]
  }
}}
```

Why one app:

- **Atomic cross-type writes.** Creating a paper and its author edges
  in the same `write` is all-or-nothing.
- **Unified search surface.** "Anything about transformers" returns
  concepts, papers, and events in one `search` call — no per-app
  fan-out.
- **Agent cognition.** The agent remembers one namespace, not three.
- **Vec isolation is still per-type.** Synap stores embeddings in
  separate vec tables per `(app, type, field)`, so the `compiled_truth`
  space for `person` doesn't contaminate the one for `concept`.

Only go multi-app when the domains are genuinely independent (a
personal `brain` and a shared `team_wiki`, say). Related entity types
of the same domain belong in one app.

`updated_at: date` with `default: "now"` has the server stamp it on
every create; `value_type: date` rejects garbage like `"yesterday"` at
the boundary. `authors` and `org` are `ref` fields — `expand_refs`
follows them in a single query (see §4).

## 2. Ingest

> Read ~/notes/transformers.md, decide it's a concept page, and add
> it.

```json
{"tool": "read_file", "args": {"path": "~/notes/transformers.md"}}
// agent parses the markdown, extracts title/tags/body, then:
{"tool": "write", "args": {
  "app": "brain",
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

### Nodes + edges in one atomic batch

Because `link` lives in the same app, edges can be written *in the
same `write` call* as their nodes — one transaction, no split-brain:

```json
{"tool": "write", "args": {
  "app": "brain",
  "ops": [
    {"op": "create", "type": "paper",
     "data": {"slug": "attention-is-all-you-need",
              "title": "Attention Is All You Need",
              "published_at": "2017-06-12"},
     "returning": ["entity_id"]},
    {"op": "create", "type": "person",
     "data": {"slug": "vaswani", "title": "Ashish Vaswani"},
     "returning": ["entity_id"]}
  ]
}}
// → paper_id = pg_p1, author_id = pg_v1
{"tool": "write", "args": {
  "app": "brain",
  "ops": [{
    "op": "create", "type": "link",
    "data": {"from": "brain/pg_p1", "to": "brain/pg_v1", "rel_type": "cites"}
  }]
}}
```

(The second call exists only because the agent needs the entity_ids
from the first response. If you already know them, fold it into a
single batch.)

## 3. Query by structure

> What concepts are tagged with "nlp", most recently updated first?

```json
{"tool": "query", "args": {
  "app":     "brain",
  "type":    "concept",
  "filters": [{"field": "tags", "op": "array_contains", "value": "nlp"}],
  "order_by":[{"field": "updated_at", "direction": "desc"}],
  "select":  ["slug", "title", "updated_at"]
}}
```

### Cross-type catalog

Omit `type` to scan the whole brain:

```json
{"tool": "query", "args": {
  "app":      "brain",
  "filters":  [{"or": [
    {"field": "tags", "op": "array_contains", "value": "nlp"},
    {"field": "tags", "op": "array_contains", "value": "deep-learning"}
  ]}],
  "order_by": [{"field": "updated_at", "direction": "desc"}],
  "select":   ["slug", "title", "updated_at"],
  "limit":    100
}}
```

For a per-type tally (how much does the brain know?):

```json
{"tool": "query", "args": {
  "app":       "brain",
  "aggregate": [{"fn": "count", "as": "pages"}],
  "group_by":  "entity_type"
}}
// TSV: key       pages
//      concept   128
//      person    42
//      company   15
//      paper     71
//      link      203
//      event     640
```

## 4. Graph traversal

### One hop: who works at Acme?

```json
{"tool": "query", "args": {
  "app":     "brain",
  "type":    "link",
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
  "app": "brain", "type": "link",
  "filters": [
    {"field": "rel_type", "op": "eq",       "value": "attended"},
    {"field": "from",     "op": "ref_from", "value": "pg_alice"}
  ],
  "select": ["to"]
}}
// → meeting_ids = [m1, m2, ...]

// Step 2: everyone else who attended those same meetings.
{"tool": "query", "args": {
  "app": "brain", "type": "link",
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

Pages no edge points at — likely candidates for pruning or stronger
cross-references. Scoped to node types to skip edges and events:

```json
{"tool": "query", "args": {
  "app":     "brain",
  "filters": [
    {"field": "entity_type", "op": "in", "value": ["concept", "person", "company", "paper"]},
    {"field": "",            "op": "orphan", "value": true}
  ],
  "select":  ["slug", "title", "updated_at"]
}}
```

## 5. Semantic & temporal search

### "What does the brain think about agent frameworks?"

```json
{"tool": "search", "args": {
  "app":        "brain",
  "query_text": "agent framework design patterns",
  "rank":       "relevance",
  "limit":      10
}}
```

### "What's new about Alice?"

Events are a distinct type precisely so freshness works naturally —
`time_decay` blends similarity with recency:

```json
{"tool": "search", "args": {
  "app":        "brain",
  "type":       "event",
  "query_text": "Alice",
  "filters":    [{"field": "page", "op": "ref_to", "value": "pg_alice"}],
  "rank":       "time_decay",
  "limit":      10
}}
```

## 6. Evolve the compiled truth

As new evidence arrives, *update* the page's `compiled_truth` (the
current understanding) and *append* an event explaining what changed.
One atomic `write` against the brain:

```json
{"tool": "write", "args": {
  "app": "brain",
  "ops": [
    {"op": "update", "entity_id": "pg_transformers",
     "data": {"compiled_truth": "…updated narrative with new findings…"}},
    {"op": "create", "type": "event",
     "data": {
       "page":    "brain/pg_transformers",
       "source":  "paper:arxiv/2406.12345",
       "summary": "Revised understanding: FlashAttention-2 supplants the naïve attention kernel for production inference."
     }}
  ]
}}
```

The update is server-stamped (`default: "now"` on `updated_at`);
event `when` is auto-set the same way. Both land or both don't — the
page reflects the latest belief and the event preserves the why.

## Design tips

- **One app, many types.** A knowledge domain is one namespace. Use
  multi-type `init_app` — don't split into `brain.pages` /
  `brain.links` / `brain.events`. Atomic cross-type writes, unified
  search, and simpler agent mental model all collapse into this.
- **Edges as a type, not as inline ref arrays.** Storing relationships
  as `link` entities (with typed `rel_type`) lets you ask *why* two
  pages are connected; inline `related: [ref1, ref2]` arrays don't.
- **Vectorize selectively.** `compiled_truth` and event `summary` are
  worth embedding; `slug`, `tags`, `rel_type` are not — FTS5 keyword
  matching handles short strings better than dense vectors.
- **`expand_refs` + dot-path `select` is the win.** On a 100-edge
  query, one round-trip returns target titles; without it you'd fire
  100 follow-up queries and pay token cost for fields you didn't want.
- **`synap.memories` vs brain events.** Memories are *about the user*
  (preferences, biographical facts). Brain events are *about pages*
  (observations on external entities). Keep them separate — the
  subject and retention policy are different.
