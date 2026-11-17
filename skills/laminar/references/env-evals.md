# Environment evals with Harbor

Each eval case is a Harbor **task**: an isolated container that boots the real application and its
dependencies, seeds the case's data, drives one entry point, and a verifier that scores what happened.
You are building a thin harness around code that already exists. The default is to change **no
production code**.

Run the evals with `lmnr-cli eval run`, not with `harbor run` directly. The CLI builds the images, then
runs a pinned Harbor (0.25.0) with the Laminar Harbor plugin (https://laminar.sh/docs/integrations/harbor)
through `uvx`, so nothing needs installing beyond `uv` and Docker. Every trial becomes a Laminar trace and
an evaluation datapoint, so the agent's trajectory (LLM calls, tool calls) lands in Laminar next to the
reward and is inspectable with the rest of this skill (SQL, signals, debugging) instead of a pile of log
files. The same folder runs locally and on Laminar Cloud without changes, so set it up exactly as below.
Its tasks are stored and versioned in Laminar ("Task sets"), so the same cases run again against every new
version of the agent.

## Harbor's design, and how this use differs

Harbor was built for benchmarks like Terminal-Bench: a **general-purpose agent** with a terminal is dropped
into wildly different tasks, so every task brings its own Dockerfile, instruction and tests.

Production evals are the opposite shape. The thing under test is **one product** (an agent, a pipeline,
an LLM-backed endpoint), and the cases are variations of the same scenario: same code, same services,
same entry point, different data. Stay compatible with Harbor's task format, but lean on that sameness:

- **one prebuilt image** for every case, not a Dockerfile per task;
- **one agent class** that drives the product's entry point, not a general terminal agent;
- **per-case data** as the main thing that varies between tasks.

## Running: `lmnr-cli eval run`

```bash
lmnr-cli eval run                       # every agent in ./evals, every task set, trials on local Docker
lmnr-cli eval run support-bot --task-set refunds --task "refund-*" --n-attempts 3
lmnr-cli eval run support-bot --task refund-double-charge   # just one case
lmnr-cli eval run support-bot --task-set refunds@4          # a published version from Laminar
lmnr-cli eval run support-bot --remote  # run in Laminar, print the run's log until it ends
```

Run from the repo root: `.env` / `.env.local` are read from the current directory. Both modes need Docker
on the machine that runs `lmnr-cli`, which builds the `agent.yaml` images there.

- **Local (default)** is the iteration loop: run it after every change. It needs `uv`, Docker, and
  `LMNR_PROJECT_API_KEY` (`lmnr-cli setup` writes it to `.env`).
- **`--remote`** scales out (one sandbox per trial) and leaves everything needed to debug a run in Laminar
  for the whole team. `lmnr-cli` builds or pulls the images the run's tasks use with local Docker for
  `linux/amd64`, uploads only the layers Laminar lacks, starts the run, and prints its log until it ends
  (Ctrl-C stops following; the run goes on). It needs `lmnr-cli login` (or `LMNR_PROJECT_API_KEY` in CI).
  On the evaluation page each trial has its trace, its task folder (Task) and its Harbor output (Log:
  `/logs/agent/` files, verifier output, `exception.txt`): debug remote failures there. Use it for many
  cases at once, for runs others will read, and when the machine can't run the stacks.
- `--n-concurrent`, `--n-attempts`, `--model` and `--task <glob>` override `agent.yaml` for one run;
  `--env KEY[=VALUE]` (repeatable) adds an agent env var, and a bare `KEY` is read from your environment.
- `--task` matches task directory names. Only the tasks a run includes get their images built and their
  `${VAR}`s required.
- Each task set is one evaluation group in Laminar, `<agent>/<set>`. Every run creates a new evaluation in
  that group, so local and remote runs are charted together.
- `--json` prints `{mode, taskSets, failed, url}` (`--remote`: `runs` with each run's `evaluations` and
  `exitCode`). Exit codes: `16` invalid evals folder or upload too large, `17` `uv`/`docker` missing, `18`
  a task set's `harbor run` failed (the others still run), `19` an unset `${VAR}`, `20` a Docker
  build/pull/save failed, `21` a remote run failed or was lost, `22` merge conflict, `23` not in Laminar.

**Remote limits and image requirements:**
- One container per trial (no docker-compose tasks); `cpus`/`memory_mb` above the sandbox cap are clamped;
  a run that exceeds the deployment's time limit is stopped.
- Agent scripts and each task set upload as one archive each (max 1 GiB), only when they changed.
  Symlinks are skipped, so are directories named `jobs`, `.git`, `node_modules`, virtualenvs and caches;
  `.env*` files are never packed. Keep large fixtures out.
- Images need `tar`, `gzip` and `bash` (and `su` when the task runs as a non-root user). They are built for
  `linux/amd64`, emulated (slowly) on ARM Macs; architecture-specific downloads should use `ARG TARGETARCH`.
- Remote runs drop `LMNR_*` vars from the agent env. Against self-hosted Laminar, set `LMNR_GRPC_PORT`
  (shell or `.env`) or the trial traces don't arrive.
- Sandboxes per project are capped (50 by default): a higher `--n-concurrent` queues. `[environment]
  build_timeout_sec` (default 600) bounds image preparation plus sandbox start; raise it for huge images.

If the user asks why Laminar instead of plain Harbor: the tasks, image, agent and verifier are plain
Harbor (bare `harbor run` needs the `agent.yaml` images built with `docker build -t <name>` and the `job`
given the task set as a dataset). Laminar adds full trajectories (when the app is instrumented), versioned
task sets, run history and diffs shared with the team, eval traces in the same SQL tables as production,
signals and clusters ("Turning production data into cases"), and `--remote`. When bare Harbor is enough,
say so.

## Task sets: versioned in Laminar

The **agent scripts** (all of `evals/<agent>/` but `task_sets/`) depend on the app's code, so they live in
the repo and change with it: `eval run` only uses the repo's copy. The **task sets** are data, stored and
versioned in Laminar, their source of truth: cases are added there without touching the repo, and a
version re-runs against any later agent. Laminar has no agents: task-set names are project-wide (two
agents can't both have `smoke`). The repo pins the version each set runs at, in `agent.yaml`:

```yaml
task_sets:          # edited by hand; CI runs these versions
  refunds: 3
```

```bash
lmnr-cli eval pull support-bot --all    # the sets agent.yaml pins (+ scripts if there are none)
lmnr-cli eval pull support-bot/refunds  # one task set (latest); support-bot/refunds@3 for one version
lmnr-cli eval push support-bot/refunds  # publish it as the latest version; support-bot = every local set
lmnr-cli eval pull support-bot          # the scripts the last run used, to commit to the repo
lmnr-cli eval list                      # the project's task sets, latest versions
```

- Which version a run uses, per set: `--task-set refunds@4` (or `@latest`), else a local copy in
  `task_sets/refunds/` as is, else the pin, else (only when named with `--task-set`) the latest. A bare
  run takes the pinned sets plus local copies and skips the rest.
- `pull` and `push` write `evals/<agent>/.gitignore` with `task_sets/`: local copies are working copies, not
  committed, so CI runs the pins. Commit that file.
- `pull` merges task by task with your copy: tasks changed only in Laminar are updated, tasks changed only
  locally are kept, and a task changed on both sides fails (exit `22`) before anything is written. The
  version a copy is based on is in `task_sets/<set>/.lmnr.json`.
- `push` sends that base; anything pushed since is merged in first, the same way. On a conflict, move your
  copy of that task aside, pull, and re-apply your change. It prints the new version: bump the pin in
  `agent.yaml` to run it in CI.
- **A run never publishes.** Its evaluation keeps the task files and scripts it ran, and links to a
  task set version only when the copy matches a pushed one exactly; only `push` creates versions.
- Commit the agent scripts: a run without `evals/<agent>/agent.yaml` fails (exit `23`). `pull <agent>`
  fetches the scripts the last run used (e.g. ones the Laminar agent wrote); a local file that differs
  fails it (exit `22`) unless `--force`.

## Principles

1. **Production fidelity over convenience.** Run the real code and real services (database, queue,
   cache, object store) for as much of the stack as possible. Seed them; don't mock them. Mock only what
   you truly cannot run (third-party SaaS, private internal services), as a last resort, at the network
   edge (base URL / profile config), never by editing the code path. **Tell the user about every mock**
   and why it was needed; a mock is a gap in what the eval measures.
2. **Zero production changes.** Find an existing public entry point (an HTTP route, CLI command, queue
   message, exported function) and drive that. A harness-only code path tests the harness. If no existing
   entry point reaches the behaviour, stop and raise it with the user (see "When the harness doesn't fit").
3. **One case = one isolated environment.** Every container boots its own stack with its own data, so
   cases run in parallel with no shared state and no ordering. Never share a database between cases.
4. **The agent scripts live next to the code** and the image is built from the same source tree and
   revision, not from vendored copies; tasks hold only data (see "Staleness").
5. **Turn the run into artifacts, then verify the artifacts.** The forward pass should leave its observable
   result in files (a `result.json`, an exported row, a transcript). A verifier is arbitrary code: a
   deterministic comparison, a rubric, or LLM-powered grading over those artifacts (see "The verifier").
   Pick what fits the case; a discrete outcome (`event` / `no_event`, `pass` / `fail`) is simply the
   easiest to start with.

## Procedure

Work through these in order; each step links to the section with the details.

1. **Find the entry point and the observable outcome.** Read the production code for the path under test:
   the **entry point** you can invoke without modification (route, CLI, queue consumer, function); the
   **inputs it reads** (rows, spans, files, messages) and where they live; the **terminal state** it
   produces (a response body, a status row, an emitted record) and how to read it; the **external calls**
   (LLM, HTTP) and how they are configured (env, config row, profile). If existing tests seed a fixture for
   this path, copy their seeding shape: it is the best statement of which fields are actually required.
2. **Lay out `evals/<agent>/`**: `agent.yaml`, and one task in `task_sets/<set>/` with `task.toml`,
   `instruction.md`, `environment/case.json` and `tests/` ("Directory structure", "Use one prebuilt image").
3. **Build the one image and boot the services** with a start script in `image/shared/`. Pin service
   versions to what production/compose uses ("Services", "Harbor runtime facts and quirks").
4. **Seed through the real interface where it is cheap, directly where it is not** ("Data isolation",
   "Seed data the agent can't tell is a test"):
   - Use the app's own API for anything that involves encryption, validation or derived fields (e.g.
     creating an LLM profile with secrets: the server encrypts them; don't hand-encrypt).
   - Insert directly into the database for things the API gates behind plan limits or auth you don't have,
     and for bulk fixtures (traces, messages, records). Mirror the columns the app actually reads; omit the
     ones with defaults. Hash credentials exactly the way the app does (read its auth code).
5. **Write the agent and the driver: trigger, then wait for a terminal state** ("The agent").
   - **Synchronous entry point** (an HTTP call or function that returns the result): call it with a timeout
     and record the response.
   - **Asynchronous entry point** (enqueue a job, the app writes a status later): don't sleep. Poll the
     status the app writes until it is terminal, with a hard timeout shorter than the agent's timeout.
   - Map to an outcome the verifier can grade (a discrete value, plus `failed` and `timeout` so those are
     never mistaken for a model verdict). Write the full detail (status, error message, payload, ids,
     duration) to `/logs/agent/result.json` and print it.
6. **Write the verifier** ("The verifier").
7. **Declare secrets and document the folder**: `${VAR}` placeholders, `.env.example`, `.gitignore` for
   `.env` and `evals/*/jobs/`, and `evals/README.md` saying how to run and how to add a case ("Secrets and
   env vars").
8. **Run it locally** from the repo root with `lmnr-cli eval run <agent>` and fix until it passes. Then verify the
   harness, not just the case: run a **positive** and a **negative** case that share the same
   prompt/config and differ only in input, and flip the reference answer once to confirm the reward goes
   to 0. A harness that can't fail is broken.
9. **Check the trajectory** in the trial's trace: LLM and tool calls should be visible ("Capturing the
   agent's trajectory").
10. **Optionally run `--remote`** once to confirm the folder works in Laminar sandboxes ("Running").
11. **Publish** with `lmnr-cli eval push <agent>`, pin the versions in `agent.yaml`, and commit the agent
    scripts ("Task sets").

## Data isolation

Each case starts from exactly the data it needs. Decide per dataset which kind it is:

- **Shared baseline data**: the same for every case (reference tables, a default org/user/project, config
  rows, an LLM profile). Seed it once in the image build, or in a shared seed script that runs at startup
  in every container. Each container still gets its own copy, so isolation holds.
- **Per-case data**: what makes the case reproduce a specific behaviour from production (the conversation,
  the spans, the records that triggered a failure). This is usually the bulk of the case. It lives in the
  task's `environment/` directory and is seeded at startup by the same shared seed code for every case.

Most real setups have both. Keep per-case data in one format your seed script reads (`case.json`, a SQL
file, a directory of fixtures), so the seed script is shared and only the data differs.

## Directory structure

This layout is what `lmnr-cli eval run` expects: one directory per **agent** (the product under test)
inside `evals/`, with its **task sets** inside it. Files marked *(optional)* depend on your setup.

```
evals/
  README.md                       # how to run, how to add a case
  <agent>/                        # e.g. support-bot
    agent.yaml                    # the images tasks use + the Harbor job (agents, concurrency, ${VAR}s)
    adapter.py                    # BaseInstalledAgent subclass: starts the stack, runs the driver
    image/
      Dockerfile                  # the one image: services, app artifact, shared scripts, baseline data
      shared/                     # copied into the image by the Dockerfile
        start_stack.sh            # boot dependencies + the app, wait for readiness
        schema/                   # (optional) schema setup when the app has no migrations of its own
        seed_baseline.*           # (optional) data every case shares
        seed_case.*               # load this case's data from the uploaded environment/ files
        run_case.*                # drive the entry point; write result files
        verify.*                  # (optional) shared grading logic each test.sh calls
        export_trajectory.*       # (optional) rebuild the agent's trajectory when the app isn't instrumented
    .gitignore                    # task_sets/ (written by eval pull)
    task_sets/                    # working copies of Laminar's task sets, not committed ("Task sets")
      <set>/                      # e.g. refunds
        README.md                 # (optional) what the cases are and where they came from
        <case-name>/
          task.toml               # [environment] docker_image = "<image name from agent.yaml>"
          instruction.md
          environment/            # no Dockerfile: uploaded into the container's workdir at start
            case.json             # this case's data; any format seed_case reads
          tests/
            test.sh               # the verifier entry point (any language, any grading method)
            expected.json         # (optional) reference answer, if the verifier compares against one
```

`agent.yaml` names each image and how to build it (paths relative to the agent directory; make `context`
the repo root so the image copies the app from the live tree), and holds `job`: a Harbor JobConfig without
`datasets`/`tasks`, run once per task set with that set as the dataset. Harbor runs from the agent
directory, so `import_path` is relative to it:

```yaml
images:
  myapp-evals:
    dockerfile: image/Dockerfile
    context: ../..
job:
  agents:
    - import_path: adapter:MyAppAgent
      model_name: anthropic/claude-sonnet-5-5   # read as self.model_name; overridable with --model
      env:
        ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
  n_concurrent_trials: 4                         # what the local machine can run
task_sets:                                       # pinned versions ("Task sets")
  refunds: 3
```

Order the Dockerfile from rarely to often changing, so an app change rebuilds (and for `--remote`
uploads) one small layer. `COPY` the specific paths the image needs, never `COPY . .` from the repo root:
BuildKit only sends what `COPY`/`ADD` reference, so a large repo costs nothing, while `COPY . .` would bake
in all of it, Harbor's job output and every task's `tests/` (reference answers) included. From `evals/`,
copy `<agent>/image/shared/` only. `.dockerignore` applies, and `lmnr-cli` never lets `.git` or `.env*` into a build.

```dockerfile
FROM python:3.12-slim-bookworm
RUN apt-get update && apt-get install -y --no-install-recommends \
      postgresql redis-server curl && rm -rf /var/lib/apt/lists/*
COPY requirements.txt /srv/app/
RUN pip install --no-cache-dir -r /srv/app/requirements.txt
COPY evals/support-bot/image/shared/ /opt/harness/
COPY migrations/ /srv/app/migrations/
COPY app/ /srv/app/app/
WORKDIR /case
```

- Keep what differs per case small (data, instruction, reference answer) and everything else shared, so a
  new case is a directory copy plus data and never needs harness changes.
- Keep reference answers in `tests/`, not in `environment/`: Harbor uploads `tests/` only when the
  verifier runs, so neither the agent nor the app can read them.

## Use one prebuilt image

Set `docker_image` in each task's `task.toml` to an image name from `agent.yaml`, and leave the Dockerfile
out of `environment/`:

```toml
[environment]
docker_image = "myapp-evals"
workdir = "/case"          # where environment/ gets uploaded; defaults to the image's WORKDIR
cpus = 2
memory_mb = 4096
```

- **Speed.** Every trial starts from the same image, built once per run (cached when nothing changed);
  `--remote` uploads only missing layers.
- **Per-case files still arrive.** With `docker_image` set and no `Dockerfile` in `environment/`, Harbor
  uploads `environment/` into the workdir after the container starts: that's how `case.json` arrives.
- **Resources.** A full stack per case is heavy: set `cpus`/`memory_mb`. Locally trials share the host, so
  keep `n_concurrent_trials` to what it can run; raise it for `--remote` runs with `--n-concurrent`.
- **No per-task Dockerfile.** `lmnr-cli` rejects a task with `environment/Dockerfile` or
  `docker-compose.yaml`: tasks are data, and code in them would drift from the app. A case that genuinely
  needs a different system setup gets its own image in `agent.yaml`.
- An image name that isn't in `agent.yaml` is pulled from its registry, so public images work as-is.

## Secrets and env vars

Declare every secret and setting the run needs as a Harbor `${VAR}` placeholder, never as a literal and
never as a `harbor run` flag:

- agent env in `agent.yaml` (`job.agents[].env`, as above). It reaches every command the agent runs in the
  container (see "The agent");
- `[verifier.env]` in `task.toml` for an LLM-graded verifier, `[environment.env]` for the container.

`${VAR:-default}` makes one optional. `lmnr-cli eval run` reads each name from the shell, then
`.env.local` / `.env`, and fails before anything starts (exit `19`) if a required one is missing. For
`--remote` the values travel with the run as env vars; files with secrets are never uploaded. Add `.env` and Harbor's job output
(`evals/*/jobs/`) to `.gitignore`, and write a `.env.example` listing the names.

## The agent: subclass `BaseInstalledAgent`

Harbor has two agent interfaces (https://docs.harborframework.com/agents/custom-agents; `lmnr-cli` pins
Harbor 0.25.0, so write against that version):

- **`BaseInstalledAgent`** (`harbor.agents.installed.base`): the agent runs **inside** the task
  environment. This is Harbor's general recommendation and the right fit here: the product under test
  already lives in the container, so the agent is just the code that starts it and drives its entry point.
- **`BaseAgent`** (`harbor.agents.base`): the loop runs on the **host** and reaches the container through
  `environment.exec(...)`. Only for an agent that *must* run outside the container (rare here).

Shape of the installed agent:

- `name()` (static), and `run(instruction, environment, context)` decorated with `@with_prompt_template`.
- `install(environment)` is required and runs in every trial, before `run()`. With a prebuilt image leave
  it as `pass`: anything it would install belongs in the image. Per-trial setup (config files, credentials)
  goes at the start of `run()`; there is no other hook for it.
- `run()` stays thin: start the stack, seed baseline + case data, run the driver, done.
- Run commands with `self.exec_as_agent(environment, command=..., env=..., cwd=..., timeout_sec=...)`, or
  `exec_as_root` for things that need root. The `agents[].env` values from `agent.yaml` are added to every
  command the agent runs (not to the verifier), so the app gets them without `env=`; use `env={...}` only
  for per-command extras. The trap is a start script that launches the app with `env -i`: it must forward
  the names the app needs. Use `self.model_name` for the model so `--model` works.
- A non-zero exit raises, so have the driver exit 0 whenever it produced an outcome (including `failed` /
  `timeout`) and non-zero only for harness errors.
- `populate_context_post_run(context)`: runs after logs are synced to the host; parse `result.json` and put
  the result into the context.
- Reference it from `agent.yaml` as `import_path: adapter:ClassName` (relative to the agent directory).
- Import only `harbor`, the standard library and modules inside the agent directory: the class itself runs
  in the Harbor process (on the user's machine or Laminar's orchestrator), not in the container. Anything
  else belongs in the image.

## The verifier

`tests/test.sh` is the entry point, and Harbor uploads `tests/` only at verification time. It can be any
language and any grading method, and writes its reward (e.g. `1` or `0`) to `/logs/verifier/reward.txt`.
Keep grading logic in the image (`image/shared/verify.*`) and per-case expectations in `tests/`, with
`test.sh` a thin call between them, so a change to the app's output is fixed once in the agent scripts,
not in every task. Where it runs is `[verifier] environment_mode` in `task.toml`:

- **`"shared"`** (the default): runs in the agent's container after the agent finishes and can read
  anything the run left behind, e.g. `/logs/agent/result.json`.
- **`"separate"`**: runs in a fresh container and sees only what the task declares. List files in
  top-level `artifacts = [...]` in `task.toml`, and use `[[verifier.collect]]` commands to snapshot runtime
  state (database contents, in-memory counters) into files before the environment is torn down. Prefer
  this when the verifier needs its own dependencies, or must not see the app's internals.

## Capturing the agent's trajectory

The trial trace is only as rich as what the product under test reports. Without a trajectory, the trace
has a single span for the agent phase: you see the reward, but not the LLM calls and tool calls that led
to it, which is what you need to debug a failing case. Check which of these applies, in this order.

### The app is instrumented with Laminar (preferred)

If the product calls `Laminar.initialize()` (Python/TS) or `registerTelemetry()` (AI SDK v7+), its spans
become the trial's trajectory: every LLM call with tokens and cost, and tool calls as `TOOL` spans when
tools are covered by framework auto-instrumentation or wrapped with `observe` / `@observe`. Nothing to
convert.

During an eval run the plugin sets `LMNR_TRAJECTORY_PATH=/logs/agent/trajectory.json` in the agent env.
With it, the SDK writes its spans to that file instead of sending them to Laminar, and the plugin attaches
the file to the trial trace. The app needs no Laminar API key and no network access to Laminar for this.
Make sure that:
- the app's SDK writes the file: `@lmnr-ai/lmnr` 0.8.54 or later, `lmnr` (Python) 0.8.4 or later. Older
  versions, and apps that export through a plain OpenTelemetry exporter, write nothing, and the trial has no
  trajectory. Upgrading is a production change: propose it, or reconstruct the trajectory (below);
- the app process actually receives `LMNR_TRAJECTORY_PATH` (forward it if the start script uses `env -i`);
- you don't also write a reconstructed trajectory (below), or the agent's steps show up twice.

If the app isn't instrumented, adding Laminar is a production change: propose it to the user rather than
doing it inside the eval work. It is small, and production traces benefit too (see
[instrumentation-python.md](instrumentation-python.md), [instrumentation-typescript.md](instrumentation-typescript.md)).
Some apps can't take it now: another language, another tracing setup, or not this change.

### Not instrumented: reconstruct it from the app's own history

Most products persist the conversation (chat history, audit, the agent's memory). The code that renders or
replays it tells you where it lives and how tool calls are stored. Common places:
- message/thread tables behind the chat UI (often with structured parts for tool calls and results);
- agent run/step/event tables, audit logs;
- framework state: checkpointers (LangGraph), session stores (agent SDKs), JSONL transcript files;
- the entry point's response, if it returns the full message list.

After the driver finishes, including on `failed` / `timeout` (partial trajectories are the most useful),
run a shared exporter (`image/shared/export_trajectory.*`) inside the container. It reads this case's
history from the store and writes `/logs/agent/trajectory.json` in Harbor's ATIF format, which the plugin
replays as `LLM` and `TOOL` spans:
- each message becomes a step with `step_id` (required; 1, 2, 3… in order), `source` (`system` / `user` /
  `agent`), `message`, and an ISO 8601 `timestamp` from the row's creation time;
- `model_name`, `reasoning_content`, `tool_calls` and `metrics` are only allowed on `agent` steps;
- history seeded from `case.json` (what happened before the run) gets `is_copied_context: true`, so it is
  shown but not mistaken for the agent's work;
- tool calls go into the agent step's `tool_calls` (`tool_call_id`, `function_name`, `arguments` as an
  object), and their results into the same step's `observation.results` (`source_call_id` matching one of
  that step's `tool_call_id`s, `content`). If the store has no call ids, generate them and pair calls with
  results by order. Infer a tool call from side effects (rows it wrote) only when it is unambiguous;
  otherwise leave it as message text;
- top-level: `schema_version`, `session_id`, `agent` (`name` and `version` required, `model_name`) and
  `steps`. Unknown fields fail validation; put custom data in `extra`.

Stored history usually lacks token counts, cost, the system prompt, each LLM call's exact prompt, retries
and latency: leave `metrics` unset rather than estimating, and say in `notes` where it came from (e.g.
`"reconstructed from chat_messages"`). In `populate_context_post_run`, validate the file with
`harbor.models.trajectories.Trajectory` and log (don't fail the trial) if it doesn't parse. Tell the user
the trajectory is reconstructed and what it lacks.

If the app keeps no history at all, the trial has only `result.json` and the service logs. Say so and
suggest instrumentation.

## Seed data the agent can't tell is a test

Capable models notice when they're being evaluated, and an agent that suspects a test may behave
differently than in production. Fixtures are where this leaks most.

- **Timestamps.** This takes judgement. Production data exported weeks ago, replayed today, reads as
  stale: "the last activity on this account was 5 months ago" is a hint the agent is looking at a fixture.
  - Default to **relative** timestamps: store offsets (or shift on load) so the most recent event is near
    "now" and the intervals between events are preserved exactly.
  - Keep **absolute** timestamps when the case depends on a specific date: end-of-month logic, a timezone or
    DST boundary, an expiry, a schedule. Then make the rest of the data consistent with that date.
  - Don't forget dates that aren't in timestamp columns: years in free text, filenames, version strings,
    "as of" lines in documents, ids that embed time (UUIDv7, ULID, snowflakes).
- **Ids.** Patterned ids give the game away (`12000000-0000-4000-8000-0123456789ab`, `user_1`). Generate them
  the way the app does (UUIDv4, v7 consistent with the timestamps, `cus_...`), keeping references
  consistent. Deterministic per case (seeded by the case name) is fine.
- **Names and content.** No `John Doe`, `test@example.com`, `Acme`, `lorem ipsum`; replace scrubbed PII
  with plausible values.
- **The harness itself.** If the agent under test can see the filesystem, process list or environment,
  keep harness artifacts out of sight or neutrally named: no `/eval`, `EVAL_MODE=1`, `fixture` or
  `test_case` in anything it can read.

## Services

Goal: the container contains everything the entry point touches, started the way production starts it.
Single container, no docker-in-docker: boot the real dependencies in it.

- *Databases.* Start them as local background processes with the engine's own bootstrap (`initdb`,
  `clickhouse server`, `mysqld`, `redis-server`...). Use trust/no-password auth locally and a throwaway
  password where the app's connection string requires one.
- *Getting the binaries.* The simplest source of the version production runs: the distro package manager,
  the vendor's apt repo or release tarball, or a multi-stage `COPY --from=<official image>:<version>` (with
  the shared libs it needs).
- *Schemas.* If the app has migrations, reproduce what its own startup/migration does, in the same order:
  extra roles, extra databases, numbered migrations, then anything created *after* migrations in code
  (dictionaries, views, grants, seed rows). Reading the app's boot hook is the only reliable way to find
  these. Apply numbered SQL files in natural sort order (`sort -V`), skip archive subdirs.
  If the app has no migrations (schema managed elsewhere, ORM auto-create off, a shared database owned by
  another service), infer the schema from the code: model/schema classes, ORM definitions, the queries the
  entry point runs. Create **only the tables and columns the path under test needs**, in init SQL files
  (e.g. `/docker-entrypoint-initdb.d`-style scripts your start script runs). Tell the user the schema was
  inferred, since it can drift from the real one.
- *Queues/caches/object stores.* If the app has an in-memory or local-disk fallback, use it by leaving the
  URL env unset. That is production-supported code, not a mock.
- *Third-party edges.* Keep external SaaS behind the app's existing base-URL/config.
- *The app itself.* Run the production artifact: the real package imported/started by the start script,
  or a compiled binary. For a library or callable function, the driver script imports and calls it
  directly in-process; the service boot still happens first.
- *Readiness.* Poll the health/readiness endpoint with a timeout; on failure `tail` the app log to stderr.
- *Env.* Start the app with `env -i` and an explicit minimal env so the host's shell never leaks in, and
  forward the agent env names the app needs (API keys, `LMNR_TRAJECTORY_PATH`).

## LLM handling
- **The LLM is real.** Evals exist to measure how the product behaves with a model; a canned response only
  tests the harness. Never substitute a fake or recorded model for the one under test.
- Route the app's model calls through its normal configuration (profile row, base URL env), pointing at the
  endpoint and key from the agent's `${VAR}` env. Don't patch the client.
- A failing LLM endpoint often makes the app **retry rather than fail**; your timeout ends the run, so
  report `timeout` distinctly from `failed`.

## Staleness
- `lmnr-cli eval run` rebuilds the `agent.yaml` images on every run, so a change to `image/shared/` or to
  the app is picked up next run.
- Tasks are versioned apart from the code, so keep everything that depends on the app's code (start
  script, seed loader, driver, grading) in the agent scripts, where a refactor updates it in the same
  commit, and keep tasks to data in a stable format the seed loader reads.
- Build from the **live source paths** (the app source, its migrations), not from a vendored copy, so evals
  can't drift from the schema they run against. The app artifact and the migrations in one image come from
  the same revision by construction.
- Docker layer caching can still hide a change (a `RUN` that downloads "latest"): check the file in a
  running container, then `docker builder prune` or bump a cache-busting `ARG` before the stale step.

## Compiled artifacts
If the product ships as a compiled binary, build it in a builder stage of the same Dockerfile, not on the
host (wrong format on macOS; cross-compiling breaks on native dependencies).
Use a builder base whose libc is not newer than the runtime stage's, and install the shared libs the binary
links at runtime. Check what it reads relative to its working directory (templates, data files); a server
missing them can start but silently never bind its port.

## Harbor runtime facts and quirks
- Harbor starts the container with its own keep-alive command (`sleep infinity` on local Docker), so don't
  rely on `CMD` or a base image's entrypoint (the `postgres` image's init) to start anything. The start
  script launches every service itself as a background process (`nohup ... &`, output to
  `/logs/agent/*.log`) and waits for readiness. Redirect the output, or the exec call may not return; the
  services survive after it does.
