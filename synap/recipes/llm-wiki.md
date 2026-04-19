---
title: LLM Wiki Recipe · Synap
---

# Recipe: LLM-maintained wiki

A personal wiki where **you curate raw sources** and **the LLM
maintains the pages** — summaries, entity pages, concept pages, and
an auto-generated index. Inspired by Karpathy's
[LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
design, but instead of loose markdown files this lives in a single
Synap database with hybrid search and structured queries built in.

The core separation:

1. **`wiki.sources`** — raw, immutable (articles, papers, transcripts,
   screenshots). You add these.
2. **`wiki.pages`** — LLM-maintained: summary / entity / concept /
   comparison pages with cross-references. The LLM writes these.
3. **`wiki.log`** — what the LLM did and when. Append-only.

The "index.md" file from the original design becomes a `query` call
against `wiki.pages`. No file to keep in sync; the catalog is always
live.

## 1. Design the schemas

### `wiki.sources` — raw material (human-curated)

```json
{"tool": "init_app", "args": {
  "app": "wiki.sources",
  "fields": [
    {"name": "title",       "value_type": "string", "indexed": true, "required": true},
    {"name": "kind",        "value_type": "enum",
     "enum_values": ["article", "paper", "transcript", "note", "image"],
     "indexed": true, "required": true},
    {"name": "url",         "value_type": "string"},
    {"name": "body",        "value_type": "string", "vectorized": true},
    {"name": "captured_at", "value_type": "date",   "indexed": true, "default": "now"}
  ]
}}
```

### `wiki.pages` — LLM-maintained knowledge

`types` gives each page shape its own field set while sharing the app.

```json
{"tool": "init_app", "args": {
  "app": "wiki.pages",
  "types": {
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
    ]
  }
}}
```

`ref` fields (`sources`, `related`, `subjects`) are arrays under the
hood — a summary can cite multiple sources, a concept can relate to
multiple other concepts. `expand_refs` follows them in one query.

### `wiki.log` — what the LLM did

```json
{"tool": "init_app", "args": {
  "app": "wiki.log",
  "fields": [
    {"name": "when",   "value_type": "date",   "indexed": true, "default": "now"},
    {"name": "action", "value_type": "enum",
     "enum_values": ["ingest", "summarize", "link", "update", "rename"],
     "indexed": true, "required": true},
    {"name": "target", "value_type": "ref"},
    {"name": "note",   "value_type": "string", "vectorized": true, "required": true}
  ]
}}
```

## 2. Ingest flow

User drops a paper in.

> I just read this paper. Add it and summarize.
> [paper contents]

```json
// 2a. Stash the source, verbatim.
{"tool": "write", "args": {
  "app": "wiki.sources",
  "ops": [{"op": "create",
           "data": {"title": "Attention Is All You Need",
                    "kind":  "paper",
                    "url":   "https://arxiv.org/abs/1706.03762",
                    "body":  "…full paper text…"},
           "returning": ["entity_id"]}]
}}
// → src_abc

// 2b. LLM writes a summary page citing it.
{"tool": "write", "args": {
  "app": "wiki.pages",
  "ops": [{"op": "create", "type": "summary",
           "data": {"slug":    "summary/attention-is-all-you-need",
                    "title":   "Summary: Attention Is All You Need",
                    "body":    "Introduces the Transformer architecture…",
                    "sources": ["wiki.sources/src_abc"],
                    "tags":    ["transformers", "architecture"]}}]
}}

// 2c. LLM may spin off entity / concept pages mentioned in the source.
{"tool": "write", "args": {
  "app": "wiki.pages",
  "ops": [
    {"op": "create", "type": "entity",
     "data": {"slug": "entity/vaswani-ashish", "title": "Ashish Vaswani",
              "body": "Co-author of …"}},
    {"op": "create", "type": "concept",
     "data": {"slug": "concept/self-attention", "title": "Self-attention",
              "body": "A mechanism where each token …"}}
  ]
}}

// 2d. Log the ingest for auditability.
{"tool": "write", "args": {
  "app": "wiki.log",
  "ops": [{"op": "create",
           "data": {"action": "ingest",
                    "target": "wiki.sources/src_abc",
                    "note":   "Ingested Attention paper; created 1 summary, 1 entity, 1 concept page."}}]
}}
```

`returning: ["entity_id"]` on the source create avoids a follow-up
`query` just to get the id for the cross-reference. The log entry
carries a `ref` to the source (not the summary) because ingests are
*about the source*; each downstream write also creates follow-up log
entries.

## 3. The live index (replaces `index.md`)

### By category

```json
{"tool": "query", "args": {
  "app":      "wiki.pages",
  "select":   ["slug", "title", "type", "updated"],
  "order_by": [{"field": "type",    "direction": "asc"},
               {"field": "updated", "direction": "desc"}],
  "limit":    200
}}
```

### Page counts per type

```json
{"tool": "query", "args": {
  "app":       "wiki.pages",
  "aggregate": [{"fn": "count", "as": "pages"}],
  "group_by":  "type"
}}
// TSV: key         pages
//      summary     14
//      entity      31
//      concept     22
//      comparison  3
```

### Recent activity

```json
{"tool": "query", "args": {
  "app":      "wiki.log",
  "order_by": [{"field": "when", "direction": "desc"}],
  "limit":    20,
  "select":   ["when", "action", "note"]
}}
```

## 4. Search — meaning, not just filename

### "Find me everything related to attention"

```json
{"tool": "search", "args": {
  "app":        "wiki.pages",
  "query_text": "attention mechanism",
  "rank":       "relevance",
  "limit":      10
}}
```

### "What did I add recently about transformers?"

Hybrid freshness + similarity:

```json
{"tool": "search", "args": {
  "app":        "wiki.pages",
  "query_text": "transformers",
  "rank":       "time_decay",
  "limit":      10
}}
```

### Scope to a type

```json
{"tool": "search", "args": {
  "app":        "wiki.pages",
  "type":       "concept",
  "query_text": "positional encoding",
  "rank":       "relevance"
}}
```

### Cross-app search

Leave `app` off to search the whole wiki — sources, pages, and log at
once:

```json
{"tool": "search", "args": {
  "query_text": "flash attention",
  "rank":       "relevance"
}}
```

## 5. Follow the link graph

### "Show me the sources behind this summary page"

```json
{"tool": "query", "args": {
  "app":     "wiki.pages",
  "filters": [{"field": "slug", "op": "eq", "value": "summary/attention-is-all-you-need"}],
  "expand_refs": 1,
  "select":  ["title", "sources.title", "sources.url"]
}}
```

One call returns the page title plus the title + URL of every cited
source. The `_ref` / `_entity_id` sentinels that `expand_refs` adds
by default are dropped when you project dot-paths, so the response
is lean.

### "What pages cite this source?" (backlinks)

```json
{"tool": "query", "args": {
  "app":     "wiki.pages",
  "filters": [{"field": "sources", "op": "ref_to", "value": "src_abc"}],
  "select":  ["slug", "title", "type"]
}}
```

### Orphaned pages (no incoming cross-references)

```json
{"tool": "query", "args": {
  "app":     "wiki.pages",
  "filters": [{"field": "", "op": "orphan", "value": true}],
  "select":  ["slug", "title", "type", "updated"]
}}
```

## 6. The LLM-as-maintainer loop

When the user adds a new source, the agent should:

1. `write wiki.sources` — stash raw (always keep the original).
2. Read existing context: `search wiki.pages` for related pages.
3. Decide what to create or update:
   - New concept → new `concept` page.
   - Existing concept gets new evidence → `update` its body, append
     a `wiki.log` event.
   - Contradiction found → update both sides, log a `rename` / `update`.
4. Wire cross-references: `related`, `sources`, `subjects` refs.
5. `write wiki.log` — one entry per action for later auditing.

## Design tips

- **Don't edit `wiki.sources`.** Treat it as immutable. If the user
  annotates a source, create a `summary` page pointing at it instead.
- **Keep `slug` human-readable**: `concept/self-attention` beats
  `c_7x2qz8`. Makes the log + index legible.
- **One `wiki.log` entry per logical action**, not per SQL write. A
  three-page ingest = one log row, not three.
- **Use `ref` arrays, not stringified slugs**. `sources` is a
  `ref`-type field; writes pass `["wiki.sources/src_abc"]`, and
  `expand_refs` can follow them. Stringifying defeats `ref_to` /
  `ref_from` / `orphan` — all the graph tools.
- **`synap.memories` for the user, `wiki.log` for the wiki**. Don't
  fold wiki-ingest events into personal memory; they're about the
  system, not about *you*.
