---
title: Recipes · Synap
---

# Recipes

Real end-to-end examples. Each recipe shows the exact MCP tool calls an
agent makes to stand up a new kind of data store, fill it, and query it.

These are written for agents to follow, not humans to copy-paste. If
you're driving Synap through Claude Desktop / Cursor / Zed, just paste
the natural-language prompts from each recipe into your chat — your agent
will translate them into the tool calls below.

## Available recipes

- **[Personal memory](/synap/recipes/personal-memory/)** — teach the agent
  facts about you and have it remember events over time.
- **[Knowledge base](/synap/recipes/knowledge-base/)** — build a
  searchable archive of notes, articles, and references with semantic
  retrieval.
- **[Task board](/synap/recipes/task-board/)** — a tasks app with
  statuses, due dates, tags, and "what should I work on next?" queries.
- **[Accounting](/synap/recipes/accounting/)** — track expenses, income,
  and categories; answer "how much did I spend on food last month?"
- **[Knowledge brain](/synap/recipes/knowledge-brain/)** — typed-graph
  knowledge store (concepts / people / companies / papers) with
  compiled-truth pages, append-only events, and graph traversal.
- **[LLM-maintained wiki](/synap/recipes/llm-wiki/)** — immutable
  sources + LLM-maintained summary / entity / concept / comparison
  pages, with a live index and cross-references.

## Common pattern

Every recipe follows the same three-step arc:

1. **Design** the schema with `init_app`. One call, idempotent.
2. **Populate** with `write` (optionally batched — N ops in one call are
   atomic).
3. **Query / search** with `query` (structured filters + projection) or
   `search` (semantic, keyword, or hybrid).

Schemas evolve without migrations — just start writing new fields. Old
rows will lack the attribute; query accordingly.
