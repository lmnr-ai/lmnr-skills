# laminar-skill

A single, unified skill for [Laminar](https://laminar.sh/docs): instrument code with tracing, run the Laminar CLI, query trace data with SQL, debug agents with record/replay, and migrate from other observability tools.

## Install as a plugin

The `laminar` plugin bundles this skill with the hosted [Laminar MCP server](https://laminar.sh/docs/platform/mcp) (`https://api.lmnr.ai/v1/mcp`), so your agent can query traces with SQL and manage signals. On first use it opens a browser to sign in to Laminar; there is no API key to paste.

**Cursor**: add `lmnr-ai/lmnr-skills` from **Cursor Settings → Plugins**.

## Install the skill only

```bash
npx skills add lmnr-ai/lmnr-skills --skill laminar
```

## Example prompts

- "Set up Laminar in this repo."
- "Instrument this codebase with Laminar and choose the most useful spans."
- "Show me a Laminar example that produces a trace in under 2 minutes."
- "Use the Laminar CLI to query the slowest spans in the last 24 hours."
- "Migrate our Langfuse tracing to Laminar."
- "Debug my agent with the Laminar debugger: record a run, find the bad LLM call, and replay up to it."

## Contents

The skill routes from a single `SKILL.md` to focused, one-level-deep reference files:

- `skills/laminar/SKILL.md` — mental model, workflow, and task router
- `skills/laminar/references/instrumentation-typescript.md` — TypeScript/JS/Next.js setup, spans, context, privacy
- `skills/laminar/references/instrumentation-python.md` — Python setup, spans, context, privacy
- `skills/laminar/references/quickstart.md` — minimal demo trace (Node + Python) and troubleshooting
- `skills/laminar/references/cli.md` — `lmnr-cli` and the SDK-bundled `lmnr` CLI (sql, datasets, dev, eval)
- `skills/laminar/references/sql-query-api.md` — SQL Query API and ClickHouse query patterns
- `skills/laminar/references/migration.md` — migrate from Langfuse, LangSmith, Helicone, or OpenTelemetry
- `skills/laminar/references/debugging.md` — agent debugger: record under `LMNR_DEBUG`, inspect with SQL, replay cached LLM calls, annotate sessions

## Notes

- To send traces from your code, or to query with the Laminar CLI, you need a project API key (dashboard → **Settings → Project API Keys**). The plugin's MCP server signs you in through the browser instead.

## Support and privacy

- Support: founders@lmnr.ai
- [Privacy Policy](https://laminar.sh/policies/privacy) · [Terms of Service](https://laminar.sh/policies/terms)