- **No ports are published to the host.** Every call to the app must come from a process inside the
  container, which is another reason to use an installed agent.
- `/logs/agent/` and `/logs/verifier/` are shared between container and host. Write `result.json` and
  every service log under `/logs/agent/`.
- Some services refuse to run as root (Postgres). Use `gosu`/`su` to a service user, and make sure the
  log directory is writable by that user (`chmod 777 /logs/agent`).
- Some servers need a password in the connection string even when the database uses trust auth; use a dummy.
- Keep exec timeouts below the task's `[agent] timeout_sec`, and the driver's own timeout below both.
- **Secret scrubbing:** Harbor redacts agent env values from its output, so a one-character placeholder
  like `x` turns every `x` into `[REDACTED]`. Keep placeholder values long and distinctive.
- **Job output:** `evals/<agent>/jobs/<timestamp>/` for local runs. Each trial has `agent/`
  (your logs, `result.json`) and `verifier/` (`reward.txt`, `test-stdout.txt`); `exception.txt` holds the
  agent traceback.

## Turning production data into cases
The loop this setup exists for: pick production traces a signal flagged (and a few it correctly didn't),
turn each into a case's data, grade whether the failure recurs, and add the case as a directory copy. To
build cases from a cluster or a signal's events (choosing and deduplicating traces, mapping a trace onto
seed data, the verifier, what to report), read [evals-from-clusters.md](evals-from-clusters.md).

