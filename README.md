# laminar-skill

Skills for Laminar: quick demos, codebase instrumentation, and the SQL Query API.

## Install

```bash
npx skills add lmnr-ai/laminar-skills --skill laminar-quickstart-trace
npx skills add lmnr-ai/laminar-skills --skill laminar-instrument-codebase
npx skills add lmnr-ai/laminar-skills --skill query-api
```

## Example prompts

- "Show me a Laminar example that produces a trace in under 2 minutes."
- "Instrument this codebase with Laminar and choose the most useful spans."
- "Use the Laminar Query API to find the slowest spans in the last 24 hours."

## Contents

- `skills/laminar-quickstart-trace/SKILL.md`
- `skills/laminar-quickstart-trace/references/quickstart-node.md`
- `skills/laminar-quickstart-trace/references/quickstart-python.md`
- `skills/laminar-quickstart-trace/references/troubleshooting.md`
- `skills/laminar-instrument-codebase/SKILL.md`
- `skills/laminar-instrument-codebase/references/function-selection.md`
- `skills/laminar-instrument-codebase/references/ts-instrumentation.md`
- `skills/laminar-instrument-codebase/references/python-instrumentation.md`
- `skills/query-api/SKILL.md`
- `skills/query-api/references/laminar-query-api.md`

## Notes

- You will need a Laminar project API key to send traces or execute queries.
