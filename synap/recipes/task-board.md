---
title: Task Board Recipe · Synap
---

# Recipe: Task board

A tasks app with statuses, due dates, tags, and semantic search over
notes.

## 1. Design

> Make a synap app "tasks" with: title, status (enum todo/doing/done/
> cancelled), priority (enum low/med/high), due, tags (array),
> notes (vectorized), created_at.

```json
{"tool": "init_app", "args": {
  "app_id": "tasks",
  "fields": [
    {"name": "title",      "value_type": "string", "indexed": true, "required": true},
    {"name": "status",     "value_type": "enum",
     "enum_values": ["todo", "doing", "done", "cancelled"],
     "indexed": true, "default": "todo"},
    {"name": "priority",   "value_type": "enum",
     "enum_values": ["low", "med", "high"], "default": "med"},
    {"name": "due",        "value_type": "string", "indexed": true},
    {"name": "tags",       "value_type": "array"},
    {"name": "notes",      "value_type": "string", "vectorized": true},
    {"name": "created_at", "value_type": "string"}
  ]
}}
```

Dates live as ISO 8601 `string` — lexicographic compare gives correct
ranges. `indexed: true` on `status` and `due` because those are the
common filter/sort axes. `required: true` on `title` + `default: "todo"`
/ `"med"` on the enums mean the server rejects half-populated creates
and fills sensible defaults — agents don't have to repeat them in
every write.

## 2. Add tasks

> Add tasks:
> - "Ship Synap 0.1.2" due 2026-05-01, high priority
> - "Write release notes" due 2026-04-29, tagged docs
> - "Code-sign macOS binary" due 2026-05-10, notes: "need Apple Developer ID ($99/yr), notarize via notarytool, embed timestamp"

```json
{"tool": "write", "args": {
  "app_id": "tasks",
  "ops": [
    {"op": "create", "data": {
      "title": "Ship Synap 0.1.2", "priority": "high",
      "due": "2026-05-01", "created_at": "2026-04-19"
    }},
    {"op": "create", "data": {
      "title": "Write release notes", "tags": ["docs"],
      "due": "2026-04-29", "created_at": "2026-04-19"
    }},
    {"op": "create", "data": {
      "title": "Code-sign macOS binary",
      "due": "2026-05-10", "created_at": "2026-04-19",
      "notes": "need Apple Developer ID ($99/yr), notarize via notarytool, embed timestamp"
    }}
  ]
}}
```

One call, atomic. `status` defaults to `"todo"` and `priority` to
`"med"` via the schema — only the overriding row (first one:
`"priority": "high"`) needs to spell them out.

## 3. Everyday queries

> What's on my plate today?

```json
{"tool": "query", "args": {
  "app_id": "tasks",
  "filters": [
    {"field": "status", "op": "in",  "value": ["todo", "doing"]},
    {"field": "due",    "op": "lte", "value": "2026-04-20"}
  ],
  "order_by": [
    {"field": "priority", "direction": "desc"},
    {"field": "due",      "direction": "asc"}
  ]
}}
```

> Show me everything I finished this month.

```json
{"tool": "query", "args": {
  "app_id": "tasks",
  "filters": [
    {"field": "status", "op": "eq", "value": "done"},
    {"field": "due",    "op": "between", "value": ["2026-04-01", "2026-04-30"]}
  ],
  "order_by": [{"field": "due", "direction": "desc"}]
}}
```

`between` is inclusive on both ends.

## 4. Update state

> Mark "Write release notes" as done.

```json
{"tool": "write", "args": {
  "app_id": "tasks",
  "ops": [{
    "op": "update",
    "entity_id": "<id-from-query>",
    "data": {"status": "done"}
  }]
}}
```

EAV update — only the changed attribute is written, not the whole row.

## 5. Semantic recall on notes

> Do I have any tasks about code signing?

```json
{"tool": "search", "args": {
  "app_id":     "tasks",
  "query_text": "code signing certificate notarization",
  "mode":       "semantic"
}}
```

Finds the macOS binary task by `notes` content even though the title
doesn't mention "certificate" or "notarization".

## 6. Cross-app status report

> Give me a status snapshot: open tasks, notes captured this week,
> recent decisions I've remembered.

An agent composes three calls:

```json
{"tool": "query",  "args": {"app_id": "tasks",
  "filters": [{"field": "status", "op": "in", "value": ["todo", "doing"]}]}}
{"tool": "query",  "args": {"app_id": "kb",
  "filters": [{"field": "captured_at", "op": "gte", "value": "2026-04-14"}]}}
{"tool": "search", "args": {"app_id": "synap.memories", "entity_type": "event",
  "query_text": "decision", "rank": "time_decay"}}
```

Then synthesizes prose. This is the Synap pattern: agents compose small,
structured reads into dashboards on the fly.
