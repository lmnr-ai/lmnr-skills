# SQL Query API

Run SELECT-only ClickHouse SQL over your project's data. Two ways in:

- **CLI** — `lmnr-cli sql query "<sql>"` (see [cli.md](cli.md)). Use the CLI is the most convenient way to query.
- **HTTP API** — `POST /v1/sql/query` (covered below). Only use the HTTP API for scripting or if you don't have CLI access.

## Endpoint and auth

- **Method/path:** `POST /v1/sql/query`
- **Base URL:** `https://api.lmnr.ai` by default. For self-hosted, use your Laminar base URL.
- **Headers:**
  - `Authorization: Bearer <project_api_key>`
  - `Content-Type: application/json`
  - `Accept: application/json`

## Request body

```json
{
  "query": "SELECT * FROM spans WHERE start_time > now() - INTERVAL 1 DAY",
  "parameters": {}
}
```

- `query` is required and must be SELECT-only.
- `parameters` is optional (send `{}` when unused). Placeholders use typed syntax `{name:Type}`:

```json
{
  "query": "SELECT * FROM spans WHERE trace_id = {trace_id:UUID} AND start_time > now() - INTERVAL 1 DAY",
  "parameters": { "trace_id": "01234567-89ab-4def-1234-426614174000" }
}
```

## Response

```json
{ "data": [ { "name": "span1", "output": "{\"result\": \"ok\"}" } ] }
```

`data` is an array of row objects. Non-2xx responses include an error body; SDKs raise.

## Tables

SELECT only. Queries are scoped to your project automatically — no tenant filter needed.

- **Traces and spans:** `spans`, `traces`, `trace_outputs` (extracted agent output messages), `logs`
- **Signals:** `signal_events`, `signal_events_all` (L0-inclusive), `signal_runs`, `clusters`, `event_clusters_all`
- **Datasets and evals:** `dataset_datapoints`, `dataset_datapoint_versions`, `evaluation_datapoints`, `labeling_queue_items`

Anything else is rejected (`Table 'x' is not allowed`), and a column that isn't on the table is rejected the same way, so don't guess: **`lmnr-cli sql schema` prints the live schema of the deployment you're pointed at**, which is the only reliable source when a project may be on Laminar Cloud or on an older self-hosted build. `project_id` is never queryable — the query is already scoped.

## Query guidance

- **Always filter by time** (`start_time`) for performance.
- **Avoid joins** — run multiple queries and combine in your app. The exception is signals: `traces` carries the trace's signal events and clusters as array columns, so read those instead of joining (see below).
- **Bucket by interval** with `toStartOfInterval` / `toStartOfDay` / `toStartOfHour`.
- **JSON is stored as strings** — use `simpleJSONExtract*` for fast access and `simpleJSONHas` to check keys.
- **Common JSON columns:** `spans` → `input`, `output`, `attributes`; `evaluation_datapoints` → `data`, `target`, `metadata`, `executor_output`, `scores`; `dataset_datapoints` → `data`, `target`, `metadata`.

## Signals and clusters

Signals are natural-language outcome/failure definitions; each match is a **signal event** on a trace, and similar events are grouped into named **clusters**. Both are readable two ways, and the direction you pick decides how much SQL you write.

**`traces` carries them as array columns.** This is the one to reach for when the row you want back is a trace, because trace cost, duration, status, and metadata are right there:

- `signal_events` — `Array(Tuple(event_id UUID, signal_id UUID, severity UInt8, payload String))`. `severity` is `0` INFO / `1` WARNING / `2` CRITICAL. Empty for traces no Signal fired on.
- `clusters` — `Array(Tuple(id UUID, signal_id UUID, name String, level UInt8, parent_id UUID, num_signal_events UInt32, created_at, updated_at))`. Already resolved to names. Contains each event's finest cluster (`level = 1`) **plus its ancestors as separate elements**, so filter on `level` when you don't want a trace counted once per level. Empty until events are clustered.

