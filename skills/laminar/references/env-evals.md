# Environment evals with Harbor

Each eval case is a Harbor **task**: an isolated container that boots the real application and its
dependencies, seeds the case's data, drives one entry point, and a verifier that scores what happened.
You are building a thin harness around code that already exists. The default is to change **no
production code**.

Run the evals with `lmnr-cli eval run`, never with `harbor run` directly. The CLI runs a pinned Harbor
with the Laminar Harbor plugin (https://laminar.sh/docs/integrations/harbor) through `uvx`, so nothing
needs installing beyond `uv` and Docker. Every trial becomes a Laminar trace and an evaluation datapoint,
so the agent's trajectory (LLM calls, tool calls) lands in Laminar next to the reward and is inspectable
with the rest of this skill (SQL, signals, debugging) instead of a pile of log files. The same folder runs
locally and on Laminar Cloud without changes, so set it up exactly as below.

## Harbor's design, and how this use differs

Harbor was built for Terminal-Bench, and later for benchmarks like it: a **general-purpose agent** with
a terminal is dropped into wildly different tasks (fix this repo, configure that server), so every task
brings its own Dockerfile, its own instruction, and its own tests.

Production evals are the opposite shape. The thing under test is **one product** (an agent, a pipeline,
an LLM-backed endpoint), and the cases are variations of the same scenario: same code, same services,
same entry point, different data. Stay compatible with Harbor's task format, but lean on that sameness:

- **one prebuilt image** for every case, not a Dockerfile per task;
- **one agent class** that drives the product's entry point, not a general terminal agent;
- **per-case data** as the main thing that varies between tasks.

## Running: `lmnr-cli eval run`

```bash
lmnr-cli eval run                       # every suite in ./evals, trials on local Docker
lmnr-cli eval run --suite support --task "refund-*" --n-attempts 3
lmnr-cli eval run --remote              # upload and run in Laminar sandboxes, return immediately
lmnr-cli eval run --remote --attach     # trials in Laminar sandboxes, orchestrated from here
```

- **Local (default)** is the iteration loop: run it after every change. It needs `uv`, Docker, and
  `LMNR_PROJECT_API_KEY` (`lmnr-cli setup` writes it to `.env`). Images listed in `evals/lmnr.yaml` are
  rebuilt with `docker build` on every run.
- **`--remote`** scales out: Laminar builds the images from the same Dockerfiles, runs every trial in its
  own sandbox, and reports into the project. No sandbox provider account, registry or local capacity.
  It needs `lmnr-cli login` (or `LMNR_PROJECT_API_KEY` in CI). The command returns once the run has
  started; results appear in the project's evaluations as trials finish.
- `--n-concurrent`, `--n-attempts`, `--model` and `--task` override `harbor.yaml` for one run.
- Each suite is one evaluation group in Laminar, so local and remote runs of a suite are charted together.
- Remote limits: one container per trial (no docker-compose tasks), resources capped per sandbox
  (`cpus`/`memory_mb` above the cap are clamped), and a run that exceeds the deployment's time limit is stopped.

If the user asks why Laminar instead of plain Harbor: the eval itself (tasks, image, agent, verifier) is
plain Harbor and works without Laminar. Laminar adds full trajectories without converting to Harbor's
trajectory format (when the app is instrumented with Laminar; see "Capturing the agent's trajectory"),
run history and side-by-side diffs shared with the team, eval traces in the same SQL
tables as production traces, signals and clusters (which closes the loop in "Turning production data into
cases"), and managed execution with `--remote`. When bare Harbor is enough (no production tracing in
Laminar, rare runs, local results are fine), say so; the folder adopts Laminar later without changes.

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
4. **Evals live next to the code** and the image is built from the same source tree and revision, not
   from vendored copies (see Staleness).
5. **Turn the run into artifacts, then verify the artifacts.** The forward pass should leave its observable
   result in files (a `result.json`, an exported row, a transcript). A verifier is arbitrary code: a
   deterministic comparison, a rubric, or LLM-powered grading over those artifacts. Pick what fits the
   case; a discrete outcome (`event` / `no_event`, `pass` / `fail`) is simply the easiest to start with.
   - **`shared` verifier** (the default): runs in the agent's container and can read anything the run left
     behind, e.g. `/logs/agent/result.json`.
   - **`separate` verifier**: runs in a fresh container and sees only what the task declares. List files in
     `artifacts = [...]` in `task.toml`, and use `[[verifier.collect]]` commands to snapshot runtime state
     (database contents, in-memory counters) into files before the environment is torn down.
     Prefer this when the verifier needs its own dependencies, or must not see the app's internals.

## Data isolation

Isolated environments let each case start from exactly the data it needs. Decide per dataset which of
two kinds it is:

- **Shared baseline data**: the same for every case (reference tables, a default org/user/project, config
  rows, an LLM profile). Seed it once in the image build, or in a shared seed script that runs at startup
  in every container. Each container still gets its own copy, so isolation holds.
- **Per-case data**: what makes the case reproduce a specific behaviour from production (the conversation,
  the spans, the records that triggered a failure). This is usually the bulk of the case. It lives in the
  task's `environment/` directory and is seeded at startup by the same shared seed code for every case.

Most real setups have both. Keep per-case data in one format your seed script reads (`case.json`, a SQL
file, a directory of fixtures), so the seed script is shared and only the data differs.

## Directory structure

This layout is what `lmnr-cli eval run` expects. Each **suite** is a directory directly inside `evals/`
holding a `harbor.yaml`; most projects have one suite. Files marked *(optional)* depend on your setup.

```
evals/
  README.md                       # how to run, how to add a case
  lmnr.yaml                       # the images tasks use, and how to build them
  image/
    Dockerfile                    # the one image: services, app artifact, shared scripts, baseline data
    shared/                       # copied into the image by the Dockerfile
      start_stack.sh              # boot dependencies + the app, wait for readiness
      schema/                     # (optional) schema setup when the app has no migrations of its own
      seed_baseline.*             # (optional) data every case shares
      seed_case.*                 # load this case's data from the uploaded environment/ files
      run_case.*                  # drive the entry point; write result files
      export_trajectory.*         # (optional) rebuild the agent's trajectory when the app isn't instrumented
  <suite>/                        # e.g. support-agent
    harbor.yaml                   # Harbor JobConfig: agents, datasets, concurrency, ${VAR} secrets
    agent/
      __init__.py
      agent.py                    # BaseInstalledAgent subclass: starts the stack, runs the driver
    tasks/
      <case-name>/
        task.toml                 # [environment] docker_image = "<image name from lmnr.yaml>"
        instruction.md
        environment/              # no Dockerfile: uploaded into the container's workdir at start
          case.json               # this case's data; any format seed_case reads
        tests/
          test.sh                 # the verifier entry point (any language, any grading method)
          expected.json           # (optional) reference answer, if the verifier compares against one
```

`evals/lmnr.yaml` names each image and how to build it. Paths are relative to `evals/`; `context` is the
`docker build` context (default: the Dockerfile's directory). Use the repo root so the image can copy the
app source and migrations from the live tree, and list under `include` exactly the paths the Dockerfile
copies, relative to the context:

```yaml
images:
  myapp-evals:
    dockerfile: image/Dockerfile
    context: ..
    include: [app/, migrations/, pyproject.toml, uv.lock]
```

Only `include` paths leave the machine for a `--remote` run; without it the CLI uploads the whole context.
A change anywhere in the uploaded files rebuilds the image from scratch, so keep the list to what the
Dockerfile reads and never include `evals/` (Harbor writes run output there). Local runs build from the same
paths, so a missing entry fails `docker build` with "not found" locally first: add the path and rerun.
`.dockerignore` still applies inside included directories.

`<suite>/harbor.yaml` is a plain Harbor JobConfig. Relative paths in it resolve from the suite directory,
and the suite directory is on `PYTHONPATH`, so the agent's `import_path` is relative to it:

```yaml
agents:
  - import_path: agent.agent:MyAppAgent
    model_name: anthropic/claude-sonnet-5-5   # read as self.model_name; overridable with --model
    env:
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
datasets:
  - path: tasks
n_concurrent_trials: 4                         # what the local machine can run; raise with --n-concurrent on --remote
```

- Keep what differs per case small (data, instruction, reference answer) and everything else shared, so a
  new case is a directory copy plus data and never needs harness changes.
- Keep reference answers in `tests/`, not in `environment/`, so neither the agent nor the app can read them.
- Gitignore Harbor's job output (`evals/<suite>/jobs/`). The CLI never uploads it, nor `.env*`, `.git`,
  `node_modules`, virtualenvs or caches.

## Secrets and env vars

Declare every secret and setting the run needs as a Harbor `${VAR}` placeholder, never as a literal and
never as a `harbor run` flag:

- agent env in `harbor.yaml` (`agents[].env`, as above);
- `[verifier.env]` in `task.toml` for an LLM-graded verifier, `[environment.env]` for the container.

`${VAR:-default}` makes one optional. `lmnr-cli eval run` reads each referenced name from the shell, then
from `.env.local` / `.env` in the directory it runs from, and fails before anything starts if a required
one is missing, so the user only has to fill in `.env`. For `--remote` the values travel with the run as
env vars; files are never uploaded with secrets in them. Add `.env` to `.gitignore`, and write a
`.env.example` listing the names.

## Use one prebuilt image

Set `docker_image` in each task's `task.toml` to an image name from `lmnr.yaml`, and leave the Dockerfile
out of `environment/`:

```toml
[environment]
docker_image = "myapp-evals"
workdir = "/case"          # where environment/ gets uploaded; defaults to the image's WORKDIR
cpus = 2
memory_mb = 4096
```

- **Speed is the main reason.** Every trial starts from the same built image instead of building its own.
  `lmnr-cli` builds it once per run (locally with Docker; for `--remote` Laminar builds it from the same
  Dockerfile and caches it by content, so an unchanged image costs nothing). No registry, no tags to bump.
- **Per-case files still arrive.** When `docker_image` is set and `environment/` has no `Dockerfile` or
  `docker-compose.yaml`, Harbor uploads the contents of `environment/` into the workdir after the container
  starts. That is how `case.json` reaches a shared image. Putting a Dockerfile back into `environment/`
  turns this upload off.
- A per-task Dockerfile is the exception, for when a case genuinely needs a different system setup that
  can't be expressed as data. Its build context is the task's `environment/` directory, and nothing is
  uploaded at start, so the Dockerfile must `COPY` the case data itself. Prefer `FROM` a public base plus
  the difference; a task can't `FROM` an image that only exists in `lmnr.yaml`.
- An image name that isn't in `lmnr.yaml` is pulled from its registry, so public images work as-is.

## The agent: subclass `BaseInstalledAgent`

Harbor has two agent interfaces (https://docs.harborframework.com/agents/custom-agents; check your
installed version, the API moves):

- **`BaseInstalledAgent`** (`harbor.agents.installed.base`): the agent runs **inside** the task
  environment. This is Harbor's general recommendation and the right fit here: the product under test
  already lives in the container, so the agent is just the code that starts it and drives its entry point.
- **`BaseAgent`** (`harbor.agents.base`): the agent loop runs on the **host** and only reaches the
  container through `environment.exec(...)`. Use it only when the agent *must* run outside the task
  environment (e.g. it is a hosted service you can't put in the container). That is rarely the case for
  a production system you're evaluating.

Shape of the installed agent:

- `name()` (static), and `run(instruction, environment, context)` decorated with `@with_prompt_template`.
- `install(environment)`: one-time setup that could be baked into the image. Users can skip it with
  `--ak skip_install=true`, and with a prebuilt image it is usually empty, since everything is already installed.
- `configure()`: per-trial setup (write credentials or config, upload a runner script). Runs even when
  install is skipped.
- Run commands with `self.exec_as_agent(environment, command=..., env=..., cwd=..., timeout_sec=...)`, or
  `exec_as_root` for things that need root. The agent's `env` from `harbor.yaml` arrives in
  `self._extra_env`; pass what the app needs explicitly with `env={...}`. Use `self.model_name` for the
  model so `--model` works. A non-zero exit raises, so have the driver exit 0 whenever it
  produced an outcome (including `failed` / `timeout`) and non-zero only for harness errors.
- `populate_context_post_run(context)`: runs after logs are synced to the host; parse `result.json` and put
  the result into the context.
- `run()` stays thin: start the stack, seed baseline + case data, run the driver, done.
- Reference it from `harbor.yaml` as `import_path: agent.agent:ClassName` (relative to the suite).
- Import only `harbor`, the standard library and modules inside the suite directory: the class itself runs
  in the Harbor process (on the user's machine or Laminar's orchestrator), not in the container. Anything
  else belongs in the image.

## Capturing the agent's trajectory

The trial trace is only as rich as what the product under test reports. Without a trajectory, the trace
has a single span for the agent phase: you see the reward, but not the LLM calls and tool calls that led
to it, which is what you need to debug a failing case. Check which of these applies, in this order.

### The app is instrumented with Laminar (preferred)

If the product calls `Laminar.initialize()` (Python/TS) or `registerTelemetry()` (AI SDK v7+), its spans
become the trial's trajectory: every LLM call with tokens and cost, and tool calls as `TOOL` spans when
tools are covered by framework auto-instrumentation or wrapped with `observe` / `@observe`. Nothing to
convert.

During an eval run, `lmnr-cli` / the plugin set env vars that switch the SDK from sending spans to Laminar
to dumping them as the trial's trajectory file, which the plugin then attaches to the trial trace. The app
needs no Laminar API key and no network access to Laminar for this. Make sure that:
- the app process actually receives those vars: <env var names>. The start script launches the app with
  `env -i`, so pass them through explicitly;
- you don't also write a reconstructed trajectory (below), or the agent's steps show up twice.

If the app isn't instrumented, adding Laminar is a production change: propose it to the user rather than
doing it inside the eval work. It is small, and production traces benefit too (see
[instrumentation-python.md](instrumentation-python.md), [instrumentation-typescript.md](instrumentation-typescript.md)).
Some apps can't take it now: another language, another tracing setup, or not this change.

### Not instrumented: reconstruct it from the app's own history

Most products already persist the conversation: to show chat history to the user, for audit, or for the
agent's own memory. Read the code that renders or replays that history; it tells you where the history
lives and how tool calls are stored. Common places:
- message/thread tables behind the chat UI (often with structured parts for tool calls and results);
- agent run/step/event tables, audit logs;
- framework state: checkpointers (LangGraph), session stores (agent SDKs), JSONL transcript files;
- the entry point's response, if it returns the full message list.

After the driver finishes, including on `failed` / `timeout` (partial trajectories are the most useful),
run a shared exporter (`image/shared/export_trajectory.*`) inside the container. It reads this case's
history from the store and writes `/logs/agent/trajectory.json` in Harbor's ATIF format, which the plugin
replays as `LLM` and `TOOL` spans:
- each message becomes a `step` with `source` (`system` / `user` / `agent`), `message`, and `timestamp`
  from the row's creation time; `model_name` and `reasoning_content` if the app stores them;
- history seeded from `case.json` (what happened before the run) gets `is_copied_context: true`, so it is
  shown but not mistaken for the agent's work;
- tool calls go into the agent step's `tool_calls` (`tool_call_id`, `function_name`, `arguments`), and their
  results into `observation.results` with a matching `source_call_id`. If the store has no call ids, generate
  them and pair calls with results by order. Infer a tool call from side effects (rows it wrote) only when
  it is unambiguous; otherwise leave it as message text;
- top-level: `schema_version`, `session_id`, and `agent` (`name`, `version`, `model_name`).

Be explicit about what's missing. Stored history usually lacks token counts, cost, the system prompt, the
exact prompt of each LLM call (one stored turn may be several calls), retries and per-call latency. Leave
`metrics` unset rather than estimating, and say in `notes` where the trajectory came from (e.g.
`"reconstructed from chat_messages"`). In `populate_context_post_run`, validate the file with
`harbor.models.trajectories.Trajectory` and log (don't fail the trial) if it doesn't parse. Tell the user
the trajectory is reconstructed and which fields it lacks.

If the app keeps no history at all, the trial has only `result.json` and the service logs. Say so and
suggest instrumentation.

## How Harbor actually works (what shapes the design)

- Harbor **overrides the container's main process**. Your Dockerfile's `CMD`/`ENTRYPOINT` (and the
  entrypoint of base images like `postgres`) will not run. The start script must launch every service
  itself as a background process (`nohup ... &`, output to `/logs/agent/*.log`) and wait for readiness.
- **No ports are published to the host.** Every call to the app must come from a process inside the
  container, which is another reason to use an installed agent.
- `/logs/agent/` and `/logs/verifier/` are shared between container and host. Write `result.json` and
  every service log under `/logs/agent/`. A `shared` verifier reads them in place;
  a `separate` one gets only declared `artifacts` (see Principle 5). Either way the verifier writes its
  reward (e.g. `1` or `0`) to `/logs/verifier/reward.txt`.
- Values reach the agent through `agents[].env` placeholders in `harbor.yaml` (see "Secrets and env vars")
  and are available as `self._extra_env`; pass them explicitly with `env={...}` to the commands that need them.
- Some services refuse to run as root (Postgres). Use `gosu`/`su` to a service user, and make sure the
  log directory is writable by that user (`chmod 777 /logs/agent`).

## Procedure

### 1. Find the entry point and the observable outcome
Read the production code for the path you want to evaluate. Identify:
- the **entry point** you can invoke without modification (route, CLI, queue consumer, function);
- the **inputs it reads** (rows, spans, files, messages) and where they live;
- the **terminal state** it produces (a response body, a status row, an emitted record) and how to read it;
- the **external calls** it makes (LLM, HTTP) and how they are configured (env, config row, profile).

If the existing tests seed a fixture for this path, copy their seeding shape. They are the best statement of
which columns/fields are actually required.

### 2. Build the environment (see Services)
One image for all cases. Pin service versions to what production/compose uses.

### 3. Seed through the real interface where it is cheap, directly where it is not
- Use the app's own API for anything that involves encryption, validation or derived fields (e.g. creating
  an LLM profile with secrets: the server encrypts them; don't hand-encrypt).
- Insert directly into the database for things the API gates behind plan limits or auth you don't have, and
  for bulk fixtures (traces, messages, records). Mirror the columns the app actually reads; omit the ones
  with defaults.
- Hash credentials exactly the way the app does (read its auth code).
- Make the data look real (see "Seed data the agent can't tell is a test").

### 4. Trigger, then wait for a terminal state
- **Synchronous entry point** (an HTTP call or function that returns the result): call it with a timeout
  and record the response. That is the whole step.
- **Asynchronous entry point** (enqueue a job, the app writes a status later): don't sleep. Poll the status
  the app writes until it is terminal, with a hard timeout shorter than the agent's timeout.

Map to an outcome the verifier can grade (a discrete value, plus `failed` and `timeout` so those are never
mistaken for a model verdict). Write the full detail (status, error message, payload, ids, duration) to
`/logs/agent/result.json` and print it.

### 5. Verify the harness, not just the case
- Run a **positive** and a **negative** case that share the same prompt/config and differ only in input.
- Flip the reference answer once and confirm the reward goes to 0. A harness that can't fail is broken.

## Seed data the agent can't tell is a test

Capable models notice when they're being evaluated, and the more general the agent, the more it
looks around and the likelier it is to notice. An agent that suspects a test may behave differently
from how it behaves in production, and then the eval measures the wrong thing. Fixtures are where this leaks most.

- **Timestamps.** This takes judgement. Production data exported weeks ago, replayed today, reads as
  stale: "the last activity on this account was 5 months ago" is a hint the agent is looking at a fixture.
  - Default to **relative** timestamps: store offsets (or shift on load) so the most recent event is near
    "now" and the intervals between events are preserved exactly.
  - Keep **absolute** timestamps when the case depends on a specific date: end-of-month logic, a timezone or
    DST boundary, an expiry, a schedule. Then make the rest of the data consistent with that date.
  - Don't forget dates that aren't in timestamp columns: years in free text, filenames, version strings,
    "as of" lines in documents, ids that embed time (UUIDv7, ULID, snowflakes).
- **Ids.** Patterned ids give the game away: `12000000-0000-4000-8000-0123456789ab`, `user_1`, `test-org`.
  Generate ids the way the app does (random UUIDv4, time-ordered v7 consistent with the timestamps,
  prefixed ids like `cus_...`) and keep references between records consistent. Deterministic per case is
  fine (seed the generator with the case name); isolation already prevents collisions.
- **Names and content.** No `John Doe`, `test@example.com`, `Acme`, `lorem ipsum`, `foo`. Scrubbed PII
  should be replaced with plausible values, not placeholders.
- **The harness itself.** If the agent under test can see the filesystem, process list or environment,
  keep harness artifacts out of sight or neutrally named: no `/eval`, `EVAL_MODE=1`, `fixture` or
  `test_case` in anything it can read.

## Services

Goal: the container contains everything the entry point touches, started the way production starts it.

**Today: boot the real dependencies in the container.** Single-container, no docker-in-docker.
- *Databases.* Start them as local background processes with the engine's own bootstrap (`initdb`,
  `clickhouse server`, `mysqld`, `redis-server`...). Use trust/no-password auth locally and a throwaway
  password where the app's connection string requires one.
- *Getting the binaries.* Use the simplest source that gives you the version production runs: the
  distro package manager (`apt-get install postgresql-16`), the vendor's apt repo, or the vendor's release
  tarball. When a specific version is hard to get that way, copy it from the official image with a
  multi-stage `COPY --from=<official image>:<version>` (copy the shared libs it needs too).
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
- *The app itself.* Run the production artifact: the real package imported/started by the start script,
  or a compiled binary. For a library or callable function, the driver script imports and calls it
  directly in-process; the service boot still happens first.
- *Readiness.* Poll the health endpoint (and any readiness endpoint) with a timeout; on failure `tail` the app
  log into stderr so the failure is visible in the job output.
- *Env.* Start the app with `env -i` and an explicit minimal env so the host's shell never leaks in.

**Soon: mocked services via `lmnr-cli`.** *(skeleton, to be filled in when the capability ships)*
- What it will provide: <list of supported services / protocol-compatible mocks>.
- How to declare it: <config format, where it lives in `tasks/<case>/environment/`>.
- How to start/stop and wait for readiness inside the container: <commands>.
- How to seed fixtures and read back calls made to the mock: <commands>.
- When to prefer it over a real service: <heavy/licensed/third-party services you can't run locally>.
Until then, keep external edges (SaaS) behind the app's existing base-URL/config.

## LLM handling
- **The LLM is real.** Evals exist to measure how the product behaves with a model; a canned response only
  tests the harness. Never substitute a fake or recorded model for the one under test.
- Route the app's model calls through its normal configuration (profile row, base URL env), pointing at the
  endpoint and key from the agent's `${VAR}` env. Don't patch the client.
- A failing or unreachable LLM endpoint often makes the app **retry rather than fail**. Your timeout is
  what ends the run, so report `timeout` distinctly from `failed`.

## Staleness
- `lmnr-cli eval run` rebuilds the `lmnr.yaml` images on every run, so a change to `image/shared/` or to the
  app is picked up next run. Don't add a separate build script, and don't run `harbor run` by hand: it would
  use whatever image happens to be tagged locally.
- Build from the **live source paths** (the app source, its migrations), not from a vendored copy, so evals
  can't drift from the schema they run against. The app artifact and the migrations in one image come from
  the same revision by construction.
- Docker layer caching can hide staleness locally. If something looks stale after a rebuild, check the file
  inside a running container, or rebuild with `docker build --no-cache`.

## Compiled artifacts
If the product ships as a compiled binary, build it in a builder stage of the same Dockerfile, not on the
host: a macOS host binary is the wrong format, and cross-compiling tends to fail on native dependencies.
Use a builder base whose libc is not newer than the runtime stage's, and install the shared libs the binary
links at runtime. Check what it reads relative to its working directory (templates, data files); a server
missing them can start but silently never bind its port.

## Harbor quirks and findings
- **Stale image** (above): the most common source of "my fix did nothing".
- **Secret scrubbing:** Harbor redacts agent env values from its output files. A very short value
  (a one-character placeholder like `x`) turns every occurrence of that character in the job output into
  `[REDACTED]` (`ma[REDACTED]_timeout_sec`), so keep placeholder values long and distinctive.
- **Job output location:** `evals/<suite>/jobs/<timestamp>/` for local and attached runs. Each trial
  has `agent/` (your logs, `result.json`) and `verifier/` (`reward.txt`, `test-stdout.txt`); `exception.txt`
  holds the agent traceback.
- Services started with `nohup ... &` inside an exec survive after the exec returns. Redirect their output, or
  the exec call may not return.
- Some servers need a password in the connection string even when the database uses trust auth; use a dummy.
- Keep exec timeouts below the task's `[agent] timeout_sec`, and the driver's own timeout below both.
- **Resources:** a full stack per case is heavy. Set `cpus`/`memory_mb` in `task.toml`. Locally every trial
  shares the host, so keep `n_concurrent_trials` in `harbor.yaml` to what the host can run. With `--remote`
  each trial gets its own sandbox; raise concurrency for that run with `--n-concurrent`.

## Turning production data into cases
The loop this setup exists for:
1. Select production traces/records with the signal of interest (a detected failure, a flagged event) and a
   few that were correctly *not* flagged.
2. Export the minimal input for each into the case data (`case.json`: messages, tool calls, spans, records).
   Strip or replace PII and secrets with plausible values; keep structure and ordering; rebase timestamps.
3. Set the reference answer from the production label after a human check. Keep positive and negative cases
   paired against the same prompt/config.
4. Copy a task directory, drop in the new case data and reference answer, run `lmnr-cli eval run`, commit.

### When the harness doesn't fit
Adding a case should not need harness changes. If one does, first try to generalise the shared seed/driver
code so every case benefits. If the behaviour can't be reached at all without changing production code
(no public entry point reaches it, or its inputs can't be set from outside), **stop and raise it with the
user**. Explain what is missing and discuss whether adding a test-only entry point is acceptable (e.g. an
internal endpoint or CLI command, disabled unless explicitly enabled by config). Don't add one silently.

## Checklist
- [ ] Layout as in "Directory structure": `evals/lmnr.yaml`, `evals/<suite>/harbor.yaml`, tasks under the suite
- [ ] `lmnr-cli eval run` passes locally; trials show up in Laminar as one evaluation per suite
- [ ] Entry point and terminal state identified; no production code changed (or a change agreed with the user)
- [ ] Real services seeded for as much of the stack as possible; any mock is a last resort and reported to
      the user
- [ ] Schemas created as the app's own boot does (or inferred from code, minimal, and flagged)
- [ ] One image from `lmnr.yaml`, referenced by `docker_image`; per-case data in `environment/`, no per-task Dockerfile
- [ ] The image's `include` lists exactly the paths its Dockerfile copies, and not `evals/`
- [ ] Reference answers only in `tests/`, never in the image or `environment/`
- [ ] Every secret is a `${VAR}` placeholder; `.env.example` lists them; `.env` is gitignored
- [ ] The agent imports only `harbor`, the standard library and suite modules, and reads `self.model_name`
- [ ] Seed data looks current and real: timestamps, ids, names
- [ ] A positive and a negative case behave as expected; a flipped reference answer scores 0
- [ ] Logs for every service land in `/logs/agent/`
- [ ] Trials show the agent's LLM and tool calls: the app is instrumented with Laminar, or a reconstructed
      `trajectory.json` is written (and the user knows what it lacks)
- [ ] Runs against the real model
- [ ] Single container per task, so `lmnr-cli eval run --remote` works without changes
