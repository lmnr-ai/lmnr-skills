# Laminar SQL Query API

## Endpoint and auth

- Method: POST
- Path: /v1/sql/query
- Base URL: default https://api.lmnr.ai (SDKs append :443). For self-hosted, use your Laminar base URL.
- Headers:
  - Authorization: Bearer <project_api_key>
  - Content-Type: application/json
  - Accept: application/json

## Request body

```json
{
  "query": "SELECT * FROM spans WHERE start_time > now() - INTERVAL 1 DAY",
  "parameters": {}
}
```

- query is required and must be SELECT-only.
- parameters is optional; send an empty object when not needed.
- Parameter placeholders use typed syntax: {name:Type}

Example with parameters:

```json
{
  "query": "SELECT * FROM spans WHERE trace_id = {trace_id:UUID} AND start_time > now() - INTERVAL 1 DAY",
  "parameters": {"trace_id": "01234567-89ab-4def-1234-426614174000"}
}
```

## Response

```json
{
  "data": [
    {"name": "span1", "output": "{\"result\": \"ok\"}"},
    {"name": "span2", "output": "{\"result\": \"fail\"}"}
  ]
}
```

- data is an array of rows represented as objects.
- Non-2xx responses include an error body; SDKs raise an error.

## Tables you can query

- spans
- traces
- events
- tags
- dataset_datapoints
- dataset_datapoint_versions
- evaluation_datapoints

Only SELECT queries are allowed.

## Query guidance

- Always filter by time (start_time) for performance.
- Avoid joins; run multiple queries and combine results in your app.
- Group by intervals using toStartOfInterval / toStartOfDay / toStartOfHour.
- JSON is stored as strings; use simpleJSONExtract* for fast access and simpleJSONHas to check keys.

## Common JSON columns

- spans: input, output, attributes
- evaluation_datapoints: data, target, metadata, executor_output, scores
- dataset_datapoints: data, target, metadata

## Example queries

Cost breakdown by model:

```sql
SELECT
    model,
    sum(total_cost) AS total_cost,
    count(*) AS call_count
FROM spans
WHERE span_type = 'LLM' AND start_time > now() - INTERVAL 7 DAY
GROUP BY model
ORDER BY total_cost DESC
```

Slowest operations:

```sql
SELECT name, avg(end_time - start_time) AS avg_duration_ms
FROM spans
WHERE start_time > now() - INTERVAL 1 DAY
GROUP BY name
ORDER BY avg_duration_ms DESC
LIMIT 10
```

Error rate by span type:

```sql
SELECT
    name,
    countIf(status = 'error') AS errors,
    count(*) AS total,
    round(errors / total * 100, 2) AS error_rate
FROM spans
WHERE start_time > now() - INTERVAL 1 DAY
GROUP BY name
HAVING total > 10
ORDER BY error_rate DESC
```

## Example curl

```bash
curl -sS -X POST "${LMNR_BASE_URL:-https://api.lmnr.ai}/v1/sql/query" \
  -H "Authorization: Bearer $LMNR_PROJECT_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"query":"SELECT name, input, output FROM spans WHERE start_time > now() - INTERVAL 1 DAY","parameters":{}}'
```
