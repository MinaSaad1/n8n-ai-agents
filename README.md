# n8n AI Agents

A curated catalog of production-ready n8n workflow templates for AI agents. Every template follows the same architectural principles, security framework, and output conventions — so learning one teaches you the rest.

![n8n](https://img.shields.io/badge/n8n-templates-EA4B71?logo=n8n) ![LangChain](https://img.shields.io/badge/LangChain-Tools_Agent-121212) ![Claude](https://img.shields.io/badge/Claude-Sonnet_4.6-D97757) ![License](https://img.shields.io/badge/License-MIT-yellow.svg)

---

## Templates

### 🟢 Supabase / Postgres Data Analyst

> Natural-language analytics chatbot over any Supabase or Postgres database. The agent introspects your schema, writes SQL, runs it safely, and returns answers with charts.

**Stack:** n8n · LangChain Tools Agent · Claude Sonnet 4.6 (OpenRouter) · Postgres Chat Memory · QuickChart

**Tools:** `list_schema` · `get_table_definition` · `run_sql` · `create_chart` · `search_schema_docs` · `think` · `export_to_sheets`

**[→ Open repo](https://github.com/MinaSaad1/n8n-supabase-data-analyst-agent)** · [Architecture](https://github.com/MinaSaad1/n8n-supabase-data-analyst-agent/blob/main/docs/ARCHITECTURE.md) · [Security](https://github.com/MinaSaad1/n8n-supabase-data-analyst-agent/blob/main/docs/SECURITY.md) · [Demo script](https://github.com/MinaSaad1/n8n-supabase-data-analyst-agent/blob/main/docs/DEMO_SCRIPT.md)

![Supabase canvas](assets/supabase-data-analyst.png)

---

### 🟡 Power BI Data Analyst

> Same idea, but over a Power BI semantic model. Agent writes DAX, hits the REST `executeQueries` API, and respects your existing measures instead of re-deriving them.

**Stack:** n8n · LangChain Tools Agent · Claude Sonnet 4.6 (OpenRouter) · Microsoft OAuth2 (Service Principal) · Power BI REST API · QuickChart

**Tools:** `list_tables_and_measures` · `get_table_columns` · `run_dax` · `create_chart` · `refresh_dataset` · `think`

**[→ Open repo](https://github.com/MinaSaad1/n8n-powerbi-data-analyst-agent)** · [Architecture](https://github.com/MinaSaad1/n8n-powerbi-data-analyst-agent/blob/main/docs/ARCHITECTURE.md) · [Security](https://github.com/MinaSaad1/n8n-powerbi-data-analyst-agent/blob/main/docs/SECURITY.md) · [Azure AD Setup](https://github.com/MinaSaad1/n8n-powerbi-data-analyst-agent/blob/main/docs/AZURE_AD_SETUP.md)

![Power BI canvas](assets/powerbi-data-analyst.png)

---

## Shared principles

All templates in this catalog follow the same patterns. Rather than duplicate these across every template's docs, the canonical versions live here:

- [**Architecture Principles**](docs/architecture-principles.md) — The Tools Agent shape, why sub-workflows for `run_*` and `create_chart`, primitive-based chart interfaces, memory handling, `think` tool rationale
- [**Security Framework**](docs/security-framework.md) — The layered defense model: auth → identity passthrough → data-source RLS → scoped roles → PII masking → query guards → audit/rate limit → output filter → cost caps → memory hygiene
- [**Output Conventions**](docs/output-conventions.md) — Strict markdown output format (bold headline → chart → pipe-table → query → observations), number formatting rules
- [**Testing Guide**](docs/testing-guide.md) — How to validate an imported template end-to-end, common failure modes, debugging via n8n's Executions panel
- [**Contributing**](docs/contributing.md) — For new templates: expected file structure, required docs, naming conventions, submission checklist

## Why these templates?

Most "AI agent + n8n" tutorials demo a single tool and call it done. These templates go further:

- **Schema-aware** — the agent introspects what exists before writing queries. No hallucinated table/column names.
- **Safe by default** — read-only access, row caps, explicit write authorization. Not production-secure out of the box (see the security framework), but the right starting point.
- **Visual by default** — 80%+ of responses include a chart. Users see trends, not just numbers.
- **Memory that works** — Postgres-backed conversation history keyed by sessionId.
- **Documented everywhere** — every workflow has sticky-note docs on the canvas. Every repo has architecture + security docs.

## Roadmap

Ideas for future templates — open issues if you want any prioritized, or [submit your own](docs/contributing.md):

- [ ] **Snowflake Data Analyst** — same pattern, Snowflake warehouse
- [ ] **BigQuery Data Analyst** — GCP-native
- [ ] **MongoDB / Document Store Analyst** — NoSQL variant with aggregation pipeline instead of SQL
- [ ] **HubSpot / Salesforce CRM Analyst** — natural-language CRM queries
- [ ] **Airtable Base Analyst** — for no-code teams
- [ ] **Metabase Question Writer** — agent writes Metabase questions from NL prompts
- [ ] **Shared evaluator/critic workflow** — validates agent responses against ground truth, for prompt-engineering iteration

## Author

Built by [Mina Saad](https://github.com/MinaSaad1). PRs and issues welcome on any template repo or this catalog.

## License

MIT — see [LICENSE](LICENSE). Each individual template repo carries its own MIT license.
