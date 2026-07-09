# Creating eval datasets

Build a Laminar dataset of datapoints for evaluations — from production traces, local files, or hand-written examples — and wire it into `evaluate()`. A good dataset is the difference between an eval that measures something and an eval that measures noise: frozen, provenance-tagged, and shaped so the executor and evaluators can consume it directly.

For the full record → eval → verify loop inside a debug session, see [eval-loop.md](eval-loop.md); this file is the standalone how-to for the dataset itself.

## Datapoint format

Every datapoint has three JSON objects, each with arbitrary keys:

- **`data`** — the executor's input. Shape it to match what the executor function receives (e.g. `{ "messages": [...] }` or `{ "country": "France" }`).
- **`target`** — reference data passed to evaluators alongside the executor output. Optional: production traces have no gold label, so regression datasets usually omit it and the evaluator checks a *property* (did the failure recur?), not exact match.
- **`metadata`** — provenance and slicing keys (`source_trace_id`, `cluster_id`, `customer_tier`, ...). Never scoring inputs.

```json
{
  "data": { "messages": [{ "role": "user", "content": "Refund my order" }] },
  "target": { "expected_action": "escalate_to_human" },
  "metadata": { "source_trace_id": "0197f2..." }
}
```

On push, each datapoint gets a UUIDv7 `id`. Ids drive versioning — **never edit or invent `id` values in local files**. Editing a datapoint and re-pushing with the same id creates a new version; deleting a datapoint locally does NOT delete it in Laminar (push only adds).

## Design rules

- **Match `data` keys to the executor signature.** `evaluate()` passes the whole `data` object to the executor; a mismatch fails at run time, not push time.
- **Keep one dataset per failure mode / capability**, named for what it tests (`refund-escalation-failures`, `capitals-of-the-world`) — not `test-data-2`.
- **Freeze before iterating.** Build the dataset once, then change the agent one thing at a time against the same datapoints. Growing the dataset mid-iteration makes run-over-run diffs meaningless.
- **Stamp provenance into `metadata`** so failing rows can be traced back to the production trace or source file they came from.
- **Size**: 20–100 datapoints is usually enough to see a score move; prefer breadth of failure categories over volume.

## Choose a source

| Datapoints come from... | Use |
|-------------------------|-----|
| Production traces (real failures, regression sets) | SQL query → JSONL → `lmnr-cli dataset create` (below) |
| Existing files (CSV / JSON / JSONL) | `lmnr-cli dataset create` / `push` (below) |
| Code (synthesized or transformed programmatically) | SDK `client.datasets.push` (below) |
| A single interesting trace in the UI | **Add to dataset** on the span (fills `data` from input, `target` from output) |
| An eval run's failing rows | SQL editor query on `evaluation_datapoints` → **Export to Dataset** in the UI |

## From production traces (SQL → JSONL → create)

Requires `lmnr-cli login` and a linked project (see [cli.md](cli.md)). Pull replay inputs for the traces you care about — filter by status, content, tags, or signal clusters. Always filter by `start_time`.

```bash
# Find candidate traces (adapt the WHERE clause: status = 'error', ILIKE on output, has(trace_tags, ...))
npx lmnr-cli sql query "
  SELECT id, root_span_input FROM traces
  WHERE status = 'error'
    AND start_time > now() - INTERVAL 30 DAY
  LIMIT 50" --json \
| jq -c '.[] | {
    data: (.root_span_input | fromjson? // .root_span_input),
    metadata: { source_trace_id: .id }
  }' > data.jsonl

npx lmnr-cli dataset create refund-escalation-failures data.jsonl -o refund-escalation-failures.jsonl
```

`root_span_input` may be a raw string or JSON — the `fromjson? // .` fallback handles both. To source from a specific span instead of the trace root, query `spans` (`SELECT input FROM spans WHERE name = '...' AND start_time > ...`).

Inspect a couple of produced datapoints before creating the dataset: the executor must be able to run on `data` as-is.

## From files

