---
title: Concepts · Synap
---

# Concepts

The mental model behind Synap. Read this once — everything else maps back to
these four ideas.

## Apps

An **app** is a namespace with a schema. Examples: `tasks`, `accounting`,
`reading`. Each app is a container for entities of one or more types.

Apps are created on demand by the agent calling `init_app`. No `CREATE
TABLE`, no migration file — the agent designs the shape in context and
writes it immediately.

Two apps are built-in and reserved (`synap.*` namespace):

- `synap.memories` — long-term memory about the user
- `synap.sessions` — agent conversation history

User-facing apps must not start with `synap.`.

## Entities and the EAV model

Inside an app, data lives as **entities** (rows) with **attributes**
(fields). Under the hood it's Entity-Attribute-Value, so adding a field
later is zero-cost — old rows just don't have the attribute.

Schemas define:
- Field name and type (`text`, `number`, `date`, `enum`, `bool`)
- `vectorized: true` to enable semantic search on that field
- `entity_type` for apps that hold multiple shapes (e.g., `task` + `note`
  in one `workspace` app)

## Memory: profile vs. event

`synap.memories` has two shapes, and picking the right one matters:

**Profile attribute** (`type: "attr"`) — enduring state about a subject.
`preferred_language`, `city`, `allergies`. One row per (subject, attribute).
Updates overwrite. Use `op: "append"` for list-valued attributes.

**Event** (`type: "event"`) — a timestamped observation. "User shipped
v0.1.0 today", "user mentioned feeling overwhelmed". Append-only, with
`when` and `description`. The description is embedded — semantic search
finds events by meaning.

Agents write via `remember` / `forget` (writes to `synap.*` apps are
blocked — these tools enforce the shape). They read via normal `query` /
`search`.

Pre-turn context: the daemon injects `[memory: profile]` and `[memory:
events]` blocks into every LLM turn automatically. Agents don't need to
`query` memory to see it.

## Hybrid search and ranking

`search` combines keyword (FTS5) and semantic (sqlite-vec embeddings) with
three ranking modes:

- **`relevance`** (default) — pure similarity score. Best for "find me
  anything about X".
- **`recency`** — newest first, ignoring similarity. Best for "latest
  notes on Y".
- **`time_decay`** — blended. Similarity weighted by a time half-life.
  Best for evolving context (events, journals) where both freshness *and*
  meaning matter.

Long text (>1024 tokens) is auto-chunked with 128-token overlap. Each
chunk is embedded separately; search deduplicates by entity, keeping the
best-matching chunk's score.

## Lossless context compaction (LCM)

Long conversations are compacted between turns. A contiguous range of raw
turns is summarized into a `<compressed_segment turn_ids="...">` block;
the originals stay in `synap.sessions` and are recoverable with the
`expand` tool.

Two thresholds control it (configurable in `agent.toml`):
- `τ_soft` (default 32K chars) — spawn async compaction between turns.
- `τ_hard` (default 64K chars) — block-compact before the next call.

This is automatic. The agent just needs to know: **if context looks
compressed, `expand` restores the originals**.

## One-sentence recap

> An agent talks to Synap through MCP to create structured apps, remember
> things about the user, and search across everything — all local, all
> backed by one SQLite file.

Next: try a [recipe](/synap/recipes/) or flip through the [reference](/synap/reference/).
