# Laminar Debugger

Use when building, testing, or debugging an LLM agent instrumented with Laminar. Covers recording a run under `LMNR_DEBUG`, inspecting the resulting trace with the Laminar CLI's SQL, replaying cached LLM calls to iterate fast and deterministically, and annotating debug sessions (names + per-trace markdown notes) so the user can follow what happened.

## Your role

You are the **parent agent**: the coding agent doing the building. The **child
agent** is the AI agent you are working on. Laminar exposes a suite of tools you must use to build, test, and debug more effectively.

You also own a second responsibility the human relies on: **making the debug
session legible**. You name each session and write a markdown note on every
trace, because the user reads those notes — not the raw spans — to understand
what you did and why.

## The core loop

**Record** — run the child agent once under the debugger to capture a trace.

**Inspect** — query the trace to understand what happened and where it went
wrong.

**Annotate** — name the session and write a note on the trace so the run is
self-explanatory in the UI.

**Replay + edit** — make your code/prompt change, then re-run replaying the
cached calls up to the point of interest and executing live past it.

**Repeat** — each iteration only pays for the calls that actually changed.

## Prerequisite: instrument the child agent

Before any of this works, the child agent must be properly instrumented with Laminar. If this has not been done yet, see [instrumentation-typescript.md](instrumentation-typescript.md) or [instrumentation-python.md](instrumentation-python.md), or learn how to instrument [here](https://laminar.sh/docs/tracing/integrations/overview).

## Prerequisite: access the CLI

Make sure the Laminar CLI is working in your environment. See [cli.md](cli.md), or learn more about the CLI [here](https://laminar.sh/docs/platform/cli#cli).

## 1. Record a run

Run the child agent with debug mode on:

```bash
LMNR_DEBUG=true python my_agent.py        # or whatever the run command is
```

Truthy values are `true`, `1`, `yes`, `on`. A debug run:

- joins the session in `.lmnr/debug-session.json` if one exists (or the one in `LMNR_DEBUG_SESSION_ID`), otherwise mints a fresh session, and registers it with Laminar,
- exports all spans as a normal trace, and
- prints a `LMNR_DEBUG_RUN ` line to the console containing the run's ids and debugger URL.

**Always capture and filter stdout/stderr for `LMNR_DEBUG_RUN`.** That line is your handoff between runs — it carries the `trace_id`, `session_id`, and `debugger_url` you need for every subsequent step. Pipe the child agent's output and grep for it explicitly:

```bash
LMNR_DEBUG=true node my_agent.js 2>&1 | tee run.log
grep 'LMNR_DEBUG_RUN' run.log
```

The JSON payload on that line is the same record written to `.lmnr/debug-session.json` (see below):

```json
{
  "session_id": "…",
  "trace_id": "…",
  "replay_trace_id": null,
  "cache_until": null,
  "debugger_url": "https://…/project/<projectId>/debugger-sessions/<sessionId>",
  "started_at": "…"
}
```

Extract what you need with `jq` or a simple pattern match. **Do not rely on the console output being easy to read at a glance** — other logging will drown the Laminar lines, so always grep explicitly.

### Session persistence is automatic — `.lmnr/debug-session.json`

You no longer have to carry the session id between runs by hand. Every debug run reads and writes `.lmnr/debug-session.json`, the default-on handoff file:

- **At startup** the SDK resolves the session id as `LMNR_DEBUG_SESSION_ID` → the file's `session_id` → a freshly-minted UUID. So a second `LMNR_DEBUG=true` run in the same project **silently rejoins the same session** — no env var needed.
- **At shutdown** it writes the run's ids back to the file (`session_id`, `trace_id`, `replay_trace_id`, `cache_until`, `debugger_url`, `started_at`).
- The file is found by **walking up from the current directory** to the nearest ancestor that has one (same nearest-wins rule as `.lmnr/project.json`), so runs and CLI commands launched from a subdirectory join the project's session.

To start a clean, registered session up front — recommended at the top of an investigation — mint one with the CLI. It resets the file and opens the debugger page in the browser:

```bash
npx lmnr-cli debug session new        # writes .lmnr/debug-session.json, prints the session id to stdout
```

After that, just run the child agent; it joins automatically:

```bash
LMNR_DEBUG=true node my_agent.js 2>&1 | tee run.log
grep 'LMNR_DEBUG_RUN' run.log
```

You can still pin a session explicitly with `LMNR_DEBUG_SESSION_ID=<session-id>` — the env var always wins over the file — but it is an override, not a requirement. Continuation is **not** replay: rejoining a session via the file never auto-replays the prior run; replay is armed explicitly (see step 4).

## 2. Name the session and note every trace

This is not optional. The session view is how the human follows your work, and a
bare session of unlabeled traces is unreadable.

Name the session once, describing the investigation. The session defaults to the one in `.lmnr/debug-session.json`, so you normally pass only the name:

```bash
npx lmnr-cli debug session set-name "Fix report length + search tool"
# target a different session explicitly with the flag:
npx lmnr-cli debug session set-name "Fix report length" --session-id <session-id>
```

**CLI ids are always flags, never positionals.** Across the debug surface the payload (a name, a note) is the positional argument and any id is an optional `--session-id` / `--trace-id` flag that defaults from `.lmnr/debug-session.json`. A stray extra positional fails fast instead of being silently treated as an id.

Notes on a trace come in two forms that complement each other.

### Pre-run note (set before you run). Mandatory.

Write this note **before** launching the child agent. It appears in the UI the moment the trace lands there, giving the human a real-time view of your reasoning. Set it via the `LMNR_TRACE_METADATA` environment variable when you invoke the agent:

```bash
LMNR_DEBUG=true \
LMNR_TRACE_METADATA='{"rollout.note": "## What I am about to test\nReplaying up to the search call, running the synthesis call live with the new length cap.\n\n"}' \
node my_agent.js 2>&1 | tee run.log
```

(The run joins the existing session from `.lmnr/debug-session.json` automatically; add `LMNR_DEBUG_SESSION_ID=<session-id>` only to override it.)

Format: `LMNR_TRACE_METADATA` must be a stringified JSON object with key `rollout.note` whose value is your note in markdown. **End the value with a double newline `\n\n`** so subsequent `append-note` entries start cleanly on a new paragraph.

### Post-run note (appended after you run). Optional.

Write a follow-up note on **every** trace after it completes (aim for ~20–200 words of well-structured markdown — headings, short lists, inline code). Record what the trace actually showed, what you observed, and what to look at next.

The note is the positional argument; the trace defaults to the file's `trace_id` (the root trace of the most recent debug run here), so you usually omit the id:

```bash
npx lmnr-cli trace append-note "## What this run showed
The <span id='<spanId>' name='synthesis call' /> now returns ~180 words (was ~600).
Length cap is working. Next: check that citations are still intact."
# target a specific trace with the flag:
npx lmnr-cli trace append-note "Citations intact." --trace-id <trace-id>
```

Notes are **append-only**: each `append-note` call adds a new paragraph to the
trace's existing note — never re-send the whole note, just the new entry.

To re-orient yourself in an ongoing session (e.g. after a context reset), dump
every trace's note in order:

```bash
npx lmnr-cli debug session summary                       # defaults to the file's session; or --session-id <id> / --json
```

Output is one block per trace, oldest first — the note followed by a
`<trace id="…" end-time="…"/>` tag you can feed back into the SQL queries
below.

To reopen the session's debugger page in the browser (works offline, before login):

```bash
npx lmnr-cli debug session open                          # or --session-id <id>
```

Reference a specific span by embedding a **span tag** in the note — the UI
renders it as a clickable **span chip** that opens that span in the trace view:

```text
<span id='<spanId>' name='the synthesis call' />
```

- `id` is the span's UUID — the `span_id` you get from the SQL queries below.
- `name` is the chip's label (free text; keep it short).
- Optional `reference_text='…'` adds a muted inline preview after the label, e.g.
  `<span id='<spanId>' name='synthesis' reference_text='~180 words, was ~600' />`.

The span must belong to the trace the note is attached to.

## 3. Inspect the trace with SQL

The printed URL is optimized for humans; for *you*, querying is faster and more
precise. Every debug run stamps `rollout.session_id` on its trace, so you can
filter to exactly the runs you care about:

```sql
SELECT id AS trace_id, start_time, status, total_tokens
FROM traces
WHERE simpleJSONExtractString(metadata, 'rollout.session_id') = '<session-id>'
ORDER BY start_time DESC
LIMIT 10;
```

Run it through the CLI:

```bash
npx lmnr-cli sql query "SELECT id, start_time, status FROM traces ORDER BY start_time DESC LIMIT 20"
```

To locate the failure, read the trace's spans in order — which LLM call produced
the bad output, what its inputs were, and how far into the loop it happened.
That tells you where to set your replay boundary. `input`/`output` columns are
large, so select them only for the span you care about (and paginate):

```sql
SELECT span_id, name, span_type, start_time, status
FROM spans
WHERE trace_id = '<trace-id>'
ORDER BY start_time ASC;
```

`span_type` is one of `LLM`, `TOOL`, `DEFAULT`, or `CACHED` (a replayed LLM
call in a replay run's trace). A replay boundary points
`LMNR_DEBUG_CACHE_UNTIL` at the **span id of an LLM call along the loop** —
tool executions can't be boundaries. Read the loop's LLM calls in order to find
the one just before the call you want to run live, and grab its `span_id`. If
the source trace is itself a replay, its cached calls (`span_type = 'CACHED'`)
are valid boundaries too, so include them:

```sql
SELECT span_id, name, start_time FROM spans
WHERE trace_id = '<trace-id>' AND span_type IN ('LLM', 'CACHED')
ORDER BY start_time ASC;
```

Discover the full schema any time with `npx lmnr-cli sql schema`. Useful tables:
`spans`, `traces`, `events`, and `signal_events`. See
[sql-query-api.md](sql-query-api.md) for more query patterns.

### Signal events — recent errors and insights

`signal_events` records signals fired during runs (evaluation failures,
flagged conditions, insights). Scan it to surface what recently went wrong
without reading every trace:

```sql
SELECT timestamp, name, trace_id, payload
FROM signal_events
ORDER BY timestamp DESC
LIMIT 20;
```

Join back to the offending trace with the `trace_id`, then drop into its spans.

### Self-hosted / local Laminar

The CLI defaults to `https://api.lmnr.ai`. Point it at a local app-server with
flags (or `LMNR_BASE_URL` / `LMNR_PORT` in the environment):

```bash
npx lmnr-cli sql query "…" --base-url http://localhost --port 8000
```

A run that **mints a fresh session** (no `LMNR_DEBUG_SESSION_ID` and no existing
`.lmnr/debug-session.json`) opens the frontend UI for the user to view. A run that
rejoins an existing session — via the env var or the file — does NOT reopen the
browser. If the Laminar SDK resolves the base URL of the backend to localhost,
the frontend URL is assumed to be `http://localhost:5667`, otherwise the cloud UI
at laminar.sh is opened. Override with `LMNR_FRONTEND_URL`.

## 4. Replay to iterate fast

After editing the child agent, re-run with the replay ids. The session is carried automatically by `.lmnr/debug-session.json`; the two ids you set per replay are the source trace and the cache boundary:

```bash
LMNR_DEBUG=true \
LMNR_DEBUG_REPLAY_TRACE_ID=<trace-id> \
LMNR_DEBUG_CACHE_UNTIL=<span-id> \
node my_agent.js 2>&1 | tee run.log
grep 'LMNR_DEBUG_RUN' run.log
```

This replays the LLM calls along the agent's main loop from the source trace's
cache instead of hitting the model. Calls inside the cache window return their
recorded responses instantly; past it, the run goes live.

**The session id is handled for you.** The run rejoins the session in `.lmnr/debug-session.json`, so all your replay traces land in the same session in the UI without setting `LMNR_DEBUG_SESSION_ID`. Set that env var only to override the file (e.g. to attach this run to a different session).

**`LMNR_DEBUG_REPLAY_TRACE_ID`** tells the debugger which recorded trace to pull cached LLM responses from. Read the `trace_id` from the `LMNR_DEBUG_RUN` line of the run you want to replay. If you want to replay an earlier run (not the most recent one), use its `trace_id` from that run's captured output or from the session's SQL listing. Both `replay_trace_id` and `cache_until` also persist to the file, so they carry into a bare `LMNR_DEBUG=true` run too — but pass them explicitly when iterating so you control the window.

**Replay needs both a replay trace and a cache boundary.** With either unset the run is fully live (no replay). Always set `LMNR_DEBUG_REPLAY_TRACE_ID` and `LMNR_DEBUG_CACHE_UNTIL` together.

**`LMNR_DEBUG_CACHE_UNTIL` is always a span id** (the old numeric-count form has been removed) — replay *through* that span (inclusive: the named call itself comes from cache, the next one runs live). The server resolves the needle, which accepts the span's full UUID, the last two UUID groups, the 16-hex OTel id, or any hex suffix — whatever you copied from SQL or the UI. A value that isn't span-id-shaped is ignored with a warning; a well-formed id that isn't one of the loop's LLM calls runs fully live.

**Cache lookup key: `(trace_id, hash_of_inputs)`.** The hash covers the LLM
call's inputs — messages, tools, model parameters — but **excludes the system
prompt**. This means you can freely rewrite your system prompt between replay
runs and still hit the cache. Changing anything else (first user message, tool
outputs, etc.) changes the hash and causes a miss. Once a miss occurs, the run
goes fully live for all subsequent calls in that iteration.

```bash
LMNR_DEBUG=true \
LMNR_DEBUG_REPLAY_TRACE_ID=<trace-id> \
LMNR_DEBUG_CACHE_UNTIL=<span-id> \
node my_agent.js
```

Replaying up to *just before* the buggy call lets you re-run that one call live
with your fix, over and over, without re-executing everything that led up to it
— pass the id of the call **before** the buggy one (inclusive semantics). Set
the window *past* the change to validate that the rest of the loop now behaves.
Each replayed iteration produces a new trace under the same session, so
attempts compare side by side in the UI (and you should note each one — see
step 2). Replayed traces can themselves be replay sources — their cached calls
count as loop positions just like live ones.

## What to keep in mind

**Replay is best-effort and never blocks you.** If the cache can't be built (no
clear loop in the source trace, or overlapping/parallel calls it can't safely
sequence), the run silently falls back to fully live — you still get a normal
debug trace, just no speedup. A live fallback is not an error.

**Replay assumes a sequential agent loop.** Wildly parallel LLM fan-out won't
replay cleanly; that's expected.

**Restart what doesn't hot-reload.** If the stack has a long-lived component
that loads code (e.g. a Temporal worker), restart it after every edit, otherwise your replay exercises stale code.

**Move your boundary, not your whole approach.** The fastest rhythm is: replay
up to the suspect call → tweak → re-run → read the new trace → adjust the
boundary. Resist re-running fully live every time — that's the cost the debugger
exists to avoid.

**Turn it off for production / normal runs** by simply not setting `LMNR_DEBUG`.
Everything is inert when it's unset.
