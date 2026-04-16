# Contributing

For building new templates that fit this catalog.

## When to add a template here vs. build your own

Add yours to this catalog if:
- It's an **n8n AI agent** (Tools Agent pattern + LangChain)
- It follows the architectural principles in [architecture-principles.md](architecture-principles.md)
- It uses the output conventions in [output-conventions.md](output-conventions.md)
- It has security docs covering the layers in [security-framework.md](security-framework.md) — even if not all are implemented

If your template does something entirely different (batch workflow, no chat, single-purpose automation), better to host standalone.

## Expected repository structure

```
your-template-name/
├── README.md              <-- quickstart + hero image + architecture diagram
├── LICENSE                <-- MIT recommended
├── .gitignore             <-- block .env, credentials.json, *.pem, *.key
├── workflows/
│   ├── 01-main-<name>.json
│   ├── 02-sub-<purpose>.json
│   └── 03-sub-<purpose>.json
├── sql/                   <-- if your template needs DB setup
│   ├── 01-setup.sql
│   └── 02-demo-data.sql   <-- optional
└── docs/
    ├── ARCHITECTURE.md    <-- template-specific deep dive
    ├── SECURITY.md        <-- template-specific layers (link to catalog framework)
    └── <data-source>_SETUP.md <-- e.g. AZURE_AD_SETUP.md, SNOWFLAKE_SETUP.md
```

## Required files

### `README.md` — must include

- Hero image (canvas screenshot at top)
- One-paragraph "what it does"
- Architecture diagram (ASCII is fine)
- Requirements list
- Quickstart (import → credentials → config → test)
- Troubleshooting table for common failure modes
- Link to catalog's shared docs for cross-cutting concerns

### `workflows/*.json` — must have

- **Credentials stripped** — no `credentials` blocks on nodes
- **Server-side fields stripped** — no `id`, `versionId`, `active`, `meta`, `pinData`, `staticData`, `tags`, `createdAt`, `updatedAt`, `versionCounter`
- **Sticky note README** on the main workflow canvas explaining setup
- **Numeric prefix** on filenames so import order is obvious (`01-`, `02-`, ...)

### `docs/ARCHITECTURE.md` — must cover

- Tool-by-tool walkthrough
- Auth flow diagram if non-trivial
- Design decisions specific to this data source
- Performance notes (typical latencies, rate limits)
- What to reuse from the catalog's shared architecture principles

### `docs/SECURITY.md` — must address

- Each of the 10 layers from [security-framework.md](security-framework.md) — which ones are default-on, which need operator setup
- Data-source-specific enforcement (RLS, row-level security, PostgREST, XMLA, etc.)
- Secret rotation procedure

## Naming conventions

- Repo name: `n8n-<source>-<purpose>-agent` (e.g. `n8n-snowflake-data-analyst-agent`)
- Main workflow name (in n8n): `<Source> <Purpose> Agent`
- Sub-workflow names (in n8n): `[<Source> Sub] <Purpose>`
- Tool names in agent (as registered via `workflowInputs.schema[].id`): lowercase snake_case verbs — `run_sql`, `create_chart`, `list_tables`, not `RunSql` or `runSql`

## Pre-submission checklist

Before opening a PR against this catalog's README (to add your template card):

- [ ] Repo is public and has a LICENSE
- [ ] README has a hero image (actual screenshot, not placeholder)
- [ ] All workflow JSON files import cleanly into a fresh n8n instance
- [ ] Credentials stripped from all workflow JSON
- [ ] System prompt follows the output conventions
- [ ] Chart tool uses primitive interface (not full Chart.js JSON)
- [ ] Row cap implemented in a sub-workflow, not just prompt
- [ ] Memory uses Postgres Chat Memory keyed by sessionId
- [ ] GitHub secret scanning + push protection enabled on the repo
- [ ] At least 3 example prompts in README's Quickstart
- [ ] Troubleshooting section with common failure modes

## How to submit

1. Build your template in its own public repo following the above.
2. Open a PR on this catalog repo that adds a section to the main README using the same card format as existing entries:

   ```markdown
   ### 🔵 <Emoji> <Template Name>

   > One-line description of what it does.

   **Stack:** ...
   **Tools:** ...

   **[→ Open repo](https://github.com/<you>/<repo>)** · [Architecture](link) · [Security](link)

   ![canvas](assets/<your-template>.png)

   ---
   ```

3. Add a canvas screenshot to `assets/<your-template>.png` (1600-2000px wide, PNG).
4. If you want your template reviewed for architectural consistency: tag `@MinaSaad1` in the PR.

## Style

- **Ship, don't explain.** The repo is the proof. Tight READMEs beat verbose ones.
- **Strip branding.** Templates should work for any user — no hardcoded usernames, URLs, or specific datasets.
- **Default to safe.** Read-only credentials by default. Writes are opt-in via explicit tool descriptions.
- **Document failure modes.** Your troubleshooting table is the most-read section of your README.

## License

By contributing, you agree your contributions will be licensed under MIT (or whatever license your repo declares). The catalog itself is MIT.