```sql
-- Which failure clusters cost the most this week?
SELECT c.name AS cluster, count() AS traces, round(sum(t.total_cost), 4) AS cost
FROM traces AS t
ARRAY JOIN t.clusters AS c
WHERE t.start_time > now() - INTERVAL 7 DAY AND c.level = 1
GROUP BY cluster ORDER BY cost DESC LIMIT 20

-- Critical signal events with the trace they came from.
SELECT t.id AS trace_id, t.end_time, e.signal_id, substring(e.payload, 1, 2000) AS payload
FROM traces AS t
ARRAY JOIN t.signal_events AS e
WHERE t.start_time > now() - INTERVAL 1 DAY AND e.severity = 2
ORDER BY t.end_time DESC LIMIT 50

-- Coverage: how many traces did a Signal fire on? (reads array length only)
SELECT countIf(notEmpty(signal_events)) AS with_signal, countIf(empty(signal_events)) AS without
FROM traces WHERE start_time > now() - INTERVAL 1 DAY
```

**Use `signal_events` when you want events back**, since `timestamp`, `name` (the Signal's name), `run_id`, and `signal_version` only exist there. Its cluster columns are `clusters` (`Array(UUID)`, leaf + ancestors), `leaf_clusters` (`Array(UUID)`, level 1 only — unnest this when each event must be counted once), and `cluster_details` (ids with names and levels resolved).

```sql
SELECT timestamp, name AS signal, severity, trace_id,
       arrayMap(c -> c.2, cluster_details) AS clusters,
       substring(payload, 1, 2000) AS payload
FROM signal_events
WHERE timestamp > now() - INTERVAL 1 DAY
ORDER BY timestamp DESC LIMIT 20
```

Use `clusters` on its own for questions about the hierarchy (sizes, parents, naming) with no trace or event involved, and `event_clusters_all` when you want (event, cluster) pairs pre-unnested.

Four things that will otherwise bite you:

- **`cluster_details` is an unnamed tuple**, so `c.name` does not resolve on it. Use positional access: `c.1` (id), `c.2` (name), `c.3` (level). The tuples on `traces` ARE named and take `c.name` / `c.level`.
- **`payload` is large, and reading any field of `traces.signal_events` reads the payload with it.** Select it as `substring(e.payload, 1, 2000)` or the row can be dropped for exceeding the result-size limit, and keep the `start_time` filter. `empty()` / `notEmpty()` read only the array length.
- **`num_signal_events` counts cluster assignments including descendants**, so it can exceed the number of distinct events you get by unnesting.
- **There is no `summary` column on `signal_events`.** Older builds exposed one; it is no longer part of the SQL surface, so read `payload` instead. There is also no `events` or `tags` table — tags are columns (`tags`, `trace_tags`) on `traces`. Run `lmnr-cli sql schema` if a query fails on an unknown column: the deployment may also predate the trace-side `signal_events` / `clusters` columns.

## Example queries

Cost by model:

```sql
SELECT model, sum(total_cost) AS total_cost, count(*) AS call_count
FROM spans
WHERE span_type = 'LLM' AND start_time > now() - INTERVAL 7 DAY
GROUP BY model ORDER BY total_cost DESC
```

Slowest operations (`duration` is a column, in seconds — don't subtract the timestamps):

```sql
SELECT name, avg(duration) AS avg_duration_s
FROM spans
WHERE start_time > now() - INTERVAL 1 DAY
GROUP BY name ORDER BY avg_duration_s DESC LIMIT 10
```

Error rate by span name:

```sql
SELECT name, countIf(status = 'error') AS errors, count(*) AS total,
       round(errors / total * 100, 2) AS error_rate
FROM spans
WHERE start_time > now() - INTERVAL 1 DAY
GROUP BY name HAVING total > 10 ORDER BY error_rate DESC
```

## curl example

```bash
curl -sS -X POST "${LMNR_BASE_URL:-https://api.lmnr.ai}/v1/sql/query" \
  -H "Authorization: Bearer $LMNR_PROJECT_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json" \
  -d '{"query":"SELECT name, input, output FROM spans WHERE start_time > now() - INTERVAL 1 DAY","parameters":{}}'
```
