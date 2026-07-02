# Laminar eval loop

Use when turning production failures into a measurable fix-and-verify loop: read what's breaking from signals/clusters, replay it as an eval against the agent you're editing, and iterate until the failure mode clears. **This loop runs inside a debug session** — each iteration runs the eval under `LMNR_DEBUG`, so the run lands as an `evaluation` block in the same session timeline as your debugger work, and you journal each run with a note (a `text` block) just like you would on a trace. Read [debugging.md](debugging.md) first; this builds on it.

## Your role

Same framing as the debugger: you're the **parent agent**, the **child agent** is what you're building. You exercise the child over a fixed dataset of its own past failures and watch a score move. **You own session legibility** — an eval run must land as one readable node with a digest note, not a wall of per-datapoint traces.

**Ask the user for the stop threshold before iterating.** "Until what score / pattern should I stop?" — not "I'll iterate until it looks good." Different tasks justify wildly different thresholds (severity ≥ 0.85, "no new failure category appears", "highest reachable in N iterations"). One question, up front. Re-confirm if you cross 5 iterations without hitting it.

**Verify the metric is real before iterating.** A measurement artifact reads as a model failure. Sample 2-3 failing rows from `evaluation_datapoints` (target + executor_output, side by side) and confirm the score field the scorer reads actually exists in the output where it looks, and the per-row score is the one you'd compute by hand. A `_safe`-style decorator silencing a KeyError scores 0.0 on every row and looks identical to "model is bad at this dimension." Five minutes up front saves a run-budget of fake hypotheses.

## The loop

1. **Find the failure mode** — `signal_events`, grouped by cluster (§1).
2. **Freeze a dataset** — pull failing traces' inputs once. Reuse every iteration (§2).
3. **Write the eval** — executor calls the child agent; evaluators invert the signal; `groupName` = cluster id (§3).
4. **Write pre-run note + launch** — `## Change` / `## Why` in `note.md`, edit the child agent (one thing), then run eval with note stamped as `rollout.note` (§4, §7).
5. **Read cheaply** — scores → `summary` of failing rows → full trace, in that order (§5).
6. **Diff** — this run vs previous run in the same group (§6).
7. **Append `## Result` to note.md and journal it** — add the completed note as a session `text` block with `debug session add-note` (§7).
8. **Stop** — target dimension ≥ the user's threshold AND no new failure category vs baseline (§8).

## Two grouping keys — keep them separate

An eval run carries both, and neither derives from the other:

