# Security Framework

The layered defense model used by every template. Source-agnostic: apply to any data backend.

## Threat model (what you're defending against)

1. **Unauthorized data access** — someone queries data not belonging to them
2. **PII exfiltration** — agent returns raw sensitive fields
3. **Prompt injection** — user tricks the agent into destructive or privileged actions
4. **Secret leakage** — API keys or service principal secrets exposed in logs, commits, or responses
5. **Rate abuse** — one user floods the agent, degrading service or burning LLM budget
6. **Cost runaway** — bad agent loops or unbounded queries consume unbounded tokens/compute
7. **Memory poisoning** — cross-user session pollution via predictable sessionIds

## Layers (priority order)

Implement top-to-bottom. Later layers harden; earlier ones are load-bearing.

### Layer 1 — Authenticate the chat trigger

**Default template state**: `public: true` on Chat Trigger → anyone with the URL can chat.

**Fix options:**
- Best: `public: false`, embed via `@n8n/chat` widget in your app, pass JWT in `metadata`
- Good: reverse proxy (Cloudflare Access, Authelia, nginx basic auth) in front of the webhook
- Never: rely on URL-as-secret ("nobody will guess the UUID")

### Layer 2 — Pass user identity into every query

Extract `user_id` / `tenant_id` from chat metadata via a Set node, then:

- Inject into system prompt (informational — the model knows the context)
- Change memory `sessionKey` to `tenant_id:user_id:sessionId` so memory can't cross-pollinate between users
- Pass through to Layer 3 via tool inputs

**Caveat:** system-prompt identity leaks easily. This layer is preparation, not enforcement.

### Layer 3 — Data-source-level access control (the only real boundary)

This is where identity is enforced. Implementation varies by source:

| Data source | Enforcement mechanism |
|---|---|
| **Postgres (Supabase, Neon, RDS)** | Row-Level Security policies + `SET LOCAL request.jwt.claims = …` before each query |
| **Power BI** | `impersonatedUserName` in `executeQueries` body + RLS roles defined on the dataset |
| **Snowflake** | Row Access Policies + `USE SECONDARY ROLES` |
| **BigQuery** | Authorized views + row-level access policies |
| **MongoDB / CosmosDB** | Field-level encryption + filter interceptors |

**Principle:** the *data source* filters rows. The agent can write any query it wants — only the user's own rows ever come back.

Without this layer, prompt-only safety is bypassable.

### Layer 4 — Scope the agent's service account narrowly

- Read-only role for queries (`n8n_readonly` for Postgres, Viewer/Member for Power BI workspace)
- `statement_timeout` / query timeout to prevent long-running queries
- `REVOKE` access to system tables, filesystem functions, external connectors
- Separate writable credential for the memory node (never write via the query credential)

### Layer 5 — PII masking via views / projections

Never give the agent raw access to PII tables. Create masking views:

```sql
CREATE VIEW public.users_safe AS
  SELECT id, country, plan,
         left(email,1) || '***@' || split_part(email,'@',2) AS email_masked
  FROM public.users;
GRANT SELECT ON public.users_safe TO n8n_readonly;
REVOKE SELECT ON public.users FROM n8n_readonly;
```

The agent can't even *see* real emails. Extend the pattern for phone numbers, addresses, tokens.

### Layer 6 — Query guard in the sub-workflow

The `run_*` sub-workflow's Code node inspects the query before executing:

```js
const q = String(input.query || '').toLowerCase();
const banned = /\b(insert|update|delete|truncate|drop|alter|create|grant|revoke|copy|dblink|pg_read_file)\b/;
if (banned.test(q)) throw new Error('Blocked: write/admin keyword detected.');
if (q.split(';').filter(s => s.trim()).length > 1) throw new Error('Blocked: multi-statement query.');
```

Defense in depth — the read-only role also blocks writes, but failing at the n8n layer saves an LLM retry.

### Layer 7 — Audit logging + rate limiting

Insert every user prompt into an audit table at the start of the main workflow:

```sql
INSERT INTO agent_audit (user_id, session_id, prompt, ts) VALUES ($1, $2, $3, now());
SELECT count(*) FROM agent_audit WHERE user_id = $1 AND ts > now() - interval '1 minute';
```

Add an If node: if recent count > 20 → short-circuit with a rate-limit message.

You now have:
- Full conversation audit trail (queryable, exportable)
- Per-user rate limiting
- Early signals for abuse investigation

### Layer 8 — Output filter

Code node *after* the agent scans the response for:
- Emails not matching the authenticated user's domain
- Raw JWT patterns (`eyJ...`)
- Base64 blobs > 200 chars
- References to system tables / auth schemas / vaults

If matched, return a generic error and log the incident.

### Layer 9 — Cost caps + agent bounds

- **OpenRouter monthly spend cap** on the API key — prevents budget runaway
- `maxIterations` lowered from 12 to 6-8 for production (12 is a dev buffer)
- `contextWindowLength` on memory: 10 messages (not 20). Fewer turns = fewer tokens per LLM call.
- Alert on OpenRouter's usage dashboard if daily spend exceeds a threshold

### Layer 10 — Memory hygiene

Without cleanup, `n8n_chat_histories` grows forever. Schedule:

```sql
DELETE FROM public.n8n_chat_histories
WHERE id IN (
  SELECT id FROM public.n8n_chat_histories
  WHERE session_id IN (
    SELECT session_id FROM public.n8n_chat_histories
    GROUP BY session_id
    HAVING max(id) < (SELECT max(id) - 10000 FROM public.n8n_chat_histories)
  )
);
```

Run nightly via `pg_cron` or an n8n Schedule Trigger.

## What does NOT work (common misconceptions)

- ❌ **"The system prompt tells the model to refuse writes, so we're safe"** — prompt injection defeats this trivially
- ❌ **"The chat URL is unguessable"** — assume it's public the moment it's generated
- ❌ **"Session IDs isolate users"** — sessionIds are UUIDs picked by the *client*; nothing stops one user from replaying another's sessionId
- ❌ **"RLS on tables protects us"** — RLS only applies when the query runs under the user's identity (Layer 3). A service account bypasses it.
- ❌ **"Regex query guards prevent all SQL injection"** — they help, but they're defense-in-depth, not the boundary. The boundary is Layer 3.

## Priority if you implement only some

1. **Layer 3** (data-source-level access control) — without this, nothing else matters
2. **Layer 1** (authenticate the chat URL) — close the open door
3. **Layer 4** (scoped service account) — massive blast-radius reduction
4. **Layer 9** (cost caps) — financial protection
5. **Layers 5-8, 10** — harden further

## Reporting security issues

Open a private security advisory on the affected template's repo. 5-day response target.
