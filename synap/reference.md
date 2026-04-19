---
title: Reference · Synap
---

# Reference

Every MCP tool Synap exposes. Tool names appear to the model prefixed
with `mcp__synap__`. Shapes here match the JSON Schemas in
`synap-tools/src/schema.rs` — the single source of truth.

## Core types

### `value_type` (field kind)

- `string` — short or long text. Can be `vectorized: true` for embedding.
- `number` — f64.
- `boolean` — true/false.
- `enum` — fixed set via `enum_values: [...]` (optionally labelled with
  `enum_labels: {value: "Label"}`).
- `ref` — reference to another entity by id (cross-entity graph).
- `array` — list-valued; use for tags, authors, categories.
- `json` — arbitrary blob; opaque to filters/search.
- `file` — local path or `synap://` URI. Can be `vectorized: true` — text
  files get auto-chunked + embedded.
- `date` — ISO 8601 string, validated on write. Accepts `YYYY-MM-DD`,
  `YYYY-MM-DDTHH:MM:SS`, `…Z`, `…±HH:MM`. Range filters work the same
  as string lex-compare, but malformed inputs get rejected up front.
  `default: "now"` resolves to the current UTC timestamp on create.

### Field spec

```json
{
  "name":       "status",
  "value_type": "enum",
  "enum_values": ["todo", "doing", "done"],
  "enum_labels": {"todo": "To do", "doing": "In progress"},
  "label":       "Status",
  "indexed":     true,
  "vectorized":  false
}
```

`indexed: true` enables B-tree lookups for fast filtering. `vectorized:
true` enables semantic search on string/file fields.

Two write-time modifiers:

- **`required: true`** — `write` create ops without this field get
  rejected with a `ValidationError`. Does not apply to `update`;
  partial updates remain legal.
- **`default: <value>`** — server fills this value on create when the
  field is absent. Type must match `value_type`; enum defaults must be
  one of `enum_values`.

### Filter operators

`eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `contains`, `in`, `not_in`,
`between`, `is_null`, `is_not_null`, `array_contains`, `array_overlaps`,
`orphan`, `ref_to`, `ref_from`.

### Boolean combinators

`filters` elements can be leaves **or** boolean nodes — `{and: [...]}`,
`{or: [...]}`, `{not: <node>}`. The top-level array is an implicit AND.
Nesting is unbounded:

```json
"filters": [
  {"or": [
    {"field": "priority", "op": "eq", "value": "high"},
    {"field": "due",      "op": "lte", "value": "2026-04-25"}
  ]},
  {"field": "status", "op": "in", "value": ["todo", "doing"]}
]
```

Ref ops (`ref_to` / `ref_from` / `orphan`) must remain as top-level
leaves — they're resolved via entity-id lookup rather than SQL and
nesting them inside an `or`/`not` returns a validation error.

## Memory tools

### `remember`

Write to `synap.memories`. The only legal write path to `synap.*`.

| Arg           | Type     | Notes |
|---------------|----------|-------|
| `type`        | `"attr"` \| `"event"` | required |
| `subject`     | string   | defaults to current user |
| `attribute`   | string   | `attr` only — snake_case name |
| `op`          | `"set"` \| `"append"` \| `"clear"` | `attr` only |
| `value`       | any      | `attr` value (scalar for set; element for append/clear) |
| `description` | string   | `event` only — vectorized |
| `when`        | integer  | `event` only — unix seconds; omit for "now" |

### `forget`

| Arg         | Type   | Notes |
|-------------|--------|-------|
| `type`      | `"attr"` \| `"event"` | required |
| `subject`   | string | defaults to current user |
| `attribute` | string | `attr` only |
| `id`        | string | `event` only — entity_id from a query on `synap.memories` |

## App management

### `init_app`

Create or evolve an app. Idempotent.

| Arg      | Type   | Notes |
|----------|--------|-------|
| `app`    | string | required; no `synap.` prefix for user apps |
| `fields` | array  | single-entity-type shorthand; equivalent to `types: {<app>: fields}`. Mutually exclusive with `types`. |
| `types`  | object | multi-type schema: `{typename: [field_spec, …]}` |

At least one of `fields` or `types` is required. Use `fields` for
single-shape apps (tasks, kb, recipes, …) and `types` when one app
holds multiple shapes.

### `list_apps`

| Arg                | Type    | Notes |
|--------------------|---------|-------|
| `include_internal` | boolean | default false; show `synap.*` apps |

## Writing

### `write`

Create / update / delete. `ops` is an array — **1 op = single write, N
ops = atomic batch**.

```json
{"app": "tasks", "ops": [ <op>, <op>, ... ]}
```

Op shapes:

| Op     | Shape |
|--------|-------|
| create | `{"op": "create", "type": "task", "data": {...}, "upsert_by": "slug", "returning": true}` |
| update | `{"op": "update", "entity_id": "...", "data": {...}, "returning": ["status"]}` |
| delete | `{"op": "delete", "entity_id": "..."}` |

`type` on create is only needed for multi-type apps. `upsert_by` (create
only): if an indexed field in `data` matches an existing row, update
instead of inserting.

**`returning`** (create / update): echo the persisted row so the caller
doesn't need a follow-up query. Two shapes:

- `"returning": true` — return the full row (entity_id + all fields,
  including server-filled defaults).
- `"returning": ["field1", "field2"]` — project to the listed fields.
  `entity_id` is always included.

When any op in `ops` uses `returning`, the response passes through the
per-op batch array (with `entities` / `entity` keys populated) instead
of the compact `{pushed, updated, deleted}` summary.

## Reading

### `query`

Structured filter / sort / paginate. TSV output.

| Arg              | Type     | Notes |
|------------------|----------|-------|
| `app`            | string   | required |
| `type`           | string   | optional entity type filter |
| `filters`        | array    | `[{field, op, value}]` |
| `order_by`       | array    | `[{field, direction}]` — `direction`: `asc` \| `desc` |
| `select`         | string[] | project specific fields |
| `limit`, `offset`| integer  | pagination |
| `cursor`         | string   | from previous `next_cursor` |
| `ref_depth`      | integer  | expand `ref` fields |
| `count_only`     | boolean  | return just the count |
| `group_by`       | string   | partitions rows (for `aggregate` or `count_only`) |
| `aggregate`      | array    | `[{fn, field?, as?}]` — compute sum/avg/min/max/count; emits `{key?, values}` rows instead of entities |
| `include_degree` | boolean  | include in/out link counts per entity |
| `full`           | boolean  | skip auto-truncation (use sparingly) |

### `search`

Hybrid semantic + keyword search. TSV output with snippets.

| Arg              | Type   | Notes |
|------------------|--------|-------|
| `query_text`     | string | required |
| `app`            | string | omit for cross-app |
| `type`           | string | optional entity type filter |
| `mode`           | `"semantic"` \| `"keyword"` \| `"hybrid"` | default `semantic` |
| `filters`        | array  | same as `query` |
| `rank`           | `"relevance"` \| `"recency"` \| `"time_decay"` | default `relevance` |
| `rank_field`     | string | timestamp field for recency/time_decay (default `when`) |
| `half_life_days` | number | `time_decay` (default 30) |
| `decay_weight`   | number | `time_decay` (default 0.5) |
| `select`         | string[] | project specific fields |
| `limit`          | integer | default 10 |

## Context management

### `expand_context`

Recover the verbatim text of a `<compressed_segment turn_ids="...">` or
`<stale_tool_result turn_id="...">` block.

| Arg        | Type     | Notes |
|------------|----------|-------|
| `turn_ids` | string[] | required; ids from the block |

## Embedding maintenance

### `embed_status`

| Arg   | Type   | Notes |
|-------|--------|-------|
| `app` | string | optional; limit count to one app |

Returns `{pending, indexed, total}`.

### `reindex`

Enqueue re-embedding for vectorized fields. Runs asynchronously.

| Arg     | Type   | Notes |
|---------|--------|-------|
| `app`   | string | required |
| `type`  | string | optional entity type filter |
| `field` | string | optional; only this vectorized field |

## Files and links

### `read_file`

Read local files or `synap://` / `file://` URIs. Auto-paginated (~4K
chars/page).

| Arg      | Type   | Notes |
|----------|--------|-------|
| `uri`    | string | absolute path or URI |
| `offset` | number | start line (0-based) |
| `limit`  | number | lines to read |

### `scan_dir`

List directory contents with optional previews. TSV output.

### `detect_links`

Opt-in: find semantic relationships between entities. Useful for
building knowledge graphs from accumulated notes.

### `patch_view`

Incremental updates to a view spec stored in `synap.views`.

---

## Response formats

- **JSON**: `remember`, `forget`, `init_app`, `write`, `read_file`,
  `detect_links`, `patch_view`, `expand_context`, `embed_status`,
  `reindex`.
- **TSV**: `query`, `search`, `list_apps`, `scan_dir`.

TSV is chosen for multi-row reads to minimize context bytes vs. JSON.
Tabs separate columns; first row is the header.
