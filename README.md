# laminar-skill

Skills for Laminar: quick demos, codebase instrumentation, the SQL Query API, and the agent debugger.

## Install

```bash
npx skills add lmnr-ai/laminar-skills --skill laminar-quickstart-trace
npx skills add lmnr-ai/laminar-skills --skill laminar-instrument-codebase
npx skills add lmnr-ai/laminar-skills --skill laminar-migrate-observability
npx skills add lmnr-ai/laminar-skills --skill query-api
npx skills add lmnr-ai/laminar-skills --skill laminar-debug-trace
```

## Example prompts

- "Show me a Laminar example that produces a trace in under 2 minutes."
- "Instrument this codebase with Laminar and choose the most useful spans."
- "Use the Laminar Query API to find the slowest spans in the last 24 hours."
- "Debug my agent with the Laminar debugger: record a run, find the bad LLM call, and replay up to it."

## Contents

- `skills/laminar-quickstart-trace/SKILL.md`
- `skills/laminar-quickstart-trace/references/quickstart-node.md`
- `skills/laminar-quickstart-trace/references/quickstart-python.md`
- `skills/laminar-quickstart-trace/references/troubleshooting.md`
- `skills/laminar-instrument-codebase/SKILL.md`
- `skills/laminar-instrument-codebase/references/function-selection.md`
- `skills/laminar-instrument-codebase/references/ts-instrumentation.md`
- `skills/laminar-instrument-codebase/references/python-instrumentation.md`
- `skills/laminar-migrate-observability/SKILL.md`
- `skills/laminar-migrate-observability/references/migration-mapping.md`
- `skills/laminar-migrate-observability/references/otel-exporter.md`
- `skills/query-api/SKILL.md`
- `skills/query-api/references/laminar-query-api.md`
- `skills/laminar-debug-trace/SKILL.md`

## Notes

- You will need a Laminar project API key to send traces or execute queries.