Once the harness exists, adding a case is: copy a task directory, drop in the new case data and reference
answer, run `lmnr-cli eval run <agent> --task-set <set> --task <new-case>`, then the whole set, and
`lmnr-cli eval push <agent>/<set>`.

### When the harness doesn't fit
Adding a case should not need harness changes. If one does, first try to generalise the shared seed/driver
code so every case benefits. If the behaviour can't be reached at all without changing production code
(no public entry point reaches it, or its inputs can't be set from outside), **stop and raise it with the
user**. Explain what is missing and discuss whether adding a test-only entry point is acceptable (e.g. an
internal endpoint or CLI command, disabled unless explicitly enabled by config). Don't add one silently.

## Checklist
- [ ] Entry point, inputs and terminal state identified; no production code changed (or a change agreed
      with the user)
- [ ] Layout as in "Directory structure": `evals/<agent>/agent.yaml`, tasks in `evals/<agent>/task_sets/<set>/`
- [ ] One image from `agent.yaml`, referenced by `docker_image`; per-case data in `environment/`; no
      per-task Dockerfile
- [ ] The Dockerfile goes from rarely to often changing layers and copies specific paths: from `evals/`,
      `<agent>/image/shared/` only; never `COPY . .` or `task_sets/` (tests hold reference answers)
- [ ] Real services seeded for as much of the stack as possible; any mock is a last resort and reported to
      the user; schemas created as the app's own boot does (or inferred from code, minimal, and flagged)
- [ ] Seed data looks current and real: timestamps, ids, names
- [ ] The agent's `install()` is `pass`, `run()` is thin, it imports only `harbor`, the standard library and
      agent-directory modules, and reads `self.model_name`; logs for every service land in `/logs/agent/`
- [ ] The verifier writes `/logs/verifier/reward.txt`; reference answers only in `tests/`
- [ ] Every secret is a `${VAR}` placeholder; `.env.example` lists them; `.env` and `evals/*/jobs/` are
      gitignored; `evals/README.md` says how to run and how to add a case
- [ ] `lmnr-cli eval run <agent>` passes locally from the repo root; each run shows up as a new evaluation
      in the `<agent>/<set>` group
- [ ] A positive and a negative case behave as expected; a flipped reference answer scores 0
- [ ] Trials show the agent's LLM and tool calls: the app is instrumented with Laminar and receives
      `LMNR_TRAJECTORY_PATH`, or a reconstructed `trajectory.json` is written (and the user knows what it lacks)
- [ ] Runs against the real model
- [ ] Single container per task, so `lmnr-cli eval run --remote` works without changes
- [ ] Task sets published with `lmnr-cli eval push <agent>` and pinned in `agent.yaml`; agent scripts committed
