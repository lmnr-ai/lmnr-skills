# Environment evals with Harbor

Each eval case is a Harbor **task**: an isolated container that boots the real application and its
dependencies, seeds the case's data, drives one entry point, and a verifier that scores what happened.
You are building a thin harness around code that already exists. The default is to change **no
production code**.

A dataset's harness is one Python file, `agent/eval.py`: an `EvalRunner` subclass (from `lmnr.evals`, in
Laminar's Python SDK) that names the image, sets up each case, drives the entry point and grades the
result. `lmnr-cli` turns it into Harbor's agent, verifier and job config, and writes each task's
boilerplate files, so a case is just its data. Start every new dataset with `eval.py`; a hand-written Harbor
agent and `tests/test.sh` still work ("Without eval.py") and are for datasets that already use them.

Run the evals with `lmnr-cli eval run`, not with `harbor run` directly. The CLI builds the images, then
runs a pinned Harbor (0.25.0) with the Laminar Harbor plugin (https://laminar.sh/docs/integrations/harbor)
through `uvx`, so nothing needs installing beyond `uv` and Docker. Every trial becomes a Laminar trace and
an evaluation datapoint, so the agent's trajectory (LLM calls, tool calls) lands in Laminar next to the
reward and is inspectable with the rest of this skill (SQL, signals, debugging) instead of a pile of log
files. The same folder runs locally and on Laminar Cloud without changes, so set it up exactly as below.

## Harbor's design, and how this use differs

Harbor was built for Terminal-Bench, and later for benchmarks like it: a **general-purpose agent** with
a terminal is dropped into wildly different tasks (fix this repo, configure that server), so every task
brings its own Dockerfile, its own instruction, and its own tests.

Production evals are the opposite shape. The thing under test is **one product** (an agent, a pipeline,
an LLM-backed endpoint), and the cases are variations of the same scenario: same code, same services,
same entry point, different data. Stay compatible with Harbor's task format, but lean on that sameness:

- **one prebuilt image** for every case, not a Dockerfile per task;
- **one runner** (`agent/eval.py`) that drives the product's entry point and grades every case, not a
  general terminal agent with per-task tests;
- **per-case data** as the main thing that varies between tasks.

## Running: `lmnr-cli eval run`

```bash
lmnr-cli eval run                       # every dataset in ./evals, trials on local Docker
lmnr-cli eval run support --task "refund-*" --n-attempts 3   # one dataset, some of its tasks
lmnr-cli eval run --task refund-double-charge   # just one case
lmnr-cli eval run --remote              # run in Laminar, print the run's log until it ends
```

Run from the repo root: `.env` / `.env.local` are read from the current directory (`--dir` points at
another evals folder; default `evals`). A dataset is named by its folder or by its `[dataset] name`. Both
modes need Docker on the machine that runs `lmnr-cli`, which builds the dataset's images there. For an
`eval.py` dataset, `eval run` first loads `eval.py` (with `uvx`, Python 3.13) and stops with its error
(exit `16`) if it doesn't import or isn't valid, then writes the task files a case is missing ("Adding a
case"). `eval run` runs the tasks in `tasks/` exactly as they are, local edits included, and never pulls: run
`lmnr-cli eval dataset pull` first in a fresh clone or after a `git pull` that moved pins. It warns
when pinned tasks are missing or differ from their pin (edited, or not pulled yet), and fails when a dataset pins tasks but has none
on disk (see "Eval datasets").

- **Local (default)** is the iteration loop: run it after every change. It needs `uv`, Docker, and
  `LMNR_PROJECT_API_KEY` (`lmnr-cli setup` writes it to `.env`).
- **`--remote`** scales out, and leaves everything needed to debug a run in Laminar, where the whole team
  can read it. `lmnr-cli` builds or pulls every image the selected datasets need with local Docker for
  `linux/amd64` (the dataset's images, per-task `environment/Dockerfile`s, public `docker_image`s), uploads
  only the layers Laminar doesn't have, and starts the run in a Laminar sandbox; every trial gets its own
  sandbox. Tasks already pushed to Laminar are not uploaded with the run (the run downloads them); only
  unpushed local edits are. It needs `lmnr-cli login` (or `LMNR_PROJECT_API_KEY` in CI), then prints the
  run's log (Harbor's output) until the run ends and exits with its outcome. Ctrl-C stops following; the run
  goes on. Results appear in the project's evaluations as trials finish. On the evaluation page each trial
  has its trace and a **Files** view with up to four folders: `task/` (the exact task version it ran),
  `agent/` (the dataset's `agent/` folder), `runner/` (shared modules its runner imported, see "Shared
  runners") and `logs/` (Harbor's output: your `/logs/agent/` files, the verifier output, `exception.txt`;
  all but `task/` start collapsed), so debug a remote failure there. Local
  runs upload nothing, so their trials show only `task/`, and only for task versions that were pushed. The
  evaluation's header links to the eval dataset the run used, once that dataset has been pushed; in its
  table, a task that ran a pushed version is a chip that opens that version's files, and so is the task
  name above the Files tree.
- **Which to use:** local for a tight loop on one machine; `--remote` for many cases at once, for runs
  someone else will look at, and when the machine can't run the stacks.
- `--n-concurrent`, `--n-attempts`, `--model` and `--task <glob>` override `eval.py` / `harbor.yaml` for
  one run; `--env KEY[=VALUE]` (repeatable) adds an agent env var, and a bare `KEY` is read from your
  environment.
- `--task` is Harbor's: on `eval run` it matches task **directory** names (on `eval dataset` it matches
  task names). Only the tasks a run includes (under a `datasets` path or listed in `tasks`, after `--task`)
  get their images built and their `${VAR}`s required.
- Every run creates a new evaluation in an evaluation group, and runs in one group are charted together.
  The group is the dataset's `[dataset] name` unless `--group <name>` names another, so local and remote
  runs of a dataset line up by default. Use `--group` to keep a series apart (a prompt or model experiment):
  `lmnr-cli eval run support --group "support (prompt v2)"`. It applies to one dataset, so name the dataset
  when the evals folder has several (another dataset gets its own command and group). Each evaluation still
  records its dataset, and each trial the task (`[task] name`) and version (content hash) it ran.
- `--json` prints `{mode, datasets, failed, url}`, or for `--remote`, once the run ends,
  `{mode: "remote", uploadId, sandboxId, datasets, url, evaluations, exitCode}` (`evaluations` lists each
  dataset's evaluation link). Exit codes: `16` invalid evals folder (including an `eval.py` that doesn't
  load) or archive too large, `17` `uv`/`docker`
  missing, `18` a dataset's `harbor run` failed (the other datasets still run), `19` an unset `${VAR}`, `20`
  a Docker build/save/pull failed or an image has the wrong platform, `21` a `--remote` run failed before
  its datasets ran or its sandbox was lost; `22` comes from `eval dataset pull` ("Eval datasets").

**Remote limits and image requirements:**
- One container per trial (no docker-compose tasks); `cpus`/`memory_mb` above the sandbox cap are clamped;
  a run that exceeds the deployment's time limit is stopped.
- `--remote` packs the run's dataset folders minus the tasks Laminar already has (max 1 GiB). Symlinks are
  skipped, so are directories named `jobs`, `.git`, `node_modules`, virtualenvs and caches; `.env*` files
  are never packed. Keep large fixtures out of `agent/`; per-case data belongs in pushed tasks.
- Images need `tar`, `gzip` and `bash` (and `su` when the task runs as a non-root user). They are built for
  `linux/amd64`, emulated (slowly) on ARM Macs; architecture-specific downloads should use `ARG TARGETARCH`.
- Remote runs drop `LMNR_*` vars from the agent env. Against self-hosted Laminar, set `LMNR_GRPC_PORT`
  (shell or `.env`) or the trial traces don't arrive. A self-hosted Laminar stores eval datasets only with
  S3 configured (`S3_DATASETS_BUCKET` plus AWS credentials); without it, `eval dataset pull` / `push` fail
  with `storage_not_configured`, and `--remote` runs upload every task.
- Soft guidance: sandboxes per project are capped (50 by default), so a higher `--n-concurrent` queues and
  retries rather than running more at once. Harbor's `[environment] build_timeout_sec` (default 600)
  bounds image preparation plus sandbox start on `--remote`; raise it for very large images.

If the user asks why Laminar instead of plain Harbor: Harbor runs the eval and the task format is
Harbor's, but an `eval.py` dataset needs `lmnr-cli` and the `lmnr` package: `lmnr-cli` builds the Harbor
job config from `eval.py`, and the agent and verifier Harbor runs are adapters in `lmnr`, so bare
`harbor run` on the folder doesn't work. A dataset written without `eval.py` ("Without eval.py") is plain
Harbor and does run with bare `harbor run` (after building each `harbor.yaml` image with
`docker build -t <name>`, since Harbor ignores the `images` key). Laminar adds full trajectories without
converting to Harbor's format (when the app is instrumented; see "Capturing the agent's trajectory"), run
history and side-by-side diffs shared with the team, eval traces in the same SQL tables as production
traces, signals and clusters (which close the loop in "Turning production data into cases"), and managed
execution with `--remote`, and task folders stored and versioned in Laminar ("Eval datasets") instead of
in git. When bare Harbor is enough (no production tracing in Laminar, rare runs, local
results are fine), say so and write the dataset without `eval.py`; that folder adopts Laminar later without
changes.

## Principles

1. **Production fidelity over convenience.** Run the real code and real services (database, queue,
   cache, object store) for as much of the stack as possible. Seed them; don't mock them. For a
   third-party SaaS API, use a simulator of it where one exists ("Mirrors"). Mock only what you truly
   cannot run (private internal services, a SaaS with no simulator), as a last resort, at the network
   edge (base URL / profile config), never by editing the code path. **Tell the user about every mock**
   and why it was needed; a mock is a gap in what the eval measures.
2. **Zero production changes.** Find an existing public entry point (an HTTP route, CLI command, queue
   message, exported function) and drive that. A harness-only code path tests the harness. If no existing
   entry point reaches the behaviour, stop and raise it with the user (see "When the harness doesn't fit").
3. **One case = one isolated environment.** Every container boots its own stack with its own data, so
   cases run in parallel with no shared state and no ordering. Never share a database between cases.
4. **Evals live next to the code** and the image is built from the same source tree and revision, not
   from vendored copies (see "Staleness").
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
2. **Lay out `evals/`**: `<dataset>/agent/eval.py`, `<dataset>/tasks/dataset.toml` with its
   `[dataset] name`, and one case under `<dataset>/tasks/` with `environment/case.json` and its reference
   answer in `tests/expected.json`. `lmnr-cli eval task new <dataset>/tasks/<case>` writes the rest
   ("Directory structure", "Use one prebuilt image"). If the project already has eval datasets in Laminar,
   run `lmnr-cli eval dataset pull` first.
3. **Declare the image and the services** in `eval.py`: an `Image` with the app's dependencies and live
   source ("Building the image"), and `services = [Postgres("16", migrations=...), ...]` pinned to what
   production/compose uses. The runner installs them into the image, and in every trial starts,
   migrates and seeds them before `setup()`; `setup()` starts the app itself with
   `ctx.container.service` ("Services").
4. **Seed through the real interface where it is cheap, directly where it is not** ("Data isolation",
   "Seed data the agent can't tell is a test"): schema through the service's migrations, the rest in
   `setup()` (usually `ctx.container.run_file("seed.py")`, a script next to `eval.py` that runs in the
   container and loads `case.json`, or the service's helpers: `self.service(Postgres).insert(ctx, ...)`):
   - Use the app's own API for anything that involves encryption, validation or derived fields (e.g.
     creating an LLM profile with secrets: the server encrypts them; don't hand-encrypt).
   - Insert directly into the database for things the API gates behind plan limits or auth you don't have,
     and for bulk fixtures (traces, messages, records). Mirror the columns the app actually reads; omit the
     ones with defaults. Hash credentials exactly the way the app does (read its auth code).
5. **Write `run()`: trigger, then wait for a terminal state** ("The runner").
   - **Synchronous entry point** (an HTTP call or function that returns the result): call it with a timeout
     and record the response.
   - **Asynchronous entry point** (enqueue a job, the app writes a status later): don't sleep. Poll the
     status the app writes until it is terminal (`ctx.container.wait_for`), with a hard timeout shorter than the
     agent's timeout.
   - Map to an outcome `verify()` can grade (a discrete value, plus `failed` and `timeout` so those are
     never mistaken for a model verdict), and record it with the full detail (status, error message,
     payload, ids, duration) via `ctx.set_result(...)`.
6. **Write `verify()`** ("The verifier").
7. **Declare secrets and document the folder**: `secrets` / `env` in `eval.py`, `.env.example`,
   `.gitignore` for `.env` and `evals/*/jobs/`, and `evals/README.md` saying how to run, how to add a case
   and how to push it ("Secrets and env vars").
8. **Run it locally** from the repo root with `lmnr-cli eval run` and fix until it passes. Then verify the
   harness, not just the case: run a **positive** and a **negative** case that share the same
   prompt/config and differ only in input, and flip the reference answer once to confirm the reward goes
   to 0. A harness that can't fail is broken.
9. **Check the trajectory** in the trial's trace: LLM and tool calls should be visible ("Capturing the
   agent's trajectory"), and `verify()`'s commands and LLM calls appear under the verifier span.
10. **Push the tasks**: `lmnr-cli eval dataset push`, then commit `tasks/dataset.toml` (the rest of
    `tasks/` is git-ignored) with the rest of `evals/` ("Eval datasets"). Tell the user the case data is now stored in
    their Laminar project.
11. **Optionally run `--remote`** once to confirm the folder works in Laminar sandboxes ("Running").

## Data isolation

Isolated environments let each case start from exactly the data it needs. Every container starts its
own services with empty data, so isolation holds whatever you seed. Most datasets have **shared baseline
data** (the same for every case: schema, reference tables, a default org/user/project, an LLM profile)
and **per-case data** (what makes the case reproduce a specific behaviour from production: the
conversation, the spans, the records that triggered a failure; usually the bulk of the case). Put each
kind where it belongs:

| Data | Where | How |
| --- | --- | --- |
| Schema, reference tables, anything static | the service's migrations, in every trial (or in the image with `migrate_at="build"`) | `Postgres("16", migrations="../../../migrations")` |
| Baseline that needs the running app (rows its API creates with encryption or validation, "now"-relative timestamps) | one shared seed script, in every trial | `setup()` runs `ctx.container.run_file("seed.py")` |
| Per-case data the app reads | the task's `environment/` (uploaded into the workdir before `setup()`), loaded by the **same** shared seed | `/case/case.json` in the container, `ctx.case.file(...)` on the host |
| Per-case data the agent must not see (a mirror's seed, a dump it would recognise) | the task's `tests/fixtures/` (not in the container while the agent runs) | `seed=case_fixture("...")` on a service, `ctx.case.fixture(...)` in `setup()` |
| Per-case logic the shared seed can't express | the task's `environment/seed.py` / `seed.sh` | the default `setup()` runs it; if many cases need one, generalise the shared seed instead |
| A shared staging or external database | outside the container | only for read-only cases or `n_concurrent_trials: 1`; it isn't reproducible and `--remote` may not reach it. Tell the user it is a gap |

Keep per-case data in one format your seed script reads (`case.json`, a SQL file, a directory of
fixtures), so the seed script is shared and only the data differs. A service's `seed` (a `.sql` file
for `Postgres`, a scenario for a mirror) is loaded after its migrations and before `setup()`, from a
temporary file that is deleted after.

## Directory structure

This layout is what `lmnr-cli eval run` expects. Each **dataset** is a directory directly inside `evals/`
holding an `agent/eval.py` and a `tasks/dataset.toml`. Names follow Harbor: the dataset's name in Laminar is
`[dataset] name` in `tasks/dataset.toml`, and each task's is `[task] name` in its `task.toml`, never the
directory name. Both are `org/name` (`^[A-Za-z0-9][A-Za-z0-9._-]*/[A-Za-z0-9][A-Za-z0-9._-]*$`, no `..`,
up to 255 bytes). For the org, use the agent under test (a stable name, like its service or package), else
the company; the same for a dataset and all its tasks: `support-agent/refunds`,
`support-agent/refund-double-charge`. Most projects have one dataset. Files
marked *(optional)* depend on your setup.

Everything specific to a dataset is in its own `agent/` directory, kept to the same few files in every
project. Code several datasets share (a whole runner) goes in `evals/_shared/` ("Shared runners").

```
evals/
  README.md                       # how to run, how to add a case
  .gitignore                      # */tasks/* and !*/tasks/dataset.toml  (lmnr-cli adds them)
  pyproject.toml                  # (optional) packages eval.py imports: [project] dependencies
  _shared/                        # (optional) Python several datasets' eval.py import ("Shared runners")
  <folder>/                       # e.g. support
    harbor.yaml                   # (optional) Harbor JobConfig settings: concurrency, timeouts, a model sweep
    agent/
      eval.py                     # the EvalRunner: image, services, secrets, setup, run, verify
      Dockerfile                  # (optional) only with Image.from_dockerfile; the builder needs none
      seed.py                     # (optional) run in the container by setup(): seed this case's data
    tasks/
      dataset.toml                # [dataset] name + the task versions this checkout uses; committed
      <case-dir>/                 # git-ignored: pulled from Laminar, pushed back with eval dataset push
        task.toml                 # written by lmnr-cli when missing: [task] name, [environment] docker_image
        instruction.md            # the input run() sends, or the case's scenario (lmnr-cli writes a stub)
        environment/              # no Dockerfile: uploaded into the container's workdir at start
          case.json               # this case's data; any format setup() reads
        tests/
          test.sh                 # written by lmnr-cli when missing (a stub: verify() grades)
          expected.json           # (optional) reference answer, read in verify() as ctx.case.expected()
          fixtures/               # (optional) per-case data for mirrors' seeds and mock handlers
```

Keep to these files rather than a helper tree. `seed.py` (any language: `seed.sh` works too) and any other
harness script reach the container only through `ctx.container.run_file`, which uploads it for the one command and
deletes it after, so the image needs no harness files.

A minimal `eval.py`:

```python
import json

from lmnr.evals import EvalRunner, Image, Postgres, http, secret


class SupportEval(EvalRunner):
    # Paths are relative to this file (evals/support/agent/eval.py): ../../.. is the repo root.
    image = (
        Image.python("3.12")
        .uv_sync("../../../pyproject.toml")          # locked dependencies: cached until uv.lock changes
        .add_dir("../../../app", "/srv/app/app")     # the live source, last
    )
    # Started, migrated and in DATABASE_URL before setup(), in every trial.
    services = [Postgres("15", database="support", migrations="../../../migrations")]
    model = "anthropic/claude-sonnet-5-5"            # ctx.model; --model overrides it
    secrets = ["ANTHROPIC_API_KEY"]                  # names only; values come from the shell or .env
    env = {"ANTHROPIC_API_KEY": secret("ANTHROPIC_API_KEY")}   # what the app's processes get
    new_task_toml = {"environment": {"workdir": "/case", "cpus": 2, "memory_mb": 4096},
                     "agent": {"timeout_sec": 600}}  # tasks without a task.toml get these

    async def setup(self, ctx):
        await ctx.container.run_file("seed.py", "/case/case.json")   # reads DATABASE_URL
        # The app: a foreground command; a process that exits counts as crashed.
        await ctx.container.service("api", "uvicorn --app-dir /srv/app app.main:app --port 8080",
                                    ready=http("http://127.0.0.1:8080/health"))

    async def run(self, ctx):
        r = await ctx.container.sh(f"python -m app.cli handle /case/case.json --model {ctx.model}",
                                   cwd="/srv/app", timeout=300, check=False)
        if r.return_code == 0:
            ctx.set_result(outcome="completed", reply=r.stdout.strip())
        else:
            ctx.set_result(outcome="failed", error=r.stderr[-2000:])

    async def verify(self, ctx):
        expected = ctx.case.expected()               # tests/expected.json, never in the container
        result = ctx.result()                        # what run() recorded
        if result.get("outcome") != "completed":
            return {"refund": 0, "reply": 0}
        # Host Python; only the query runs in the container (the agent's, by default).
        rows = await self.service(Postgres).sql(ctx, "SELECT * FROM refunds")
        reply_ok = await judge(result["reply"], expected["reply_should"])   # your own function
        return {"refund": len(rows) == expected["refund_count"], "reply": reply_ok}
```

- **One `EvalRunner` subclass per file.** `eval.py` runs in the Harbor process (your machine, or Laminar's
  orchestrator for `--remote`), not in the container: commands reach the container through
  `ctx.container`. Import the standard library, `lmnr`, and packages listed in `evals/pyproject.toml`
  (`[project] dependencies`; lmnr-cli installs them next to Harbor, Python 3.13; dependency groups and
  `requires-python` are ignored). A PEP 723 block at the top of one `eval.py` (`# /// script` / `# dependencies = ["httpx"]` /
  `# ///`) adds packages for that dataset only. Everything the app or a script run in the container
  needs belongs in the image.
  The file is imported more than once (to read its configuration, then by Harbor), so keep module-level
  code to constants and definitions.
- **`image`**: built with `Image`'s methods from the live source tree ("Building the image"), or
  `Image.from_dockerfile(path, context=...)` for an existing Dockerfile, or
  `Image.from_registry("python:3.13-slim")` as is. The image is named `<dataset folder>-evals`. Several
  images: a dict, `{"support-evals": ..., "support-ocr": ...}`; the first is the default, and a task picks
  another with `docker_image`.
- **`services`**: the databases, caches and simulated SaaS APIs the app talks to ("Services",
  "Mirrors"). `self.service(Postgres)` (or `self.service("postgres")`) returns one, for its helpers.
- **`model`** is the default model (`ctx.model`); `--model` and `harbor.yaml` override it, so read
  `ctx.model` instead of hard-coding one in `run()`.
- **`new_task_toml`** holds the `task.toml` tables every case shares (`[environment]` resources and
  workdir, `[agent] timeout_sec`); `lmnr-cli` writes them into a task only when it has no `task.toml` yet.
  Existing ones aren't rewritten: change those by hand.
- **`harbor.yaml` is optional.** Without it the dataset runs `tasks/` with Harbor's defaults. Add one for
  job settings (`n_concurrent_trials`, `timeout_multiplier`, `datasets` / `tasks` filters) or a model
  sweep: each `agents` entry with only `model_name` (and extra `kwargs`) runs the runner with that model.
  `eval.py` owns the image, the agent and the verifier, so a `harbor.yaml` next to it that defines
  `images`, an agent `import_path` / `name` / `env`, or a `verifier` `import_path` / `kwargs` is an error
  (exit `16`).

```yaml
n_concurrent_trials: 4                         # what the local machine can run
agents:                                        # optional: a model sweep, every case runs on each
  - model_name: anthropic/claude-sonnet-5-5
  - model_name: anthropic/claude-opus-5-5
```

- Keep what differs per case small (data, instruction, reference answer) and everything else in `agent/`,
  so a new case is its data and never needs harness changes.
- Keep reference answers in `tests/`, not in `environment/`: `verify()` reads them on the host
  (`ctx.case.expected()`, `ctx.case.fixture(...)` for `tests/fixtures/`), and Harbor never uploads them
  to the container, so neither the agent nor the app can read them.
- `instruction.md` is either the input `run()` sends to the entry point (the user message, the prompt),
  verbatim, or, when the input lives in `environment/`, a short scenario for people reading the task:
  what the case tests, its setup and input, and the expected outcome. `run()` reads it as
  `ctx.case.instruction`. `lmnr-cli` writes a one-line stub into new tasks: always replace it, never leave
  it empty or boilerplate. A driver that sends it would pass the stub to the product, and teammates read
  it in Laminar's task view.

## Eval datasets: tasks in Laminar, versions in git

Task folders (case data, instructions, verifiers) are stored in Laminar, not in git, so a dataset can hold
thousands of cases with large data. Each task version is Harbor's content hash of the task (the same one
`harbor` computes); `tasks/dataset.toml`, Harbor's dataset manifest, names the dataset and pins one
version per task, and is what you commit:

```toml
[dataset]
name = "support-agent/refunds"
description = "Refund and billing cases"   # optional; any other keys are kept

[[tasks]]
name = "support-agent/customer-complaint"
digest = "sha256:e82b0240004c2fae7c87f0430211ef4652ce1dc15ad1efa8c8bdfb8f0b71a7e0"

[[tasks]]
name = "support-agent/refund"
digest = "sha256:5fc5d83e2db1dd8cbc6c508e273fbd9604fc38d743443435755405a7ce91809d"
```

Write `[dataset] name` yourself when creating the dataset; let `lmnr-cli` write the `[[tasks]]` (sorted);
never hand-edit digests. The rest of `tasks/` is git-ignored (`evals/.gitignore` gets `*/tasks/*` and
`!*/tasks/dataset.toml`).

**Versions are independent.** Laminar stores every version of every task side by side and keeps no
"latest": `dataset.toml`, in git, is the only record of which version a dataset uses. If two people push
edits of the same task, both versions are stored, each checkout pins its own, and the git merge of
`dataset.toml` decides which one the dataset keeps (resolve a conflict on a task's `digest` by picking
one, then `pull`).

```bash
lmnr-cli eval dataset push [<dataset>...] [--task <glob>]... [--dry-run]
lmnr-cli eval dataset pull [<dataset>...] [--task <glob>]... [--force] [--dry-run]
```

- `<dataset>` is a folder name or a `[dataset] name`; `--task` globs task names, with or without the org
  (`refund-*` or `support-agent/refund-*`).
- Every `task.toml` needs a valid `[task] name`, and no two task directories may share one: pull and push
  fail otherwise. **Adding a case by copying a task directory means changing `[task] name` in the copy**
  before pushing.
- What pull and push do with each task:

  | Task in `tasks/` | pull | push |
  | --- | --- | --- |
  | Same as its pin | nothing | nothing (uploads it if Laminar doesn't have it, e.g. digests `harbor sync` wrote) |
  | Differs from its pin; Laminar doesn't have it (your edit) | refuses (exit 22); `--force` discards it | uploads it and pins it |
  | Differs from its pin; Laminar has it (out of date, e.g. after a `git pull`) | replaces it with the pin | skips it, with a warning |
  | Not pinned; Laminar doesn't have it (a new task) | leaves it | uploads it and pins it |
  | Not pinned; Laminar has it (removed from `dataset.toml`) | deletes it | skips it, with a warning |
  | Pinned, no folder | downloads it into `tasks/<name after the org>` | keeps the pin |

  Push after adding or changing cases, then commit `dataset.toml`. Pull after a `git pull` or in a fresh
  clone; it never changes `dataset.toml`, and fails if a new task's folder name holds something else.
  Push refuses tasks with `.env*` files among their hashed files.
- **Removing a case** means deleting its `[[tasks]]` entry from `dataset.toml` (push never unpins); the
  next pull deletes its folder, and Laminar keeps its versions. Renaming `[task] name` makes a new task:
  remove the old entry by hand.
- **Push the edits before pulling a `git pull`'s changes.** An edit to a task whose pin a `git pull` just
  moved is still pushed and pinned over your teammate's version; check the `dataset.toml` diff before
  committing.
- A folder without `tasks/dataset.toml` is skipped (with a warning); name it on the command line and the
  error shows the file to create.
- `eval run` never pulls: it runs what is in `tasks/`. A fresh clone needs `eval dataset pull` first
  (`eval run` fails with that hint when a dataset's pinned tasks are all missing).
- `--dry-run` lists what would be added, changed, removed or skipped, and runs the same checks (`--json`
  reports each skipped task with its reason).
- **Exit 22** (pull): it would overwrite edits Laminar doesn't have. Push them first (then merge
  `dataset.toml` as usual), or save them, pull with `--force` (which discards them), re-apply them and
  push. Never `--force` without saving edits the user wants to keep.
- The hash is Harbor's: only `task.toml`, `instruction.md`, `README.md`, `trajectory.json` and the
  `environment/`, `tests/`, `solution/` and `steps/` directories count, minus what the task's `.gitignore`
  (or Harbor's default ignores) and `.harborignore` (in `tests/`, `solution/`) exclude. Anything else in the
  task folder is neither hashed nor stored, so keep notes in `README.md`. File modes don't count. A
  symlinked task folder is not pulled, pushed or hashed.
- The project's **Datasets** page (sidebar) lists each eval dataset's tasks and every version; the old
  datasets of datapoints are under **Datasets / legacy**. The **Evaluations** table has a **Dataset**
  column linking each run to its dataset; to see one dataset's runs together, open its evaluation group
  (the dataset name, or the `--group` you used).

## Building the image

Declare the image in `eval.py` with `Image`'s methods; `lmnr-cli` compiles it to a Dockerfile and builds
it before the run. Each method returns a new `Image` and adds one layer, in the order written:

```python
image = (
    Image.python("3.12")                                   # python:3.12-slim-trixie
    .apt_install("poppler-utils")                          # system packages the app itself needs
    .uv_sync("../../../pyproject.toml")                    # locked dependencies, not the project
    .add_dir("../../../app", "/srv/app/app")               # the code, last
    .env(APP_ENV="production")
)
```

- **Bases:** `Image.python(v)`, `Image.uv_python(v)` (Python with `uv`), `Image.node(v)`,
  `Image.debian_slim()`, `Image.from_registry(ref)`. Match the language version production runs.
- **Dependencies:** `uv_sync(pyproject)` (`pyproject.toml` and `uv.lock` mounted for the build, `uv sync
  --frozen` into the image's Python), `pip_install_from_requirements(path)`, `npm_ci(dir, dst)`
  (`package.json` and the lockfile, then `npm ci`), `pnpm_install(dir, dst)` (`package.json`,
  `pnpm-lock.yaml` and `pnpm-workspace.yaml`, then a frozen install with the pnpm version in
  `packageManager`; `.npmrc` is only mounted), `pip_install(...)`, `uv_pip_install(...)`,
  `apt_install(...)`. Install from the project's lockfile so the image has the versions production has.
  Databases and caches don't go here: declare them in `services`.
- **Files and settings:** `add_dir(src, dst)`, `add_file(src, dst)`, `run_commands(...)`, `env(...)`,
  `workdir(path)` (default `/case`, where `environment/` is uploaded).
- **Order from rarely to often changing**: dependencies, then code. An app change then rebuilds (and for
  `--remote` uploads) one small layer; services always come first, so it never reinstalls them.
- **Paths are relative to the file that declares the step** (`eval.py`, or a shared module). The build
  context is the deepest directory holding everything the image adds, and only the added paths are sent,
  so a large repo costs nothing. Add specific paths from the live tree; nothing from `evals/` (harness
  scripts reach the container through `ctx.container.run_file`, reference answers stay in `tests/`).
  `.dockerignore` in the context applies, and `lmnr-cli` never lets `.git` or `.env*` into a build.
- **See what it builds:** `lmnr-cli eval describe <dataset>` prints the image's steps and services with
  the file and line that declared each; `--json` has the whole generated Dockerfile. Every run keeps a
  copy in the job folder (`jobs/<job>/<image>.Dockerfile`), and each block of it starts with a comment
  naming its origin (`# _shared/support.py:12 (SupportRunner)`).

**From a Dockerfile.** When the app already has an image definition, or needs more than these steps,
`Image.from_dockerfile("Dockerfile", context="../../..")` (relative to the file that calls it; `context`
defaults to the Dockerfile's directory: use the repo root so it copies the live source). Services and
any methods chained after it go on top of its last stage. Order it from rarely to often changing, and
`COPY` the specific paths the image needs, never `COPY . .` from the repo root: that would bake in the
whole repo, Harbor's job output and every task's `tests/` (reference answers) included. The readiness
checks of `ctx.container.service` / `ctx.container.wait_for` run under `bash`, so the image needs it (the
builder's bases have it).

```dockerfile
FROM python:3.12-slim-trixie
COPY requirements.txt /srv/app/
RUN pip install --no-cache-dir -r /srv/app/requirements.txt
COPY migrations/ /srv/app/migrations/
COPY app/ /srv/app/app/
WORKDIR /case
```

## Use one prebuilt image

Every task's `task.toml` sets `docker_image` to one of the dataset's images (`lmnr-cli` writes the default
image's name into new tasks; for a registry image, its reference), and leaves the Dockerfile out of
`environment/`:

```toml
[environment]
docker_image = "support-evals"
workdir = "/case"          # where environment/ gets uploaded; defaults to the image's WORKDIR
cpus = 2
memory_mb = 4096
```

- **Speed is the main reason.** Every trial starts from the same built image instead of building its own.
  `lmnr-cli` builds it once per run (a cached build when nothing changed); for `--remote` it uploads only
  missing layers, so an unchanged image uploads nothing. No tags to bump.
- **Per-case files still arrive.** When `docker_image` is set and `environment/` has no `Dockerfile` or
  `docker-compose.yaml`, Harbor uploads the contents of `environment/` into the workdir after the container
  starts. That is how `case.json` reaches a shared image.
- **Resources.** A full stack per case is heavy: set `cpus`/`memory_mb`. Locally every trial shares the
  host, so keep `n_concurrent_trials` to what it can run. On `--remote` each trial gets its own sandbox;
  raise concurrency for that run with `--n-concurrent`.
- **Tasks never define an image.** A task with its own `environment/Dockerfile`, or a `docker_image` the
  runner doesn't declare, fails `eval run` and `eval dataset push` (exit `16`) before anything runs:
  Harbor would build or pull that image itself, without what the runner adds to its images. When a case
  genuinely needs a different system, declare a named image on the runner and pick it in that task:
  `image = {"support-evals": base, "support-ocr": base.apt_install("tesseract-ocr")}` (the first is the
  default), then `docker_image = "support-ocr"` in its `task.toml`. A variant built from the same `base`
  shares its layers, so it costs one extra layer. Everything else that varies per case is data:
  `environment/`, `tests/fixtures/`, `[metadata]` in `task.toml`.

## Shared runners

When several datasets test the same product with the same harness (same app, services and entry
point), they share the runner instead of copying it, and so share one image:

```python
# evals/_shared/support.py
from lmnr.evals import EvalRunner, Image, Postgres, Valkey, secret

class SupportRunner(EvalRunner):
    image = Image.python("3.12").uv_sync("../../pyproject.toml").add_dir("../../app", "/srv/app/app")
    services = [Postgres("16", migrations="../../migrations"), Valkey("8")]
    secrets = ["ANTHROPIC_API_KEY"]
    env = {"ANTHROPIC_API_KEY": secret("ANTHROPIC_API_KEY")}

    async def setup(self, ctx):
        await ctx.container.run_file("seed.py")                               # evals/_shared/seed.py

    async def run(self, ctx): ...
    async def verify(self, ctx): ...

# evals/refunds/agent/eval.py
from _shared.support import SupportRunner

class RefundsEval(SupportRunner):
    env = {**SupportRunner.env, "SUPPORT_QUEUE": "refunds"}       # the only difference, visible here

# evals/ocr/agent/eval.py
class OcrEval(SupportRunner):
    image = SupportRunner.image.apt_install("tesseract-ocr")      # a variant: one more layer
```

- Any folder directly in `evals/` that isn't a dataset (no `harbor.yaml`, no `agent/eval.py`; by
  convention `_shared/`) holds shared Python. The evals folder is on the import path both when `lmnr-cli`
  reads `eval.py` and when Harbor runs it, and `--remote` uploads the folder with the run.
- Each `eval.py` still defines exactly one runner class, which can be a one-line subclass. Write a
  difference as an explicit extension of the parent's attribute, so a reader sees it in one place:
  `env = {**SupportRunner.env, ...}`, `secrets = SupportRunner.secrets + [...]`,
  `services = [*SupportRunner.services, ClickHouse("25.8")]`, `image = SupportRunner.image.<step>(...)`.
- **Identical images are built once.** Datasets whose images come out the same (same steps, same
  services, same files) share one build in a run, whichever runner declared them, and a variant reuses
  the layers of the image it extends.
- **Each step records where it was declared.** The generated Dockerfile has a comment above each block
  (`# _shared/support.py:12 (SupportRunner)`, `# service: Postgres 16 (_shared/support.py:13)`), and
  `lmnr-cli eval describe <dataset>` shows the runner's class chain with their files, each image step and
  each service with its origin. Run it to answer "where does this dataset's image / env come from?"
  before editing a shared runner.
- **Relative paths resolve against the file that declares them.** `Image` steps,
  `Image.from_dockerfile(...)` and services' `migrations` / `seed` resolve against the file they are
  called in, `ctx.container.run_file(...)` / `ctx.container.copy(...)` against the file whose code calls
  them. So the shared runner's paths and scripts work from every dataset, while a subclass's own
  `ctx.container.run_file("x.py")` finds `x.py` next to its `eval.py`.
- **The image comes only from the runner class chain.** Tasks pick one of the runner's named images and
  nothing else ("Use one prebuilt image"); what differs per case is data the shared code reads.
- Shared modules import the standard library, `lmnr` and packages from `evals/pyproject.toml`, never
  `harbor`.
- A remote run's trial **Files** view shows the shared modules the runner imported in `runner/`, next to
  `agent/`.

## Secrets and env vars

Declare secrets by name in `eval.py`, never as values and never as `harbor run` flags:

- `secrets = ["ANTHROPIC_API_KEY", ...]` lists every secret the run needs. `verify()` reads them in the
  Harbor process as `ctx.secrets["NAME"]` (an LLM judge's key never has to enter the container). Reading
  an undeclared name is an error.
- `env = {...}` is the env of every command `ctx` runs in the container, in every phase. Pass a secret
  with `secret("NAME")` (it must be in `secrets`); plain strings are settings. A secret that isn't in
  `env` never reaches the container. Per-command extras go in `ctx.container.sh(..., env={...})`, but in `setup` /
  `run` / `collect` they can't override a key of `env` (Harbor applies the agent env last).

`lmnr-cli eval run` reads each secret from the shell, then from `.env.local` / `.env` in the current
directory, and fails before anything starts (exit `19`) if one is missing, so the user only has to fill in
`.env`. For `--remote` the values travel with the run as env vars; files with secrets in them are never
uploaded, and neither are a task's `.env*` files when you push it. Add `.env` and Harbor's job output
(`evals/*/jobs/`) to `.gitignore`, and write a `.env.example` listing the names.

`harbor.yaml` and `task.toml` still take Harbor `${VAR}` placeholders (`[environment.env]` for the
container, `[verifier.env]`), resolved the same way; `${VAR:-default}` makes one optional.

## The runner: `agent/eval.py`

Each trial runs four phases, each given a `ctx` for that trial:

- **`setup(ctx)`**: seed this case's data and start the app, after Harbor uploaded `environment/` into
  the workdir and the runner started the declared `services`. Default: run the case's own
  `environment/seed.py` / `seed.sh` if it has one.
- **`run(ctx)`**: drive the entry point and record the outcome with `ctx.set_result(...)`. For a single
  command, set `run_command = "..."` instead of defining it.
- **`collect(ctx)`** *(optional)*: runs after `run()`, also when it raised or timed out (with two minutes
  of its own after a timeout). Export what the run left behind: a reconstructed trajectory, extra logs,
  state for grading (`ctx.artifacts.save`, see "The verifier").
- **`verify(ctx)`**: grade (see "The verifier").

Every method runs in the Harbor process, on the host (your machine, or Laminar's orchestrator for
`--remote`), never in the container. `ctx` splits along that line:

| | Where it acts | What |
|---|---|---|
| `ctx.container.*` | in the container | `sh`, `run_file`, `service`, `wait_for`, `copy`, `write_file`, `read_text`, `read_json` |
| the rest of `ctx` | on the host | `case`, `model`, `env`, `logger`, `set_result` / `result`, `artifacts`, `secrets` and `mocks` (verify) |

`ctx.container` is the agent's container in `setup` / `run` / `collect`, and the grading container in
`verify` (the same one by default; see "The verifier"). Anything that must run next to the app goes
through it; plain Python in your methods (a judge call, reading `ctx.case.expected()`) runs on the host.

- `ctx.container.sh(cmd, env=None, cwd=None, user="agent", timeout=None, check=True)`: a shell command;
  returns `stdout`, `stderr`, `return_code`, and raises `CommandError` (with the output's tail) on a
  non-zero exit unless `check=False`. `user="agent"` is the task's agent user (the verifier user in
  `verify()`), `"root"` is root.
- `ctx.container.run_file("seed.py", *args, ...)`: uploads a host file to a fresh `/tmp` directory, runs
  it (shebang, `.py` with `python3`, else `bash`) and deletes it, so harness scripts never stay where the
  agent under test could find them. A relative path resolves against the file whose code calls it
  (`eval.py`, or a shared module); `ctx.container.copy` resolves `host` the same way.
- `ctx.container.service(name, cmd, ready=..., env=None, timeout=60, user="agent")`: for processes
  that aren't declared `services`, typically the app under test. Starts `cmd` in the
  background (output in `/logs/agent/<name>.log`) and waits until `ready` passes. `cmd` must stay in the
  foreground (`postgres -D ...`, not `pg_ctl start`; no `--daemonize`): a process that exits counts as
  crashed. `user` also takes a service user (`"postgres"`). Ready checks: `port(5432)`, `http(url)`,
  `cmd("pg_isready")` (from `lmnr.evals`), or a plain string command. If the process exits or isn't ready
  in time, the error carries the end of its log. `ctx.container.wait_for(ready, timeout=60)` waits
  without starting anything (an async job's terminal status, say).
- `ctx.container.copy(host, path)` copies a file or directory from the host into the container at
  `path`. `ctx.container.write_file(path, content)`, `read_text(path)` and `read_json(path)` write and
  read container files.
- `ctx.set_result(**fields)` (agent phase) records fields in the trial's `agent/result.json` on the host,
  merged with earlier calls; `ctx.result()` reads them back, in `verify()` too, in both verifier modes.
  The fields show as the trial's output in Laminar when there is no trajectory.
- `ctx.case`: `name` (`[task] name`), `dir` (the task directory on the host), `metadata` (`[metadata]`
  in `task.toml`), `instruction`, `file(path)` (under `environment/`), `expected()` (`tests/expected.json`),
  `fixture(path)` / `fixture_json(path)` (under `tests/fixtures/`).
- `ctx.model`, `ctx.env` (the resolved `env`), `ctx.logger` (its lines also go to the verifier's log).
- `self.service(Postgres)` / `self.service("name")` and `self.mirror("slack")`: a declared service, for its
  helpers ("Services", "Mirrors"); `ctx.state("postgres", "refunds")` (verify) reads a table a service
  saved with `snapshot=[...]`.

Rules that keep cases comparable:

- **An exception fails the trial** (Harbor records it in `exception.txt`, and there is no reward). Use
  `check=False` and record the product's own failures as outcomes (`ctx.set_result(outcome="failed", ...)`,
  `"timeout"`), so `verify()` grades them; let only harness errors (a service that won't start, a seed
  error) raise.
- `[agent] timeout_sec` in `task.toml` bounds `setup` + `run`; keep command timeouts below it.
- Pass `ctx.model` to the app so `--model` and a model sweep work.

## The verifier

`verify(ctx)` returns the rewards: a number, a bool (`1` / `0`), or a dict of them for several scores
(`{"refund": True, "reply": 0.5}`), each charted in Laminar. One method grades every case. It runs in the
Harbor process (the host), so it can call an LLM judge with a key the container never sees and read
`ctx.case.expected()`, which never enters the container; `ctx.container` commands run in the grading
container, so it can query the app's database or read files the run wrote. Which container that is, is
`[verifier] environment_mode` in `task.toml`:

- **`"shared"`** (the default): the agent's container, after the agent finishes, with everything the run
  left behind (services keep running).
- **`"separate"`**: a fresh container from the same image, which sees only the run's artifacts.
  `ctx.result()` still works. No services run there and their env isn't set: give a service
  `snapshot=["table", ...]` (a mirror `snapshot=True`) so `collect` saves its state, and read it with
  `ctx.state(...)` / `mirror.state(ctx)`; the live helpers (`sql`, `query`, `cmd`) raise. Prefer this
  when grading must not see the app's internals. It's rare: most datasets grade in the shared container.

**Artifacts** are the files collected for grading after the agent phase (both modes). Declare them on the
runner, or save them from `collect`:

```python
from lmnr.evals import Artifact

class SupportEval(EvalRunner):
    artifacts = ["/srv/app/out/", Artifact("/var/lib/app/state.db", destination="app.db")]

    async def collect(self, ctx):
        await ctx.container.sh("python -m app.cli dump-state > /tmp/state.json", check=False)
        await ctx.artifacts.save("/tmp/state.json")  # -> /logs/artifacts/state.json
```

- `ctx.artifacts.save(path, name=None)` (agent phase) copies a container file or folder to
  `/logs/artifacts/<name>` (default: its basename), replacing whatever the agent under test may have put there. Snapshot runtime state
  (database contents, in-memory counters) this way before the container goes away.
- In `verify()`, `ctx.artifacts.read_text(path)` / `read_json(path)` / `path(path)` read them on the host
  by the container path they came from (`"/srv/app/out/report.json"`, `"/logs/artifacts/state.json"`),
  in both modes; `ctx.artifacts.list()` lists what was collected.
- In `"separate"` mode the grading container gets them too: declared ones at their original path, saved
  ones under `/logs/artifacts/`, so a script run with `ctx.container.run_file` can read them there.
- Declare them on the runner, not in `harbor.yaml` (`lmnr-cli` rejects that); a task's own
  `artifacts = [...]` in `task.toml` adds to them.

**LLM judges are your own code.** `verify()` runs in the Harbor process, so a judge is a plain function
that calls any SDK listed in `evals/pyproject.toml`, with its key from `ctx.secrets` and its inputs from
`ctx.result()`, `ctx.case.expected()` and artifacts. Decorate it with `@observe(name="judge",
ignore_inputs=["api_key"])` to get its own span without recording the key. Combine hard checks on the
terminal state (rows, status, the response) with a judge only for what can't be checked exactly (the
quality of a reply). lmnr-python's `examples/agent-evals` has one.

In Laminar, `verify()` is traced under the trial's verifier span, with LLM calls it makes as LLM spans,
and its log (`ctx.logger` lines) is the trial's `verifier/test-stdout.txt`.

**Grading an imported Harbor dataset.** Its tasks bring their own `tests/test.sh`. To grade with those
instead of `verify()`, set `grade_with_test_sh = True` on the runner and don't define `verify()` (both is
an error). Harbor's default verifier then runs each task's `test.sh` in the grading container, the script
writes its reward to `/logs/verifier/reward.txt`, and there is no verify `ctx`. `lmnr-cli` doesn't write
stub `test.sh` files then, and warns about a task without one.

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
- the app process actually receives `LMNR_TRAJECTORY_PATH` (every `ctx` command in `setup` / `run` /
  `collect` gets it; forward it if the app is started with `env -i`);
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

Export the trajectory in `collect()`, which runs after `run()` including on `failed` / `timeout` (partial
trajectories are the most useful): `ctx.container.run_file` a script that reads this case's history from the store,
inside the container, and writes `/logs/agent/trajectory.json` in Harbor's ATIF format, which the plugin
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

Be explicit about what's missing. Stored history usually lacks token counts, cost, the system prompt, the
exact prompt of each LLM call (one stored turn may be several calls), retries and per-call latency. Leave
`metrics` unset rather than estimating, and say in `notes` where the trajectory came from (e.g.
`"reconstructed from chat_messages"`). While developing, validate a written file with
`harbor.models.trajectories.Trajectory` (`uvx --from harbor==0.25.0 python`), and check the replayed steps
in the trial's trace. Tell the user the trajectory is reconstructed and which fields it lacks.

If the app keeps no history at all, the trial has only `ctx.set_result(...)` and the service logs. Say so and
suggest instrumentation.

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
Single container, no docker-in-docker: the real dependencies run in it. Declare them on the runner;
user code never starts them:

```python
from lmnr.evals import ClickHouse, Postgres, Valkey

class SupportEval(EvalRunner):
    services = [
        Postgres("16", database="support", migrations="../../../migrations", extensions=["pg_trgm"]),
        ClickHouse("25.8", database="analytics", migrations="../../../clickhouse/migrations"),
        Valkey("8"),                                        # REDIS_URL
    ]
```

- **At image build** `lmnr-cli` adds each service to every image of the runner: its binaries come from
  the vendor's official image at that version, in a stage of their own, so app changes never reinstall
  them.
- **In every trial, before `setup()`** the runner starts all of them (in parallel, each as its own OS
  user, on 127.0.0.1, logging to `/logs/agent/<name>.log`), waits until they're ready, runs their
  migrations and this case's `seed`, then adds their connection env (`DATABASE_URL`, `CLICKHOUSE_URL`,
  `REDIS_URL`) to every later command, the app's included. A service that fails to start or migrate fails
  the trial with the end of its log.
- **`setup()` only seeds and starts the app**: `ctx.container.run_file("seed.py")` (it reads the same
  env), or the helpers: `self.service(Postgres).insert(ctx, "users", rows)`, `.sql(ctx, query) -> rows`,
  `.execute(ctx, sql)`, `.load_sql(ctx, "baseline.sql")`; `self.service(ClickHouse).insert(ctx, table,
  rows)` / `.query(ctx, sql)`; `self.service(Valkey).cmd(ctx, "SET", "k", "v")`. `ctx.container.service`
  is for processes no `Service` covers, typically the app itself.
- **Versions are required:** pin the version production (or the project's compose file) runs. `"latest"`
  works but warns on every run.
- **Env:** `env="DATABASE_URL"` names the variable; a dict sets several from templates:
  `env={"PGHOST": "{host}", "PGPORT": "{port}", "APP_DB": "{url}"}` (`{url}`, `{host}`, `{port}`,
  `{database}`, `{user}`, `{password}`). The app must read its connection from env; a service's variable
  may not also be in the runner's `env`. In a separate verifier, service env isn't set.
- **Schemas:** if the app has migrations, reproduce what its own startup/migration does, in the same
  order: extra roles, extra databases, numbered migrations, then anything created *after* migrations in
  code (dictionaries, views, grants, seed rows). Reading the app's boot hook is the only reliable way to
  find these. `Postgres(migrations=dir)` applies the directory's `*.sql` files in natural sort order
  (subfolders skipped) from the live tree. `ClickHouse(migrations=dir)` applies them with the
  `clickhouse-migrations` CLI (files named `1_init.sql`, versions recorded in `_migrations`); for files
  the app applies some other way, `migrator="clickhouse-client"` runs every `*.sql` in natural order.
  `Postgres(migrate="alembic upgrade head")` runs the app's own migration command instead (in the
  container, with the service env), and
  `init_sql=[...]` / `extensions=[...]` run before migrations. With plain SQL migrations,
  `migrate_at="build"` runs them while the image is built, so trials start from a migrated database.
  What must happen after migrations goes in `setup()`.
  If the app has no migrations (schema managed elsewhere, ORM auto-create off, a shared database owned by
  another service), infer the schema from the code: model/schema classes, ORM definitions, the queries the
  entry point runs. Create **only the tables and columns the path under test needs**, in a SQL directory
  next to `eval.py` passed as `migrations=`. Tell the user the schema was inferred, since it can drift
  from the real one.
- **Server config:** `ClickHouse(config="clickhouse/config.xml", users="clickhouse/users.xml")` merges
  production's server and user settings over the defaults (keep the ports in step with `http_port` /
  `tcp_port`, and the `default` user without a password).
- **Base image:** the services need glibc 2.36 or newer (Debian bookworm / trixie, Ubuntu 24.04; the
  builder's bases are trixie). On an older base, pass `source=` an official image built on an older
  distribution (`Postgres("16", source="postgres:16-bullseye")`); Alpine can't take them. `source=` also
  picks a compatible image with extras: `Postgres("16", source="pgvector/pgvector:pg16-bookworm",
  extensions=["vector"])`. ClickHouse's default migrator needs glibc 2.38 (trixie, Ubuntu 24.04): on a
  bookworm base, pass `migrator="clickhouse-client"` (the build fails with that hint otherwise).
- **Anything else** (MySQL, MinIO, an internal service): `Service.custom(name, version, start=...,
  ready=..., source=..., bundle=[paths], env=..., port=...)` copies `bundle` paths (and the shared
  libraries they need) from the `source` image; or `install=[Dockerfile lines]` installs it in the final
  image. `start` must stay in the foreground ("Harbor runtime facts and quirks").
- *Queues/caches/object stores.* If the app has an in-memory or local-disk fallback, use it by leaving the
  URL env unset. That is production-supported code, not a mock.
- *Third-party SaaS APIs.* Use a mirror where one exists ("Mirrors"); otherwise an HTTP mock behind the
  app's existing base-URL setting ("Mocks"). Not the model provider: the model stays real ("LLM
  handling").
- *The app itself.* Run the production artifact: the real package started by `setup()`, or a compiled
  binary. For a library or callable function, `run()` runs a small script (`ctx.container.run_file`) that
  imports and calls it in-process.
- *Readiness.* Give the app's `ctx.container.service` a `ready` check: the health endpoint (`http(...)`),
  the port (`port(...)`), or a probe command (`cmd(...)`). Declared services have theirs built in.
- *Env.* `ctx` commands get only the runner's `env`, the services' env and what Harbor sets, never the
  host's shell. If the app is started with `env -i` and an explicit minimal env, forward the names it
  needs (the service URLs, API keys, `LMNR_TRAJECTORY_PATH`).

## Mirrors

A mirror is a service that simulates a third-party SaaS API (Slack, Linear, GitHub, Stripe...) inside
the container: the app talks to it through its usual base-URL setting, each case seeds it with its own
data, and `verify()` reads what the app did to it. Rystic is the first provider:

```python
from lmnr.evals import case_fixture
from lmnr.evals.mirrors import Rystic

class SupportEval(EvalRunner):
    services = [
        Postgres("16", migrations="../../../migrations"),
        Rystic("slack", version="1.4", seed=case_fixture("slack.json"),
               env={"SLACK_API_URL": "{url}/api/"}),
    ]

    async def verify(self, ctx):
        slack = self.mirror("slack")
        state = await slack.state(ctx)        # the simulator's state after the run
        calls = await slack.calls(ctx)        # every API call the app made
        ...
```

- **Remote runs only, for now** (`lmnr-cli eval run --remote`). Laminar gives every trial sandbox its own
  short-lived Rystic access (`RYSTIC_TOKEN` and the proxy URLs). Never put `RYSTIC_TOKEN` in `env`,
  `secrets` or `harbor.yaml`: it would replace the trial's. A local run fails at the mirror's start with
  that reason; tell the user to run the dataset with `--remote`.
- `version` pins the simulator. The image gets the pinned Rystic CLI (no key needed); each trial
  downloads the simulator before it starts.
- `seed=case_fixture("slack.json")` loads each case's `tests/fixtures/slack.json` (a register map) after
  start; a case without the file starts empty (`case_fixture(..., required=True)` makes it an error). A
  plain path is the same seed for every case. The seed never stays on disk in the container.
- `env` templates use `{url}` (`http://127.0.0.1:<port>`) and `{port}`: point the app's existing
  base-URL setting at it. If the app hard-codes the vendor's URL, that is a gap to raise with the user,
  not a reason to patch the client.
- In `verify()`: `mirror.state(ctx)`, `mirror.calls(ctx)`, `mirror.get(ctx, "/api/conversations.history",
  channel="C1")` (a live request), and `await mirror.settle(ctx)` in `run()` to wait until the app
  stopped reacting. With a separate verifier, set `snapshot=True` so `collect` saves the state and the
  calls for `state()` / `calls()`.
- See [simulated-services.md](simulated-services.md) for Rystic's products, scenarios and the hosted
  proxy.

## Mocks

A mock answers in place of a dependency you can't run: a private internal service, a SaaS with no
mirror. It is the last resort, in this order: **the real service** (a declared `services` entry), **a
mirror**, **a mock**. A mock only says what you wrote it to say, so it is a gap in what the eval
measures: **tell the user about every mock**, what it replaces, and why the real thing or a mirror
couldn't be used.

```python
from lmnr.evals import EvalRunner, mock

def get_customer(call, ctx):              # function mock: the app's observed crm.get_customer
    customer = ctx.fixture_json("crm.json").get(call.args["email"])
    if customer is None:
        raise mock.Error("CustomerNotFound", f"no customer {call.args['email']}")
    return customer

def helpdesk(req, ctx):                   # HTTP mock
    if req.method == "POST" and req.path.endswith("/replies"):
        return mock.Response(201, json={"id": f"reply-{req.index}"})
    return None                           # anything else: 501, logged as unmatched

class SupportEval(EvalRunner):
    mocks = {
        "crm.get_customer": get_customer,
        "https://helpdesk.example.com/api": mock.http(helpdesk, base_url_env="HELPDESK_API_URL"),
    }
    # What eval.py imports at the top (here the judge's SDK; the model itself is never mocked).
    mock_dependencies = ["anthropic>=2.4.0"]

    async def verify(self, ctx):
        posts = [c for c in ctx.mocks.calls("https://helpdesk.example.com/api")
                 if c["request"]["method"] == "POST"]
        return {"posted": len(posts) == 1}
```

- **The model stays real.** The app's LLM calls go to the real model ("LLM handling"), not to a
  mock. If the user explicitly asks to run without model calls, the stub that section describes is an
  HTTP mock on the model's base URL (`mock.http(..., base_url_env=<the app's model base URL variable>)`),
  never a function mock of the code that calls the model. Say in the report and the dataset's README
  that the scores don't measure the model.
- **HTTP mocks** are the default. The key is the API's base URL (a host, optionally with a path); the
  runner sets `base_url_env` to the mock's URL, so point the app's existing base-URL setting at it,
  exactly as for a mirror. If the app hard-codes the vendor's URL, raise it with the user; don't patch
  the client. A handler gets `req` (`method`, `path` (everything after the host), `query` (a dict of
  lists), `headers`, `body`, `text()`, `json()`, `index`) and returns a JSON value (sent as `200`),
  `mock.Response(status, json=... | text=... | body=..., headers=...)`, or `None` / raises
  `mock.Unmatched` for a request it doesn't handle (`501`). `raise mock.Error(type, message)` answers
  `500` with that error.
- **Function mocks** replace an observed function in the app: when the app calls it, its Laminar SDK
  asks the mock server first and returns or raises what the handler says, without running it. Use one
  when the dependency has no base-URL setting to point at a mock (an SDK client, gRPC, a function that
  does several calls) but sits behind a function the app already observes.
  - *Key*: `observe(mock_key=...)` (TS `mockKey`), else the span name (`@observe(name="crm.get_customer")`),
    else the function's name. `observe(mockable=False)` (TS `mockable: false`) makes a function always
    run.
  - *Handler*: gets `call` (`key`, `args`, `index`) and returns the function's return value, raises
    `mock.Error(type, message)` for the function to raise, or returns `mock.PASSTHROUGH` to run the real
    function. `args` are by parameter name in Python (`self` / `cls` dropped, defaults filled in) and the
    argument list in TypeScript, serialized like span input.
  - *In the app*, `mock.Error` raises `lmnr.MockedError` (`.type`, `.message`) in Python and a
    `MockedError` from `@lmnr-ai/lmnr` whose `name` is the type in TypeScript. Python builds a returned
    value into the function's return annotation when it fits (pydantic models, dataclasses, TypedDicts),
    and returns the JSON as is, with a warning in the app's log, when it doesn't.
  - *Requirements*: the app needs `lmnr` 0.9.3 or later (Python) / `@lmnr-ai/lmnr` 0.9.8 or later
    (TypeScript); with an older SDK the real function runs, silently. Check the app's lockfile. Upgrading
    the SDK, or adding `@observe` to a function that doesn't have it, is a change to the product: propose
    it to the user, don't make it as part of the eval.
  - *Sync and async*: a mocked function keeps its shape in both SDKs: a sync function returns the value,
    an async one a coroutine / Promise. In TypeScript only functions declared `async` count as async: a
    plain function that returns a Promise (`(q) => client.search(q)`) gets the value itself, which
    `await` accepts but `.then(...)` doesn't, so mock it with an `async` function. Generators can't be
    mocked in either SDK.
  - *Failures fail the call, never fall back*: an unreachable mock server makes every observed call in the
    app raise `MockedError` (type `MockFailure`), and so does a handler that crashes (its traceback is in
    `agent/mocks.log`). The trace marks answered calls with `lmnr.span.mocked`, and the call log has each
    call's `trace_id` / `span_id`.
- Every handler also gets `ctx`: `ctx.state` (one dict per trial, shared by all handlers),
  `ctx.calls` (this key's earlier calls), `ctx.fixture(path)` / `fixture_text` / `fixture_json` (the
  case's `tests/fixtures/`), `ctx.case.name` / `ctx.case.metadata`, `ctx.env`. Calls are handled one at a
  time, so `index` and `state` are deterministic.
- **Handlers run in the container**, in a mock server the runner starts after `services` and before
  `setup`. They must be top-level functions (sync or async) in `eval.py` or a module under `evals/`
  (`_shared/mocks.py`), not lambdas or methods. The server imports `eval.py` there, so whatever it
  imports at the top must be installed: list it in `mock_dependencies` (they go into the Laminar layer,
  not the app's environment). The handlers and fixtures exist only in the server's memory once it has
  started; the agent can't read them.
- **Per-case data** goes in `tests/fixtures/`, next to `expected.json`, never in `environment/` (which
  the agent can read).
- **Grade with the call log.** In `verify()`, `ctx.mocks.calls(key)` lists one mock's calls in order
  (`request`, `outcome`, `response`, `index`, `ts`); `ctx.mocks.all()` all of them; `ctx.mocks.unmatched()`
  the requests no handler answered (a sign the mock misses a route the app uses). The log is collected
  as `/logs/artifacts/mocks.jsonl` (so it works with a separate verifier too); the server's own log,
  with handler tracebacks, is the trial's `agent/mocks.log`.
- The runner also sets `LMNR_MOCK_URL` (the server's address, for the SDK's function mocks). Never set
  it yourself; if the app is started with an explicit env (`env -i`), forward it and every
  `base_url_env` like the service URLs ("Services", *Env*).

## LLM handling
- **The LLM is real.** Evals exist to measure how the product behaves with a model; a canned response only
  tests the harness. Never substitute a fake or recorded model for the one under test unless the user
  explicitly asks for it (e.g. to run the harness without model calls). Then serve the canned responses
  at the network edge (a stub server on the app's model base URL, not a patched client), and say in the
  report and the dataset's README that the scores don't measure the model.
- Route the app's model calls through its normal configuration (profile row, base URL env), pointing at the
  endpoint and key from the runner's `env` (`secret("...")`). Don't patch the client.
- A failing or unreachable LLM endpoint often makes the app **retry rather than fail**. Your timeout is
  what ends the run, so report `timeout` distinctly from `failed`.

## Staleness
- `lmnr-cli eval run` rebuilds the dataset's images and reloads `eval.py` on every run, so a change to
  `agent/` or to the app is picked up next run.
- Build from the **live source paths** (the app source, its migrations), not from a vendored copy, so evals
  can't drift from the schema they run against. The app artifact and the migrations in one image come from
  the same revision by construction.
- Docker layer caching can still hide a change (e.g. a `RUN` that downloads "latest"). If something looks
  stale, check the file inside a running container, then clear the cache with `docker builder prune` or
  bump a cache-busting `ARG` placed before the stale step.

## Compiled artifacts
If the product ships as a compiled binary, build it in a builder stage of the same Dockerfile
(`Image.from_dockerfile`; services still go on top of its last stage), not on the host: a macOS host
binary is the wrong format, and cross-compiling tends to fail on native dependencies. Use a builder base
whose libc is not newer than the runtime stage's, and install the shared libs the binary links at
runtime. Check what it reads relative to its working directory (templates, data files); a server
missing them can start but silently never bind its port.

## Harbor runtime facts and quirks
- Harbor starts the container with its own keep-alive command (`sleep infinity` on local Docker), so don't
  rely on `CMD` or a base image's entrypoint (the `postgres` image's init) to start anything. Declared
  services and `ctx.container.service` processes (`nohup`, output to `/logs/agent/<name>.log`) survive
  after `setup()` returns. A background process started with plain `ctx.container.sh` must redirect its
  output, or the call may not return.
- **No ports are published to the host.** Every call to the app must come from a process inside the
  container (`ctx.container.sh`, `ctx.container.run_file`).
- `/logs/agent/` and `/logs/verifier/` are shared between container and host. Write every log under
  `/logs/agent/`.
- For `Service.custom` and `ctx.container.service` processes (declared built-ins handle these): some
  servers refuse to run as root (start them with `user=` their service user, and make the log directory
  writable by it first: `await ctx.container.sh("chmod 777 /logs/agent", user="root")`); `cmd` must stay in
  the foreground (`postgres -D ...`, not `pg_ctl start`; no `--daemonize`), since an exited process counts
  as crashed.
- Some apps need a password in the connection string even when the database uses trust auth; built-in
  `Postgres` puts a dummy one (`password=`) in the URL.
- Keep command timeouts below the task's `[agent] timeout_sec`, and the entry point's own timeout below
  both.
- **Secret scrubbing:** Harbor redacts agent env values from its output files. A very short value
  (a one-character placeholder like `x`) turns every occurrence of that character in the job output into
  `[REDACTED]` (`ma[REDACTED]_timeout_sec`), so keep placeholder values long and distinctive.
- **Job output:** `evals/<dataset>/jobs/<timestamp>/` for local runs. Each trial has `agent/`
  (your logs, `result.json`) and `verifier/` (`reward.json`, `test-stdout.txt` with `verify()`'s log);
  `exception.txt` holds the traceback of a phase that raised.

## Turning production data into cases
The loop this setup exists for: pick production traces a signal flagged (and a few it correctly didn't),
turn each into a case's data, grade whether the failure recurs, and add the case as a directory copy. To
build cases from a cluster or a signal's events (choosing and deduplicating traces, mapping a trace onto
seed data, the verifier, what to report), read [evals-from-clusters.md](evals-from-clusters.md).

### Adding a case

Once the harness exists, a case is its data:

1. `lmnr-cli eval task new evals/<dataset>/tasks/<case-dir>` writes `task.toml` (`[task] name =
   "<org>/<case-dir>"` with the `[dataset] name`'s org, `docker_image`, the runner's `new_task_toml`),
   `instruction.md` and `tests/test.sh` (stubs; replace `instruction.md` in step 2). `eval run` and `eval dataset push` write the same files
   for any task directory that lacks them, and add a missing `[task] name` / `docker_image` to an existing
   `task.toml`, so a directory with `environment/`, `tests/expected.json` and its `instruction.md` is a case.
2. Drop in the case data (`environment/case.json`) and the reference answer (`tests/expected.json`), and
   write `instruction.md`: the input `run()` sends, verbatim, or the case's scenario (see "Directory
   structure").
3. `lmnr-cli eval run --task <case-dir>`, then the whole dataset, then `lmnr-cli eval dataset push` and
   commit `tasks/dataset.toml`.

Copying a task directory instead works too, but then change `[task] name` in the copy.

### When the harness doesn't fit
Adding a case should not need harness changes. If one does, first try to generalise `setup()` / `run()` in
`eval.py` (or `seed.py`) so every case benefits. If the behaviour can't be reached at all without changing production code
(no public entry point reaches it, or its inputs can't be set from outside), **stop and raise it with the
user**. Explain what is missing and discuss whether adding a test-only entry point is acceptable (e.g. an
internal endpoint or CLI command, disabled unless explicitly enabled by config). Don't add one silently.

## Without eval.py: a hand-written Harbor agent

Datasets written before `eval.py`, or kept runnable with bare `harbor run`, use Harbor's interfaces
directly. Don't mix the two in one dataset: next to `eval.py`, `lmnr-cli` rejects an agent `import_path`
or a verifier in `harbor.yaml`. Declared `services`, mirrors and the `Image` builder need `eval.py`;
here the Dockerfile installs the services and the agent starts them. The layout replaces `eval.py` with:

```
<folder>/
  harbor.yaml          # agents, datasets, ${VAR} secrets, and the images key (required here)
  agent/
    Dockerfile         # also COPYs seed.py, run.py and verify.py to /opt/agent/
    agent.py           # BaseInstalledAgent subclass: runs seed.py, then run.py, in the container
    seed.py            # boot the services, seed baseline data + this case's environment/ files
    run.py             # drive the entry point; write /logs/agent/result.json
    verify.py          # grade the run; each task's tests/test.sh calls it
  tasks/<case-dir>/    # task.toml with [task] name and docker_image, instruction.md, environment/,
                       # tests/test.sh (`python /opt/agent/verify.py`), tests/expected.json; all by hand
```

In this layout a task may also bring its own image, as plain Harbor allows: an `environment/Dockerfile`
with no `docker_image` (while `docker_image` is set Harbor uses it and ignores the Dockerfile, unless the
job sets `environment.force_build: true`). Harbor then uploads nothing at start, so the Dockerfile must
`COPY` the case data from its build context, the task's `environment/`; it can't `FROM` an image that
exists only in `harbor.yaml`. A `docker_image` that isn't a `harbor.yaml` image is pulled from its
registry.

```yaml
agents:
  - import_path: agent.agent:MyAppAgent      # relative to the dataset dir, which is on PYTHONPATH
    model_name: anthropic/claude-sonnet-5-5   # read as self.model_name; overridable with --model
    env:
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
datasets:
  - path: tasks
n_concurrent_trials: 4
images:                                      # read (and stripped) by lmnr-cli; Harbor ignores it
  myapp-support-evals:
    dockerfile: agent/Dockerfile
    context: ../..
```

The agent subclasses `BaseInstalledAgent` (`harbor.agents.installed.base`; write against Harbor 0.25.0,
https://docs.harborframework.com/agents/custom-agents), which runs **inside** the task environment.
(`BaseAgent` runs on the host and only reaches the container through `environment.exec(...)`; it is
rarely the right fit here.)

- `name()` (static), and `run(instruction, environment, context)` decorated with `@with_prompt_template`.
- `install(environment)` is required and runs in every trial, before `run()`. With a prebuilt image leave
  it as `pass`: anything it would install belongs in the image. Per-trial setup goes at the start of
  `run()`.
- `run()` stays thin: run `seed.py`, then `run.py`, with `self.exec_as_agent(environment, command=...,
  env=..., cwd=..., timeout_sec=...)` (or `exec_as_root`). `agents[].env` reaches every command the agent
  runs (not the verifier). Use `self.model_name` so `--model` works.
- A non-zero exit raises, so have the driver exit 0 whenever it produced an outcome (including `failed` /
  `timeout`) and non-zero only for harness errors.
- `populate_context_post_run(context)` runs after logs are synced to the host: parse `result.json` into the
  context, and validate a reconstructed `trajectory.json` with `harbor.models.trajectories.Trajectory`
  (log, don't fail, if it doesn't parse).
- Import only `harbor` and the standard library: the class runs in the Harbor process.
- The verifier is `tests/test.sh`, uploaded only at verification time; it writes the reward (e.g. `1`) to
  `/logs/verifier/reward.txt`. Secrets for an LLM-graded verifier go in `[verifier.env]` as `${VAR}`s.

Moving such a dataset to `eval.py`: `agent.py`'s `run()` becomes `setup()` + `run()`, `verify.py` becomes
`verify()`, `harbor.yaml`'s `images` and agent `env` move to `image` / `secrets` / `env`, and the image
stops copying the scripts (`seed.py` stays, run with `ctx.container.run_file`).

## Checklist
- [ ] Entry point, inputs and terminal state identified; no production code changed (or a change agreed
      with the user)
- [ ] Layout as in "Directory structure": `evals/<dataset>/agent/eval.py` with one `EvalRunner`,
      everything specific to the dataset in its own `agent/` (eval.py, optional seed script and
      Dockerfile), tasks under `evals/<dataset>/tasks/`, `evals/.gitignore` with `*/tasks/*` and
      `!*/tasks/dataset.toml`
- [ ] `tasks/dataset.toml` has `[dataset] name = "<org>/<name>"`; every `task.toml` has a unique
      `[task] name = "<org>/<case>"` (`lmnr-cli` writes it for new cases)
- [ ] One image from `eval.py` (named variants on the runner if a case needs another system), referenced
      by `docker_image`; per-case data in `environment/`; no per-task Dockerfile
- [ ] A harness several datasets share is one runner in `evals/_shared/`, subclassed in each `eval.py`
      with its differences written as explicit extensions of the parent's attributes
- [ ] The image goes from rarely to often changing layers (lockfile installs, then code) and adds
      specific paths from the live tree, nothing from `evals/`; never `COPY . .`
- [ ] Real services seeded for as much of the stack as possible, then mirrors; any mock is a last resort
      (HTTP mocks through the app's base-URL setting, or function mocks of observed functions with a
      recent enough Laminar SDK in the app, declared `async` in TypeScript if callers expect a Promise; data in
      `tests/fixtures/`, graded with `ctx.mocks.calls`) and reported to the user; schemas created as the app's own boot does (or
      inferred from code, minimal, and flagged)
- [ ] Seed data looks current and real: timestamps, ids, names
- [ ] Databases and caches are declared `services` at production's versions, with the app's own
      migrations; `setup()` seeds and starts the app with `ctx.container.service` and a `ready` check;
      `run()` passes `ctx.model`,
      records the outcome (including `failed` / `timeout`) with `ctx.set_result(...)`, and raises only for
      harness errors
- [ ] `verify()` grades from the terminal state and `ctx.case.expected()`; reference answers only in
      `tests/`; an LLM judge only for what can't be checked exactly
- [ ] Every secret is in `secrets` (and in `env` only if the app needs it); `.env.example` lists them;
      `.env` and `evals/*/jobs/` are gitignored; `evals/README.md` says how to run and how to add a case
- [ ] Every `instruction.md` is the driver's input or describes the case's scenario
- [ ] `lmnr-cli eval run` passes locally from the repo root; each run shows up as a new evaluation in the
      dataset's group (or the `--group` you passed)
- [ ] Tasks pushed with `lmnr-cli eval dataset push`; `tasks/dataset.toml` committed, the task folders not
- [ ] A positive and a negative case behave as expected; a flipped reference answer scores 0
- [ ] Trials show the agent's LLM and tool calls: the app is instrumented with Laminar and receives
      `LMNR_TRAJECTORY_PATH`, or `collect()` writes a reconstructed `trajectory.json` (and the user knows
      what it lacks)
- [ ] Runs against the real model (or a stub the user asked for, reported)
- [ ] Single container per task, so `lmnr-cli eval run --remote` works without changes
