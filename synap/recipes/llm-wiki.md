---
title: LLM Wiki Recipe · Synap
---

# Recipe: LLM-maintained wiki

A personal wiki where **you curate raw sources** and **the LLM
maintains the pages** — summaries, entity pages, concept pages, and
an auto-generated index. Inspired by Karpathy's
[LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
design, but instead of loose markdown files this lives in a single
Synap app with hybrid search and structured queries built in.

The core separation is by **entity type within one app**:

- `source` — raw, immutable (articles, papers, transcripts,
  screenshots). Human-added.
- `summary` / `entity` / `concept` / `comparison` — LLM-maintained
  pages with cross-references.
- `log` — append-only record of what the LLM did.

All six live in the same `wiki` app. Karpathy's `index.md` becomes a
`query` against `wiki`. No file to keep in sync; the catalog is
always live.

## 1. Design the schema — one app, many types

```json
{"tool": "init_app", "args": {
  "app": "wiki",
  "types": {
    "source": [
      {"name": "title",       "value_type": "string", "indexed": true, "required": true},
      {"name": "kind",        "value_type": "enum",
       "enum_values": ["article", "paper", "transcript", "note", "image"],
       "indexed": true, "required": true},
      {"name": "url",         "value_type": "string"},
      {"name": "body",        "value_type": "string", "vectorized": true},
      {"name": "captured_at", "value_type": "date",   "indexed": true, "default": "now"}
    ],
    "summary": [
      {"name": "slug",     "value_type": "string", "indexed": true, "required": true},
      {"name": "title",    "value_type": "string", "indexed": true, "vectorized": true, "required": true},
      {"name": "body",     "value_type": "string", "vectorized": true},
      {"name": "sources",  "value_type": "ref"},
      {"name": "tags",     "value_type": "array"},
      {"name": "updated",  "value_type": "date",   "indexed": true, "default": "now"}
    ],
    "entity": [
      {"name": "slug",     "value_type": "string", "indexed": true, "required": true},
      {"name": "title",    "value_type": "string", "indexed": true, "vectorized": true, "required": true},
      {"name": "body",     "value_type": "string", "vectorized": true},
      {"name": "related",  "value_type": "ref"},
      {"name": "tags",     "value_type": "array"},
      {"name": "updated",  "value_type": "date",   "indexed": true, "default": "now"}
    ],
    "concept": [
      {"name": "slug",     "value_type": "string", "indexed": true, "required": true},
      {"name": "title",    "value_type": "string", "indexed": true, "vectorized": true, "required": true},
      {"name": "body",     "value_type": "string", "vectorized": true},
      {"name": "related",  "value_type": "ref"},
      {"name": "tags",     "value_type": "array"},
      {"name": "updated",  "value_type": "date",   "indexed": true, "default": "now"}
    ],
    "comparison": [
      {"name": "slug",     "value_type": "string", "indexed": true, "required": true},
      {"name": "title",    "value_type": "string", "indexed": true, "vectorized": true, "required": true},
      {"name": "body",     "value_type": "string", "vectorized": true},
      {"name": "subjects", "value_type": "ref",    "required": true},
      {"name": "updated",  "value_type": "date",   "indexed": true, "default": "now"}
    ],
    "log": [
      {"name": "when",   "value_type": "date",   "indexed": true, "default": "now"},
      {"name": "action", "value_type": "enum",
       "enum_values": ["ingest", "summarize", "link", "update", "rename"],
       "indexed": true, "required": true},
      {"name": "target", "value_type": "ref"},
      {"name": "note",   "value_type": "string", "vectorized": true, "required": true}
    ]
  }
}}
```

Why one app:

- **Atomic ingests.** Creating a source, a summary citing it, and a
  log entry lands in one `write` — all three or nothing.
- **Unified search.** "Flash attention" asks once against `wiki` and
  finds the source, the concept page, and the ingest log line in
  one call.
- **Live index.** A single `query` partitions by `entity_type` for
  the catalog — no cross-app joins.

`ref` fields (`sources`, `related`, `subjects`, `target`) are arrays
under the hood — a summary can cite multiple sources, a concept can
relate to multiple others. `expand_refs` follows them in one query.

## 2. Ingest flow

User drops a paper in.

> I just read this paper. Add it and summarize.
> [paper contents]

```json
{"tool": "write", "args": {
  "app": "wiki",
  "ops": [
    {"op": "create", "type": "source",
     "data": {"title": "Attention Is All You Need",
              "kind":  "paper",
              "url":   "https://arxiv.org/abs/1706.03762",
              "body":  "…full paper text…"},
     "returning": ["entity_id"]},
    {"op": "create", "type": "entity",
     "data": {"slug":  "entity/vaswani-ashish",
              "title": "Ashish Vaswani",
              "body":  "Co-author of …"}},
    {"op": "create", "type": "concept",
     "data": {"slug":  "concept/self-attention",
              "title": "Self-attention",
              "body":  "A mechanism where each token …"}}
  ]
}}
// → source_id = src_abc, entity_id = ent_v1, concept_id = cpt_sa
```

Then wire the summary (it needs the source_id) + log entry atomically:

```json
{"tool": "write", "args": {
  "app": "wiki",
  "ops": [
    {"op": "create", "type": "summary",
     "data": {"slug":    "summary/attention-is-all-you-need",
              "title":   "Summary: Attention Is All You Need",
              "body":    "Introduces the Transformer architecture…",
              "sources": ["wiki/src_abc"],
              "tags":    ["transformers", "architecture"]}},
    {"op": "create", "type": "log",
     "data": {"action": "ingest",
              "target": "wiki/src_abc",
              "note":   "Ingested Attention paper; created 1 summary, 1 entity, 1 concept page."}}
  ]
}}
```

`returning: ["entity_id"]` on the source create avoids a follow-up
`query` just to get the id. The log entry carries a `ref` to the
source because the ingest is *about the source*; downstream
refinements can log their own entries pointing at the specific
summary or concept they edit.

## 3. The live index (replaces `index.md`)

### By category

```json
{"tool": "query", "args": {
  "app":      "wiki",
  "filters":  [{"field": "entity_type", "op": "in",
                "value": ["summary", "entity", "concept", "comparison"]}],
  "select":   ["slug", "title", "entity_type", "updated"],
  "order_by": [{"field": "entity_type", "direction": "asc"},
               {"field": "updated",     "direction": "desc"}],
  "limit":    200
}}
```

(Excluding `source` and `log` from the page index — those are their
own views below.)

### Page counts per type

```json
{"tool": "query", "args": {
  "app":       "wiki",
  "aggregate": [{"fn": "count", "as": "n"}],
  "group_by":  "entity_type"
}}
// TSV: key         n
//      source      27
//      summary     14
//      entity      31
//      concept     22
//      comparison  3
//      log         98
```

### Recent activity

```json
{"tool": "query", "args": {
  "app":      "wiki",
  "type":     "log",
  "order_by": [{"field": "when", "direction": "desc"}],
  "limit":    20,
  "select":   ["when", "action", "note"]
}}
```

## 4. Search — meaning, not just filename

### "Find me everything related to attention"

Cross-type search over the whole wiki:

```json
{"tool": "search", "args": {
  "app":        "wiki",
  "query_text": "attention mechanism",
  "rank":       "relevance",
  "limit":      10
}}
```

### "What did I add recently about transformers?"

Hybrid freshness + similarity:

```json
{"tool": "search", "args": {
  "app":        "wiki",
  "query_text": "transformers",
  "rank":       "time_decay",
  "limit":      10
}}
```

### Scope to one type

```json
{"tool": "search", "args": {
  "app":        "wiki",
  "type":       "concept",
  "query_text": "positional encoding",
  "rank":       "relevance"
}}
```

## 5. Follow the link graph

### "Show me the sources behind this summary page"

```json
{"tool": "query", "args": {
  "app":         "wiki",
  "type":        "summary",
  "filters":     [{"field": "slug", "op": "eq", "value": "summary/attention-is-all-you-need"}],
  "expand_refs": 1,
  "select":      ["title", "sources.title", "sources.url"]
}}
```

One call returns the page title plus the title + URL of every cited
source. The `_ref` / `_entity_id` sentinels that `expand_refs` adds
by default are dropped when you project dot-paths, so the response
stays lean.

### "What pages cite this source?" (backlinks)

```json
{"tool": "query", "args": {
  "app":     "wiki",
  "filters": [{"field": "sources", "op": "ref_to", "value": "src_abc"}],
  "select":  ["slug", "title", "entity_type"]
}}
```

No `type` filter — any page whose `sources` points at `src_abc`
comes back, regardless of whether it's a summary or a comparison.

### Orphaned pages (no incoming cross-references)

```json
{"tool": "query", "args": {
  "app":     "wiki",
  "filters": [
    {"field": "entity_type", "op": "in", "value": ["summary", "entity", "concept", "comparison"]},
    {"field": "",            "op": "orphan", "value": true}
  ],
  "select":  ["slug", "title", "entity_type", "updated"]
}}
```

## 6. The LLM-as-maintainer loop

When the user adds a new source, the agent should:

1. `write wiki` — stash the `source` (always keep the original).
2. Read existing context: `search wiki` for related pages.
3. Decide what to create or update:
   - New concept → new `concept` page.
   - Existing concept gets new evidence → `update` its body, append
     a `log` event.
   - Contradiction found → update both sides, log a `rename` / `update`.
4. Wire cross-references: `related`, `sources`, `subjects` refs.
5. One `log` entry per logical action (not per SQL write).

Because everything is in one app, the whole step can be a single
atomic `write` batch when the agent knows all the target ids up
front.

## Design tips

- **One app, many types.** Sources, pages, and logs are one wiki.
  Multi-type `init_app` keeps them in one namespace — atomic
  ingests, unified search, single catalog query.
- **Don't edit `source` entities.** Treat them as immutable. If the
  user annotates a source, create a `summary` pointing at it instead.
- **Keep `slug` human-readable.** `concept/self-attention` beats
  `c_7x2qz8`. Makes the log + index legible.
- **One `log` entry per logical action**, not per SQL write. A
  three-page ingest is one `log` row, not three.
- **Use `ref` arrays, not stringified slugs.** `sources` is a
  `ref`-type field; writes pass `["wiki/src_abc"]`, and `expand_refs`
  can follow them. Stringifying defeats `ref_to` / `ref_from` /
  `orphan` — all the graph tools.
- **`synap.memories` for the user, `wiki/log` for the wiki.** Don't
  fold wiki-ingest events into personal memory; they're about the
  system, not about *you*.
