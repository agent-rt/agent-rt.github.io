---
title: Reference · Synap
---

# Reference

Every MCP tool Synap exposes. Tool names appear to the model with the
`mcp__synap__` prefix.

## Memory tools

### `remember`

Write to `synap.memories`. Use for user facts and timestamped events.

| Arg         | Type     | Notes |
|-------------|----------|-------|
| `type`      | `"attr"` \| `"event"` | required |
| `subject`   | string   | defaults to `"user"` |
| `attribute` | string   | required when `type: "attr"` |
| `op`        | `"set"` \| `"append"` \| `"clear"` | for `type: "attr"` |
| `value`     | any      | attr value (scalar or list) |
| `description` | string | required when `type: "event"`; vectorized |
| `when`      | date     | for `type: "event"`; defaults to now |

### `forget`

Precise memory deletion. Either clears one attribute or removes one event.

| Arg         | Type   | Notes |
|-------------|--------|-------|
| `type`      | `"attr"` \| `"event"` | required |
| `subject`   | string | defaults to `"user"` |
| `attribute` | string | for `type: "attr"` |
| `entity_id` | string | for `type: "event"` |

## App management

### `init_app`

Create or evolve an app schema. Idempotent.

| Arg           | Type   | Notes |
|---------------|--------|-------|
| `name`        | string | app id; no `synap.` prefix |
| `description` | string | human-facing summary |
| `fields`      | object | simple schema (single entity type) |
| `types`       | object | multi-type schema: `{typename: {fields: ...}}` |

Field types: `text`, `number`, `date`, `enum` (with `values`), `bool`.
Modifiers: `required`, `default`, `multi`, `vectorized`.

### `list_apps`

List apps with schemas. Pass `include_internal: true` to also see
`synap.*` apps.

## Reading entities

### `query`

Structured filter / sort / paginate. TSV output.

| Arg           | Type     | Notes |
|---------------|----------|-------|
| `app_id`      | string   | required |
| `entity_type` | string   | optional, for multi-type apps |
| `filters`     | array    | `[{field, op, value}]` — ops: `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`, `contains`, `like` |
| `sort`        | array    | `[{field, dir}]` — `dir`: `asc` \| `desc` |
| `select`      | string[] | project specific fields |
| `limit`, `offset` | number | pagination; or pass `cursor` |

### `search`

Hybrid semantic + keyword search. TSV output with snippet.

| Arg           | Type   | Notes |
|---------------|--------|-------|
| `query`       | string | required |
| `app_id`      | string | scope to one app; omit for cross-app |
| `entity_type` | string | optional |
| `filters`     | array  | same shape as `query` |
| `rank`        | `"relevance"` \| `"recency"` \| `"time_decay"` | default `"relevance"` |
| `limit`       | number | default 10 |

## Writing entities

### `write`

Create / update / delete. `ops` is an array — **1 op = single write, N
ops = atomic batch (all-or-nothing)**.

| Op shape (entries in `ops`)                              | Notes |
|----------------------------------------------------------|-------|
| `{op: "create", entity: {...}, entity_type?: "..."}`     | returns new `entity_id` |
| `{op: "update", entity_id: "...", entity: {...}}`        | partial update (EAV) |
| `{op: "delete", entity_id: "..."}`                       |       |

## Files and links

### `read_file`

Read local files or `synap://` / `file://` URIs. Auto-paginated (~4K
chars/page).

| Arg    | Type   | Notes |
|--------|--------|-------|
| `path` | string | absolute path or URI |
| `page` | number | default 0 |

### `scan_dir`

List directory contents with optional previews. TSV output.

### `detect_links`

Opt-in: find semantic relationships between entities. Useful for
building knowledge graphs from accumulated notes.

## Context management

### `expand`

Recover the verbatim text of a `<compressed_segment turn_ids="...">` or
`<stale_tool_result turn_id="...">` block. Pass the `turn_ids` from the
block.

## Embeddings (maintenance)

### `embed`

| Arg    | Type   | Notes |
|--------|--------|-------|
| `op`   | `"status"` \| `"reindex"` | |
| `app_id`, `field` | string | for `"reindex"` |

Rarely needed — embeddings auto-update on write.

## Dashboard views (experimental)

### `patch_view`

Incremental updates to a view spec stored in `synap.views`.

---

## Response formats

- **JSON**: `remember`, `forget`, `init_app`, `write`, `read_file`,
  `detect_links`, `embed`, `patch_view`, `expand`.
- **TSV**: `query`, `search`, `list_apps`, `scan_dir`.

TSV is chosen for multi-row reads to minimize context bytes vs. JSON.
Tabs separate columns; first row is the header.
