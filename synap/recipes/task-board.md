---
title: Task Board Recipe · Synap
---

# Recipe: Task board

A tasks app with statuses, due dates, tags, and semantic search over
notes.

## 1. Design

> Make a synap app "tasks". Fields: title (required), status (enum:
> todo/doing/done/cancelled), priority (enum: low/med/high),
> due (date), tags (multi), notes (text, vectorized), created_at (date).

```json
{"tool": "init_app", "args": {
  "name": "tasks",
  "description": "Personal task board",
  "fields": {
    "title":      {"type": "text", "required": true},
    "status":     {"type": "enum", "values": ["todo", "doing", "done", "cancelled"], "default": "todo"},
    "priority":   {"type": "enum", "values": ["low", "med", "high"], "default": "med"},
    "due":        {"type": "date"},
    "tags":       {"type": "text", "multi": true},
    "notes":      {"type": "text", "vectorized": true},
    "created_at": {"type": "date"}
  }
}}
```

## 2. Add tasks

> Add tasks:
> - "Ship Synap 0.1.2" due 2026-05-01, high priority
> - "Write release notes" due 2026-04-29, tagged docs
> - "Code-sign macOS binary" due 2026-05-10, notes: "need Apple Developer ID ($99/yr), notarize via notarytool, embed timestamp"

```json
{"tool": "write", "args": {
  "app_id": "tasks",
  "ops": [
    {"op": "create", "entity": {
      "title": "Ship Synap 0.1.2", "priority": "high",
      "due": "2026-05-01", "created_at": "2026-04-19"
    }},
    {"op": "create", "entity": {
      "title": "Write release notes", "tags": ["docs"],
      "due": "2026-04-29", "created_at": "2026-04-19"
    }},
    {"op": "create", "entity": {
      "title": "Code-sign macOS binary",
      "due": "2026-05-10", "created_at": "2026-04-19",
      "notes": "need Apple Developer ID ($99/yr), notarize via notarytool, embed timestamp"
    }}
  ]
}}
```

One call, atomic.

## 3. Everyday queries

> What's on my plate today?

```json
{"tool": "query", "args": {
  "app_id": "tasks",
  "filters": [
    {"field": "status", "op": "in",  "value": ["todo", "doing"]},
    {"field": "due",    "op": "lte", "value": "2026-04-20"}
  ],
  "sort": [{"field": "priority", "dir": "desc"}, {"field": "due", "dir": "asc"}]
}}
```

> Show me everything I finished this month.

```json
{"tool": "query", "args": {
  "app_id": "tasks",
  "filters": [
    {"field": "status", "op": "eq", "value": "done"},
    {"field": "due",    "op": "gte", "value": "2026-04-01"}
  ],
  "sort": [{"field": "due", "dir": "desc"}]
}}
```

## 4. Update state

> Mark "Write release notes" as done.

```json
{"tool": "write", "args": {
  "app_id": "tasks",
  "ops": [{
    "op": "update",
    "entity_id": "<id-from-query>",
    "entity": {"status": "done"}
  }]
}}
```

EAV update — only the changed attribute is written, not the whole row.

## 5. Semantic recall on notes

> Do I have any tasks about code signing?

```json
{"tool": "search", "args": {
  "app_id": "tasks",
  "query": "code signing certificate notarization",
  "rank":  "relevance"
}}
```

Finds the macOS binary task by `notes` content even though the title
doesn't mention "certificate" or "notarization".

## 6. Cross-app status report

> Give me a status snapshot: open tasks, notes captured this week, recent
> decisions I've remembered.

An agent can orchestrate three calls:

```json
{"tool": "query",  "args": {"app_id": "tasks", "filters": [{"field": "status", "op": "in", "value": ["todo", "doing"]}]}}
{"tool": "query",  "args": {"app_id": "kb",    "filters": [{"field": "captured_at", "op": "gte", "value": "2026-04-14"}]}}
{"tool": "search", "args": {"app_id": "synap.memories", "query": "decision", "rank": "time_decay"}}
```

Then synthesizes a one-page report. This is the Synap pattern: agents
compose small, structured reads into dashboards on the fly.
