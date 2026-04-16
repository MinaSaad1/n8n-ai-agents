# Testing Guide

How to validate an imported template works end-to-end, plus common failure modes.

## Before you test — checklist

- [ ] All workflows (main + subs) imported
- [ ] Credentials attached to every node that needs one (tool nodes in the main AND inside each sub-workflow)
- [ ] Sub-workflows **activated** (Tool Workflow nodes fail if the sub is inactive)
- [ ] Configuration values filled in (database IDs, workspace IDs, etc.)
- [ ] Main workflow activated
- [ ] The chat-memory table exists in your database

## Smoke test — 3 prompts

### 1. Schema discovery

> "What tables do you have?" (or "What tables and measures does this model have?")

**Expect:** agent calls the list tool, returns a formatted list. Execution should complete in <5 seconds.

**If it fails:** check the list tool's HTTP response in Executions. Most common cause: credential not attached, or (for Power BI) service principal not added to the workspace.

### 2. Simple aggregate

> "How many rows are in <table>?"

**Expect:** `get_table_definition` → `run_sql`/`run_dax` → single number in the response.

**If it stalls:** open Executions → find the hanging tool call → see its input/output. Usually the query itself is fine but credentials on the sub-workflow's internal node are missing.

### 3. Chart-generating question

> "Show me <metric> by <dimension> as a bar chart."

**Expect:** `get_table_definition` → `run_sql`/`run_dax` → `create_chart` → response with a QuickChart URL that renders a real chart when clicked.

**If the chart image is garbled/error:**
- Click the URL in a browser — QuickChart returns an error page if the spec is invalid
- Check the `create_chart` sub-workflow's last execution — inspect `chart_config` or the assembled Chart.js object
- If the agent sent malformed data shape, the Code node in the sub should throw a descriptive error

## Observability — use the Executions tab

n8n's Executions tab (top-right of the workflow editor) is your best debugging friend.

1. Run a test message.
2. Go to Executions → click the latest run.
3. You see every tool call in sequence with inputs and outputs expanded.
4. Red = errored node. Click it to see the error message and stack trace.

**Tip for demos:** split-screen the canvas and the chat. Nodes flash green as they fire in real time — it's a great visual of "the agent is thinking" without any custom UI work.

## Common failure modes

| Symptom | Root cause | Fix |
|---|---|---|
| "Agent stopped due to max iterations" | Usually memory poisoning (agent copying old tool-call pattern), or a tool erroring on every attempt | Click ↺ refresh in chat panel for a new sessionId. If it persists, check Executions for the recurring tool failure. |
| "Node does not have any credentials set" | Imported template, forgot to attach cred on one node | Open the failing node, pick the credential from the dropdown. Check sub-workflows too. |
| "Failed to parse tool arguments from chat model response" | LLM produced malformed JSON in a tool call (usually when the tool expects a big nested object) | Use primitive-typed tool inputs instead of `'json'` type. See [architecture-principles.md](architecture-principles.md) decision 3. |
| Tool returns 401/403 | Credential scope wrong, or role lacks DB/workspace access | Verify the cred in-isolation by running a manual REST call or opening the data source as that user |
| Chart URL returns a generic/error image | Chart.js config was malformed (wrong axis structure, missing data) | Inspect the chart sub-workflow's Build node output. If agent sent bad primitives, tighten the prompt. |
| Response formatting looks wrong (space-aligned table, no chart) | Agent is ignoring output rules | Tighten system prompt. Check memory isn't anchoring on old bad examples. |
| "Tenant or user not found" (Postgres connect) | Wrong username format (Supabase pooler expects `postgres.<project-ref>`) | Copy from Supabase's Session Pooler tab exactly |
| SSL handshake fails on Supabase | Pooler uses a cert Node doesn't trust | Toggle "Ignore SSL Issues" on the credential (still TLS, just skips cert chain verification) |
| IPv6 timeout on Supabase direct host | Supabase direct is IPv6-only; n8n host may be IPv4-only | Switch to the Session Pooler host |
| "Feature 'DatasetExecuteQueries' is not available" (Power BI) | Dataset is in a classic/personal workspace | Move to a modern workspace (v2) |

## Memory-related issues

If the agent behaves inconsistently across turns or copies old broken patterns:

**Most of the time:** click the ↺ refresh icon in the chat panel. New sessionId → empty memory load → fresh behavior. No database changes needed.

**If that doesn't help:** delete just your test session:
```sql
DELETE FROM public.n8n_chat_histories WHERE session_id = '<from chat header>';
```

**Never truncate** the whole table during development — other users/sessions don't deserve the nuke.

**During rapid tool schema iteration:** deactivate the memory node entirely. Stateless behavior, no cross-turn pattern anchoring. Re-enable when you're done.

## When to suspect the LLM vs. the workflow

| Symptom | Likely culprit |
|---|---|
| Agent writes bad SQL/DAX but the tool runs it | LLM — tighten prompt, give examples |
| Agent writes correct query, tool returns error | Workflow / data source — check the execution |
| Tool call succeeds but response formatting is wrong | LLM — tighten output rules in prompt |
| Tool loops more than expected | LLM — bad tool descriptions causing confusion, or memory anchoring |
| Tool returns wrong data | Workflow — query parameters, response parsing, or cap logic wrong |

## Load testing

Not covered by these templates out of the box. If you need to support >50 concurrent chats:
- Switch memory to Redis-backed (`memoryRedisChat` node)
- Multiply OpenRouter keys or switch to direct Anthropic (higher rate limits)
- For Power BI specifically: 120 req/min per Service Principal is the bottleneck — round-robin across multiple SPs or cache `list_tables_and_measures` in Postgres

## When demo-recording

- Open the workflow canvas next to the chat panel to show live node activation
- Start with an empty chat session (↺ before recording)
- Have 8-12 prompts pre-planned in escalating complexity
- Include one prompt that shows a safety refusal (e.g. "delete all orders")
- Include one that triggers the 200-row truncation (wider query than fits)
- Record in 1600-2000px width for crisp text on retina displays
