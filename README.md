# laminar-skill

Skill for working with Laminar's SQL Query API (POST /v1/sql/query).

## Install

```bash
npx skills add lmnr-ai/laminar-skills --skill query-api
```

## Example prompts

- "Use the Laminar Query API to find the slowest spans in the last 24 hours."
- "Write a SELECT query for spans by trace_id with a 1 day time filter."
- "How do I pass parameters to /v1/sql/query and parse the response?"

## Contents

- `skills/query-api/SKILL.md`
- `skills/query-api/references/laminar-query-api.md`

## Notes

- You will need a Laminar project API key to execute queries.
