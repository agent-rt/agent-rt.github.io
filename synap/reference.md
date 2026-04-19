---
title: Reference · Synap
---

# Reference

Every MCP tool Synap exposes. Tool names appear to the model prefixed with
`mcp__synap__`. Shapes here match the JSON Schemas in
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

Dates are stored as `string` (ISO 8601) or `number` (unix seconds) —
there's no dedicated date type.

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

### Filter operators

`eq`, `ne`, `gt`, `gte`, `lt`, `lte`, `contains`, `in`, `not_in`,
`between`, `is_null`, `is_not_null`, `array_contains`, `array_overlaps`,
`orphan`, `ref_to`, `ref_from`.

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
| `app_id` | string | required; no `synap.` prefix for user apps |
| `types`  | object | required; `{typename: [field_spec, …]}` |

### `list_apps`

| Arg                | Type    | Notes |
|--------------------|---------|-------|
| `include_internal` | boolean | default false; show `synap.*` apps |

## Writing

### `write`

Create / update / delete. `ops` is an array — **1 op = single write, N ops
= atomic batch**.

```json
{"app_id": "tasks", "ops": [ <op>, <op>, ... ]}
```

Op shapes:

| Op | Shape |
|----|-------|
| create | `{"op": "create", "entity_type": "task", "data": {...}, "upsert_by": "slug"}` |
| update | `{"op": "update", "entity_id": "...", "data": {...}}` |
| delete | `{"op": "delete", "entity_id": "..."}` |

`upsert_by` (create only): if an indexed field in `data` matches an
existing row, update instead of inserting.

## Reading

### `query`

Structured filter / sort / paginate. TSV output.

| Arg              | Type     | Notes |
|------------------|----------|-------|
| `app_id`         | string   | required |
| `entity_type`    | string   | optional |
| `filters`        | array    | `[{field, op, value}]` |
| `order_by`       | array    | `[{field, direction}]` — `direction`: `asc` \| `desc` |
| `fields`         | string[] | project specific fields |
| `limit`, `offset`| integer  | pagination |
| `cursor`         | string   | from previous `next_cursor` |
| `ref_depth`      | integer  | expand `ref` fields |
| `count_only`     | boolean  | return just the count |
| `group_by`       | string   | with `count_only`, group counts |
| `include_degree` | boolean  | include in/out link counts per entity |
| `full`           | boolean  | skip auto-truncation (use sparingly) |

### `search`

Hybrid semantic + keyword search. TSV output with snippets.

| Arg              | Type   | Notes |
|------------------|--------|-------|
| `query_text`     | string | required |
| `app_id`         | string | omit for cross-app |
| `entity_type`    | string | optional |
| `mode`           | `"semantic"` \| `"keyword"` \| `"hybrid"` | default `semantic` |
| `filters`        | array  | same as `query` |
| `rank`           | `"relevance"` \| `"recency"` \| `"time_decay"` | default `relevance` |
| `rank_field`     | string | timestamp field for recency/time_decay (default `when`) |
| `half_life_days` | number | `time_decay` (default 30) |
| `decay_weight`   | number | `time_decay` (default 0.5) |
| `fields`         | string[] | project |
| `limit`          | integer | default 10 |

## Context management

### `expand`

Recover the verbatim text of a `<compressed_segment turn_ids="...">` or
`<stale_tool_result turn_id="...">` block.

| Arg        | Type     | Notes |
|------------|----------|-------|
| `turn_ids` | string[] | required; ids from the block |

---

## Response formats

- **JSON** — `remember`, `forget`, `init_app`, `write`, `expand`, and
  other control tools.
- **TSV** — `query`, `search`, `list_apps`. Dense, context-cheap; first
  row is the header.
