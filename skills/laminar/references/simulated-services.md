# Simulated SaaS in env evals (Rystic)

Slack, Linear and GitHub can't run in the eval container. Instead of a hand-written mock, declare a
[Rystic](https://www.rystic.ai/docs) simulator as a mirror in `eval.py`: it serves the vendor's API on
localhost inside the trial, keeps state, starts with the case's data, and records every call the app made
for `verify()`. The rest of the harness is as [env-evals.md](env-evals.md) describes; its "Mirrors" section
has the API (`Rystic(...)`, `state()`, `calls()`, `settle()`, `snapshot=True`).

A simulator is still a stand-in: under env-evals.md's Principles it counts as a mock, so **tell the user**
which services the cases simulate and which calls the simulator doesn't cover.

## When to use one

| Service | Product | Point the client at (`env` template) |
|---|---|---|
| Slack | `slack` | `{url}/api/` (`slack_sdk` `base_url`, `@slack/web-api` `slackApiUrl`), any `xoxb-` token |
| Linear | `linear` | `{url}/graphql` (`@linear/sdk` `apiUrl`), any non-empty `Authorization` |
| GitHub | `github-pulls` | `{url}` (Octokit `baseUrl`, PyGithub `base_url`), any token |

`version=` is required: `rystic list` (with a key) shows the latest, pin it and keep it, so a new twin
release never changes a running eval.

Don't use one when the app reaches the service through the vendor's hosted MCP server (`mcp.linear.app`
and the like: it can't be pointed at a simulator), or when the case depends on a call the simulator
doesn't serve. Check the trace's methods against `GET <url>/_rystic/routes` (`rystic run <product>` on your
machine, or `await self.service(Rystic).get(ctx, "/_rystic/routes")` in a trial); unserved ones answer an
error (Slack: `501`). Report either gap instead of building the case. The exception is Linear's MCP
server: start [portomatic](https://www.npmjs.com/package/portomatic) instead, from `setup()` with
`ctx.container.spawn("linear", "npx -y portomatic start --only linear", ready=...)` (Node 24+ in the
image; `npx` downloads it when the trial starts; `npx portomatic linear help` lists its endpoints,
credentials and seeding), and point the app's MCP URL at it.

Rystic documents each simulator in detail: `rystic docs <product>` lists the sections (`coverage`,
`seeding-state`, `auth`, ...) and `rystic docs <product> <section>` prints one. Read `seeding-state` for
the one you use. The CLI reads them without a key.

## Access

The simulator runs inside the trial; only its download and its license check-ins leave the container.
lmnr-cli installs the pinned Rystic CLI into the image (no key needed), and each trial downloads the
simulator.

- **`lmnr-cli eval run --remote`:** when the Laminar deployment provides Rystic, each trial sandbox
  already has its own short-lived access (`RYSTIC_TOKEN` and the proxy URLs). Don't put `RYSTIC_TOKEN`
  in `env` or `harbor.yaml`: it would replace the trial's (lmnr-cli drops a `--env RYSTIC_TOKEN`).
- **Local runs:** the user's own key, which `lmnr-cli eval run` reads as `RYSTIC_TOKEN` from the shell,
  `.env.local` or `.env` and passes to the trials (keys at https://www.rystic.ai/account/api-keys; the
  Slack, Linear and GitHub simulators are on the free plan). Without one the run fails before building
  anything (exit `19`). Never write it into the image, `eval.py` or a task. An app started with `env -i` doesn't
need `RYSTIC_*`: only the runner's `rystic` commands use them.

## Seed from the case

`seed=case_fixture("slack.json", required=True)` loads each case's `tests/fixtures/slack.json` after the
simulator starts; the agent never sees it as a file.

- **Seed from the trace**, not a built-in scenario (those are fictional companies). The seed is a register
  map, `{"<register>": {"<key>": {record}}}`, in the vendor's own object shapes; `GET <url>/_rystic/registers`
  lists the registers and their keys.
- **Nothing is derived.** Seed what the API would have stored. The usual misses: a Slack `state_membership`
  row for every channel the bot reads (else `not_in_channel`), and the records a GitHub PR or Linear issue
  points at (repo, branches, commits, author; team, states, users).
- **Key the identity the app acts as under `member`** in the user register (Slack, Linear). GitHub picks
  the caller per request; see its `auth` section.
- **Rebase the case's timestamps** to now (env-evals.md, "Seed data the agent can't tell is a test"), and
  set `clock=` (RFC 3339) to the same now, so records the app creates fall after the seeded ones.
- One `Rystic(...)` per product; give two of the same product their own `name=` and pick one with
  `self.service("<name>")`.

lmnr-python's `examples/agent-evals` (`billing` dataset) seeds a Slack workspace with one customer
message: `tests/fixtures/slack.json` in each case.

## Point the app at it

Through the app's existing config, as for any service: the simulator's URL as its base URL (`env=` on the
mirror, e.g. `{"SLACK_API_URL": "{url}/api/"}`, named the way the app's config names it, not after
Rystic), and a placeholder token of the vendor's shape in the runner's `env`. If the vendor host is
hard-coded, that is a gap to raise with the user: the mirror doesn't run Rystic's intercept mode.

## Grade

- For a Slack bot driven by an event, `await self.service(Rystic).settle(ctx, quiet_ms=...)` in `run()`
  returns once its turn is over; set `quiet_ms` above the longest pause in a turn (a model call).
- In `verify()`, `calls(ctx)` is every API call the app made (`path`, `args`, `ok`, `writes`, ...) and
  `state(ctx)` the final state. With a separate verifier, set `snapshot=True` so `collect` saves both.
- Grade the effect deterministically where you can (the right message posted, in the right thread, no
  forbidden write), and use an LLM judge only for the judgement part
  ([evals-from-clusters.md](evals-from-clusters.md)).