- **`group_id` = the cluster id** (eval's `groupName`). **Durable.** Same value across weeks and sessions, so the progression chart accumulates and regressions show over time. This is the eval's real identity.
- **`rollout.session_id`** (debugger's key). **Ephemeral.** Ties the run to *this* investigation's timeline; resets when you start a new debug session.

Key evals off the session and you lose cross-session progression. Stamp both: cluster id is identity, session id is "show it here."

## How the run lands in the session

A debug session is a timeline of **blocks** (trace / evaluation / text). An eval run joins it by carrying `rollout.session_id` in the **evaluation's** metadata (`evaluations.metadata`): the backend writes one `evaluation` block for the run, and its eval traces also carry `rollout.session_id` and become `trace` blocks. `debug session summary` reads these blocks back (it does NOT scan `traces.metadata`). The session id must land on `evaluations.metadata` — not only the datapoint `metadata` field, which is a different column.

**TS SDK:** `LMNR_DEBUG=1 npx lmnr eval` stamps `rollout.session_id` on the run automatically, resolving the session from `.lmnr/debug-session.json` exactly like a traced run (env `LMNR_DEBUG_SESSION_ID` → file → freshly minted). It also reads `LMNR_DEBUG_RUN_NOTES_FILE` and writes its contents to `evaluations.metadata.rollout.note` — a single pre-run intent note that rides on the run's `evaluation` block (one blob per run, NOT per datapoint).

**Python SDK:** no `LMNR_DEBUG` support. Create the session first with `npx lmnr-cli debug session new` (writes `.lmnr/debug-session.json` and registers it), read the id from that file, and pass `evaluate(metadata={"rollout.session_id": <id>, "rollout.note": Path("note.md").read_text()})`. Same destination column (`evaluations.metadata`), just wired by hand.

## Prerequisites

- **Child agent is runnable from this codebase** (local function or callable endpoint). If you can't run it, you can't close the loop.
  - If the endpoint is behind a build flag (Rust `#[cfg(feature = "...")]`, Go build tags), confirm the running binary was built with the flag. Smoke-test with curl before any eval — a 404 on the endpoint scores 0 on every row and looks like a model failure.
- **`lmnr-cli login`** ([cli.md](cli.md)) is done. This loop drives everything through the CLI. If not logged in, stop and run `npx lmnr-cli setup` first.
  - The CLI resolves the project from `.lmnr/project.json` OR a per-call `--project-id <uuid>`. If you can't run `lmnr-cli setup` in the loop directory (e.g. self-hosted app-server whose project id differs from the linked one), pass `--project-id <uuid>` on every call.
- **A debug session exists** — `npx lmnr-cli debug session new` mints one, **registers a `debugger_sessions` row on the server**, and writes `.lmnr/debug-session.json`. Every later `LMNR_DEBUG=1` run (eval or agent) and CLI call in this directory rejoins it silently. Optional: `npx lmnr-cli debug session set-name "<what you're tuning>"` so the UI shows a readable label.
  - **Do NOT hand-write `.lmnr/debug-session.json`** — a hand-rolled session id skips server registration, so `lmnr-cli debug session summary|add-note|open` and the debugger UI 404 on that id. Always mint via `debug session new`. That command also degrades to a local-only file when the backend is unreachable (WARN + exit 0); treat that WARN as a hard error, fix the backend, retry.
- **Project has recent traces.** The default Failure Detector signal only fires when traces flow. On a brand-new project, §1 returns zero rows — check with `sql query "SELECT count() FROM traces WHERE start_time > now() - INTERVAL 7 DAY"`. If empty, tell the user to instrument first (<https://laminar.sh/docs/tracing/introduction>).

## Hard rules

- **Tier your reads. Never `SELECT` raw span input/output in bulk.** `scores` is a summary; `summary` column is the next tier; full traces are the last resort. Dumping traces blows your context.
- **One note per run.** Journal each run once — the pre-run intent on the run's `evaluation` block plus one post-run `text` block via `add-note` — never a note per datapoint.
- **Always filter by `start_time` / `timestamp`** (ClickHouse scans the whole table otherwise).
- **SQL is SELECT-only, allowlisted.** `evaluation_datapoints`, `traces`, `spans`, `signal_events`, `signal_events_all`, `datasets` are queryable; `evaluations`, `debugger_sessions` are NOT (inspect via UI or direct DB).
- **Avoid ClickHouse joins** — run two queries and combine ids in your own code.
- **Freeze the dataset, change one thing per iteration, keep `groupName` stable** — or the diff is meaningless.
- **Adapt the aggregate SQL to your scorers.** If the eval has multiple evaluators (classification / severity / cost), don't copy-paste a single-scorer template. `cost`-style scorers that record dollar amounts need their own comparison; other scorers each need their own `avg(simpleJSONExtractFloat(scores, '<name>'))` line.

---

## 1. Find the failure mode

Group recent events by cluster:

```bash
npx lmnr-cli sql query "
  SELECT arrayJoin(clusters) AS cluster_id,
         count(*) AS n,
         any(summary) AS example,
         max(severity) AS severity
  FROM signal_events
  WHERE timestamp > now() - INTERVAL 7 DAY
  GROUP BY cluster_id
  ORDER BY n DESC
  LIMIT 20" --json
```

Scope to one signal with `AND signal_id = '<uuid>'`. `signal_events` exposes non-L0 clusters; use `signal_events_all` for leaf membership. Pick the cluster you're fixing and note its `cluster_id`.

## 2. Freeze a dataset

Two queries (different tables — do not join):

```bash
# (a) trace ids in the cluster
npx lmnr-cli sql query "
  SELECT DISTINCT trace_id FROM signal_events
  WHERE has(clusters, toUUID('<cluster_id>'))
    AND timestamp > now() - INTERVAL 30 DAY" --json

# (b) replay inputs -> JSONL datapoints
npx lmnr-cli sql query "
  SELECT id AS source_trace_id, root_span_input FROM traces
  WHERE id IN ('<id1>','<id2>', ...)
    AND start_time > now() - INTERVAL 30 DAY" --json \
| jq -c '.[] | {
    data: (.root_span_input | fromjson? // .root_span_input),
    metadata: { source_trace_id: .source_trace_id, cluster_id: "<cluster_id>" }
  }' > data.jsonl

npx lmnr-cli dataset create cluster-<cluster_id>-failures data.jsonl
```

Targets are usually omitted: production traces have no gold label, so the evaluator checks a *property* (did the failure recur?), not exact match. **Build the dataset once and reuse it by name every iteration.**

## 3. Write the eval

Executor runs the child agent. Invert the signal into a pass/fail evaluator. `groupName` = the cluster id, stable across every iteration:

```ts
import { evaluate, LaminarDataset } from '@lmnr-ai/lmnr';
import { runAgent } from '../src/agent';
import { detectsLoop } from './checks';

evaluate({
  data: new LaminarDataset('cluster-<cluster_id>-failures'),
  executor: async (data) => runAgent(data),
  evaluators: {
    not_stuck_loop: (output) => (detectsLoop(output) ? 0 : 1),
  },
  groupName: 'cluster-<cluster_id>',
});
```

Python is equivalent: `from lmnr import evaluate, LaminarDataset`, with `executor=`, `evaluators={...}`, `group_name=`.

## 4. Launch the run

**Pre-run note first.** Write `note.md` BEFORE launching — the harness reads it at launch time and stamps it on `rollout.note`. Appending `## Result` later updates the file on disk but does not retroactively update already-stamped run metadata.

```markdown
## Change
<one paragraph: file, lines, what you changed>

## Why
<hypothesis: which failure-summary pattern this should move, what tail you're compressing>
```

Then launch:

```bash
# TS
LMNR_DEBUG=1 LMNR_DEBUG_RUN_NOTES_FILE=note.md npx lmnr eval evals/cluster-<cluster_id>.eval.ts
# Python: wire it yourself into evaluate(metadata=...) — the env var is TS-only.
```

Grab the `evaluation_id` from the printed link.

## 5. Read results — cheap first

```bash
# Aggregate (per scorer; adapt names)
npx lmnr-cli sql query "
  SELECT avg(simpleJSONExtractFloat(scores, 'not_stuck_loop')) AS avg_score,
         countIf(simpleJSONExtractFloat(scores, 'not_stuck_loop') < 1) AS failures,
         count(*) AS n
  FROM evaluation_datapoints
  WHERE evaluation_id = '<evaluation_id>'" --json
```

**Failing rows with `summary`** (populated synchronously by the time `lmnr eval` returns — query immediately, don't poll):

```bash
npx lmnr-cli sql query "
  SELECT index, summary, scores, executor_output, data, target
  FROM evaluation_datapoints
  WHERE evaluation_id = '<evaluation_id>'
    AND simpleJSONExtractFloat(scores, 'not_stuck_loop') < 1
  ORDER BY index" --json
```

**Full trace — last resort, one or two max:**

```bash
npx lmnr-cli sql query "
  SELECT name, span_type, status, input, output FROM spans
  WHERE trace_id = '<trace_id>' AND start_time > now() - INTERVAL 7 DAY
  ORDER BY start_time" --json
```

## 6. Diff against the previous run

```bash
npx lmnr-cli sql query "
  SELECT evaluation_id,
         min(created_at) AS run_at,
         avg(simpleJSONExtractFloat(scores, 'not_stuck_loop')) AS score,
         countIf(simpleJSONExtractFloat(scores, 'not_stuck_loop') < 1) AS failures
  FROM evaluation_datapoints
  WHERE group_id = 'cluster-<cluster_id>'
  GROUP BY evaluation_id ORDER BY run_at DESC LIMIT 2" --json
```

Improvement = score up / failures down. Also compare failing-row summaries run-over-run: a category that wasn't there before is a regression you introduced.

## 7. Journal the run

The note is the loop's changelog — the reasoning artifact the human reads weeks later. So it must answer: (1) what observation from the previous iteration's summaries motivated this edit, (2) why this specific change should move the failing bucket without breaking others, (3) what the agent now looks at in the trace it wasn't before. `## Change` and `## Why` are the minimum shape; expand freely to make the intent legible. The `## Change`/`## Why` half rode on the run's `evaluation` block via the pre-run `rollout.note` (§4); after you read the scores you append `## Result` and drop the completed note into the session timeline.

**Post-run: append `## Result`** to the same file:

```markdown
## Result
severity 0.733 -> 0.800 (+0.067), reasoning held, cost +$0.001. KEEP.
Next: tighten the info floor — 3/6 remaining failures are target=warning,
output=info.
```

Then add the completed note to the session as a `text` block:

```bash
# Same for TS and Python — the note is keyed to the SESSION, not a trace.
# It reads the session id from .lmnr/debug-session.json (the one `debug
# session new` / the eval run wrote); pass --session-id to target another.
npx lmnr-cli debug session add-note "$(cat note.md)"
```

Each `add-note` adds a new `text` block, interleaved by time with the run's `evaluation` / `trace` blocks — the same kind of note you'd write on a debug trace, keyed only by session id (no trace id to resolve). `npx lmnr-cli debug session summary` dumps the whole session timeline oldest-first (traces, evals, and these notes) — that's the human's changelog.

**Replay caching may not apply.** The debugger's record/replay speedup only helps when the agent's LLM calls happen *in the process you run*. If the agent under test runs server-side (you POST inputs to an endpoint), there's nothing local to cache — you get the session + journal layer, not replay. Per-iteration cost lever there is the dataset size.

## 8. Stop

Stop when, on the frozen dataset: target dimension ≥ the user's threshold **and** no failing-row summary describes a category that wasn't in the baseline. If you cross 5 iterations without hitting the target, surface the latest scores + remaining pattern and ask whether to keep going, lower the threshold, or stop. Don't silently grind past the budget the user has in mind.

---

## Schema you will touch

Full schema: `npx lmnr-cli sql schema`, or <https://laminar.sh/docs/platform/sql-editor#table-schemas>. Loop-relevant columns:

- **`signal_events`** — `trace_id`, `signal_id`, `summary`, `payload` (JSON string), `clusters` `Array(UUID)` (non-L0; `signal_events_all` for L0), `severity` (0 INFO / 1 WARN / 2 CRIT), `timestamp`.
- **`traces`** — `id`, `metadata` (`rollout.session_id` / `rollout.note` for debug runs), `root_span_input` / `root_span_output` (parse as JSON, fall back to string), `status`, `start_time`.
- **`evaluation_datapoints`** — `evaluation_id`, `group_id` (= cluster id), `index`, `data` / `target` / `executor_output` / `scores` / `metadata` (JSON strings; `scores` is `{name: number}`), `summary`, `trace_id`, `trace_metadata` (mirrors trace metadata — `rollout.session_id` lands here too), `created_at`.
- **`spans`** — `trace_id`, `name`, `span_type`, `input` / `output`, `status`, `attributes` (JSON string), `start_time`.

JSON columns are guaranteed valid objects — use `simpleJSONExtract*` (fast) or `JSONExtract*` (nested) in-query. `input` / `output` / `root_span_*` may be raw strings; try JSON, fall back. Use `ILIKE` on `input` / `output`.
