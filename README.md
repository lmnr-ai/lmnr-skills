# laminar-skill

A single, unified skill for [Laminar](https://docs.laminar.sh): instrument code with tracing, run the Laminar CLI, query trace data with SQL, and migrate from other observability tools.

## Install

```bash
npx skills add lmnr-ai/laminar-skills --skill laminar
```

## Example prompts

- "Set up Laminar in this repo."
- "Instrument this codebase with Laminar and choose the most useful spans."
- "Show me a Laminar example that produces a trace in under 2 minutes."
- "Use the Laminar CLI to query the slowest spans in the last 24 hours."
- "Migrate our Langfuse tracing to Laminar."

## Contents

The skill routes from a single `SKILL.md` to focused, one-level-deep reference files:

- `skills/laminar/SKILL.md` — mental model, workflow, and task router
- `skills/laminar/references/instrumentation-typescript.md` — TypeScript/JS/Next.js setup, spans, context, privacy
- `skills/laminar/references/instrumentation-python.md` — Python setup, spans, context, privacy
- `skills/laminar/references/quickstart.md` — minimal demo trace (Node + Python) and troubleshooting
- `skills/laminar/references/cli.md` — `lmnr-cli` and the SDK-bundled `lmnr` CLI (sql, datasets, dev, eval)
- `skills/laminar/references/sql-query-api.md` — SQL Query API and ClickHouse query patterns
- `skills/laminar/references/migration.md` — migrate from Langfuse, LangSmith, Helicone, or OpenTelemetry

## Notes

- You will need a Laminar project API key (dashboard → **Settings → Project API Keys**) to send traces or execute queries.
