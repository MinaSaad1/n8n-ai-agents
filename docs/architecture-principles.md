# Architecture Principles

The patterns every template in this catalog follows. If you're building your own, these are the decisions already made for you.

## The shape

```
Chat Trigger  →  Tools Agent (LangChain)  →  final answer
                    │                        (streams to chat)
                    ├─ LLM (OpenRouter)
                    ├─ Postgres Chat Memory
                    └─ N tools (HTTP, sub-workflows, Code)
```

Every template is this shape. Swap the data source; the skeleton stays.

## Decision 1 — Tools Agent, not SQL Agent or Conversational Agent

n8n has a built-in SQL Agent that handles schema discovery and query execution automatically. It's opaque: you can't add custom tools (chart generation, exports, `think`), can't tighten the prompt, and can't guard writes.

**Tools Agent** is more code but infinitely more extensible. Every template uses it with `agent: toolsAgent`, `maxIterations: 12`.

## Decision 2 — Sub-workflows wrap every complex tool

`run_sql`, `run_dax`, `create_chart` — each is a separate sub-workflow, not an inline node. Why:

- **Post-processing**: row capping, response flattening, error repair happen in a Code node the agent can't skip.
- **Reuse**: other workflows (like a scheduled briefing) can call the same sub-workflow.
- **Testing**: each sub runs independently from n8n's UI.
- **Future hooks**: audit logging, query rewriting, RLS impersonation fit here without touching the agent.

Simple tools (schema introspection queries, `think` scratchpad) stay as single HTTP/Code tools. The rule: **if the tool needs post-processing, make it a sub-workflow.**

## Decision 3 — Chart interface takes primitives, not full Chart.js

First version of `create_chart` took a complete Chart.js v4 JSON object. Worked ~90-95% of the time. Under memory pressure (>5k token context), Sonnet produced JSON with nested-brace syntax errors roughly 5-10% of calls → LangChain parser rejected them → agent looped until max iterations.

**Solution**: the tool takes flat primitives (`chart_type`, `title`, `labels[]`, `series[{name, values}]`, `x_label`, `y_label`). The sub-workflow's Code node assembles the Chart.js config from these primitives. Zero parse failures since.

**Principle**: LLMs reliably emit small flat fields; they unreliably emit large nested JSON. Design tool interfaces accordingly.

## Decision 4 — Row cap in the sub-workflow, not the prompt

The prompt *tells* the agent to `LIMIT 500` (or `TOPN(500, …)` for DAX). But prompts can't be trusted — the model might forget, or the user might ask "show me ALL rows."

**Hard cap lives in code**: the `run_sql` / `run_dax` sub-workflow's Code node slices to 200 rows max and adds `truncated: true` + `total_rows` + a human-readable note. The agent sees this and is instructed to tell the user when results are truncated.

This is the **single biggest prod reliability win**. Unbounded queries blow up the LLM context in seconds.

## Decision 5 — `think` as a first-class tool

The `think` tool is a Code node that just returns the agent's thought string back. It does no real work.

But: **forcing the model to verbalize reasoning before complex queries or after errors measurably improves tool-call accuracy.** The system prompt instructs the agent to call `think` before any complex query and after any tool error — it's a prompt-engineering device disguised as a tool.

## Decision 6 — Postgres Chat Memory, not Redis / in-memory

- Survives n8n restarts
- Same DB you're already connected to → zero extra infra
- Queryable: you can audit conversations, export for fine-tuning, spot agent misbehavior patterns
- `sessionId` from Chat Trigger is the isolation key

Trade-off: memory loads from DB per turn. For high concurrency (>100 simultaneous sessions), swap for Redis.

## Decision 7 — Memory pattern anchoring is real; plan for it

When you change a tool's signature mid-development, the agent sees old tool calls in memory and copies the old shape → every call fails → loop.

**Fixes** (in order of what you should reach for):

1. Click the ↺ refresh icon in the chat panel → new sessionId → fresh memory. Does NOT require deleting rows.
2. Delete just your test session: `DELETE FROM n8n_chat_histories WHERE session_id = '...'`.
3. Deactivate the memory node entirely during rapid tool iteration.
4. **Never**: `TRUNCATE n8n_chat_histories`. Don't nuke other users' conversations for your dev issue.

In production this doesn't happen because tool schemas are stable.

## Decision 8 — Schema discovery is a tool, not a fixed prompt

Injecting the entire schema into the system prompt (what some tutorials do) breaks at scale: 50+ tables = massive context on every turn, and the prompt goes stale as the schema evolves.

**Instead**: `list_schema` / `list_tables_and_measures` is a tool the agent calls *once per conversation*. Memory carries the result forward. For specific-table details, `get_table_definition` / `get_table_columns` fetches just one table's columns.

**Principle**: treat metadata as data. Fetch it lazily, cache it via memory.

## Decision 9 — Output formatting lives in the system prompt, not post-processing

Every template's system prompt enforces the same 5-section output:

1. Bold headline with key numbers
2. Chart (markdown image)
3. Pipe-delimited markdown table (`| ---: |` for right-aligned numerics)
4. Fenced code block with the exact query used
5. 2-4 observation bullets

Post-processing the agent's markdown is fragile (regex on LLM output breaks). **Prompt-level constraints are more robust**, especially when the chat UI renders GFM markdown natively.

See [Output Conventions](output-conventions.md) for the exact rules.

## Decision 10 — Credentials stripped from shipped JSON

Every template's workflow JSON in the GitHub repo has `credentials` blocks removed. Users attach their own on import. Trade-off: a "cred not found" warning on import that requires one click per node to resolve. Worth it — zero risk of leaking your instance's credential IDs.

## Anti-patterns (what NOT to do)

- ❌ **Giant tool call with nested JSON** — use primitives
- ❌ **Prompt-only safety** — always have a code-level cap or guard
- ❌ **Embedding full schema in prompt** — use a tool
- ❌ **Truncating shared memory on dev issues** — refresh session instead
- ❌ **LLM-layering for reliability** — each LLM hop adds failure modes. We tried Haiku for chart-spec generation; it hallucinated. Direct is simpler.
- ❌ **"Streaming" via polling** — if streaming matters, use n8n's streaming mode, don't fake it
- ❌ **Re-exporting workflow JSON with credentials attached** — strip before commit