`lmnr-cli dataset` accepts `.jsonl` (one datapoint per line), `.json` (one array), and `.csv` (header row required; cell values that look like JSON are parsed). **Every row must have a `data` field** — `target`, `metadata`, and `id` are optional; other top-level keys are silently dropped on this path. (The UI file upload is more lenient: there, unrecognized keys fold into `data`.)

```bash
lmnr-cli dataset create my-dataset data.jsonl -o my-dataset.jsonl  # create + local copy WITH ids
lmnr-cli dataset push data.jsonl -n my-dataset                     # add datapoints to existing dataset
lmnr-cli dataset pull out.jsonl -n my-dataset                      # download (overwrites out.jsonl)
lmnr-cli dataset list --json
```

`push`/`pull` take `-n <name>` or `--id <id>`; `create`/`push`/`pull` accept `--batch-size` (default 100) and `-r`/`--recursive` for directories. `-o` is required on `create`; the copy it writes is the file to keep editing locally — it carries the server-assigned ids, so re-pushing it versions datapoints instead of duplicating them.

No `lmnr-cli login` available (headless, no browser)? The SDK-bundled CLI does the same job with just `LMNR_PROJECT_API_KEY`: `npx lmnr datasets create <name> <file>` (TS) or `lmnr datasets create <name> <file>` (Python).

## From code (SDK)

Use the SDK client when datapoints are synthesized or transformed programmatically. `createDataset` / `create_dataset: true` creates the dataset if it doesn't exist (name required).

**TypeScript:**

```typescript
import { LaminarClient } from '@lmnr-ai/lmnr';

const client = new LaminarClient(); // reads LMNR_PROJECT_API_KEY from env
await client.datasets.push({
  name: 'refund-escalation-failures',
  createDataset: true,
  points: cases.map((c) => ({
    data: { messages: c.messages },
    target: { expected_action: c.expected },
    metadata: { source: 'synthetic' },
  })),
});
```

**Python:**

```python
from lmnr import LaminarClient
from lmnr.sdk.types import Datapoint

client = LaminarClient()  # reads LMNR_PROJECT_API_KEY from env
client.datasets.push(
    points=[
        Datapoint(
            data={"messages": case["messages"]},
            target={"expected_action": case["expected"]},
            metadata={"source": "synthetic"},
        )
        for case in cases
    ],
    name="refund-escalation-failures",
    create_dataset=True,
)
```

Both push in batches of 100 by default. Read back with `client.datasets.pull({ name })` / `client.datasets.pull(name=...)` (paginated via `offset` / `limit`).

## Wire it into `evaluate()`

Pass the dataset by name; each row's `data` / `target` / `metadata` become the datapoint the executor and evaluators receive.

```typescript
import { evaluate, LaminarDataset } from '@lmnr-ai/lmnr';

evaluate({
  data: new LaminarDataset('refund-escalation-failures'),
  executor: async (data) => runAgent(data),
  evaluators: { escalated: (output, target) => (output.action === target?.expected_action ? 1 : 0) },
  groupName: 'refund-escalation',
});
```

```python
from lmnr import evaluate, LaminarDataset

evaluate(
    data=LaminarDataset("refund-escalation-failures"),
    executor=run_agent,
    evaluators={"escalated": escalated},
    group_name="refund-escalation",
)
```

`LaminarDataset` takes an optional `fetchSize` / `fetch_size` (default 25) — set it to a multiple of the eval concurrency for throughput. If the data lives somewhere Laminar doesn't own (a database, S3), subclass `EvaluationDataset` and implement `size()` + `get(index)` (TS) / `__len__` + `__getitem__` (Python) instead of copying it in.

## Verify before running the eval

```bash
# A sample — confirm shape and provenance landed
npx lmnr-cli dataset pull -n refund-escalation-failures --limit 3

# Row count (dataset id from `lmnr-cli dataset list --json`)
npx lmnr-cli sql query "
  SELECT count(*) FROM dataset_datapoints
  WHERE dataset_id = '<dataset-id>'" --json
```

Then sanity-run the eval on the dataset and spot-check 2–3 rows' `data` / `target` / `executor_output` side by side (`evaluation_datapoints` table) — a dataset whose `data` shape the executor can't consume scores 0 on every row and looks identical to "the model is bad at this."
