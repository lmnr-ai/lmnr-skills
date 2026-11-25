# Simulated SaaS in env evals (Rystic)

Some services a product calls can't run in the eval container: Slack, Linear and GitHub are closed SaaS.
Instead of hand-writing a mock, run a [Rystic](https://www.rystic.ai/docs) simulator for them inside the
trial's container. A simulator serves the vendor's API on localhost, keeps state across calls, can be
seeded with the case's data, and records every call the app made, which the verifier can grade. This
file covers when to use one, how to boot and seed it, and how to grade with it. Build the rest of the
harness as [env-evals.md](env-evals.md) describes.

A simulator is still a stand-in, not the real service. It is better than a hand-written mock (it
reproduces the vendor's behaviour, measured against the live API), but it counts as a mock under
env-evals.md's Principles: **tell the user about it**, which simulator, and which calls it doesn't cover.

## When to use one

Use a simulator when the path under test calls Slack, Linear or GitHub (the simulators every Rystic plan
includes) and you would otherwise mock that service. Don't use one when:

- **The app reaches the service through the vendor's hosted MCP server** (`mcp.linear.app`, Slack's or
  GitHub's remote MCP). A hosted MCP server can't be pointed at a simulator. Say so; the case either
  mocks the MCP tools at the app's own config or isn't reproducible.
- **The trace uses calls the simulator doesn't serve.** Compare the methods in the trace's tool calls with
  the simulator's routes (`GET $URL/_rystic/routes`). An unserved operation answers an error (Slack:
  `501`, "The Slack simulator doesn't support ... yet"). If the case depends on one, don't build it on the
  simulator; report the gap.

| Service | Product | Covers | Client setup |
|---|---|---|---|
| Slack | `slack-twin` | Web API: conversations, messages, users, reactions, files, views, search, Events API (Socket Mode or request URL). No `admin.*`, Enterprise Grid, `files.upload`, `rtm`, `oauth` | base URL `$URL/api/` (`slack_sdk` `base_url`, `@slack/web-api` `slackApiUrl`), any `xoxb-`/`xoxp-` token |
| Linear | `linear-twin` | GraphQL `POST /graphql`: issues, comments, labels, relations, attachments, documents, projects, milestones, webhooks; teams, users, states, cycles read-only. No subscriptions, no whole-schema introspection | `@linear/sdk` `apiUrl: $URL/graphql`, any non-empty `Authorization` (a `lin_api_` key bare, not as `Bearer`) |
| GitHub | `github-pulls-twin` | REST (pulls, reviews, issues, comments, labels, checks, statuses, contents, git data, branches, releases, Apps, webhooks), the GraphQL subset `gh` uses, git over HTTP. No repo/org admin, code search, Actions beyond workflow runs | Octokit `baseUrl: $URL`, PyGithub `base_url`, any token |

Each simulator documents itself: `GET $URL/_rystic/docs` (sections), `$URL/_rystic/docs/<section>`, and
`rystic docs <product>` once the CLI is installed. Read the seeding section of the one you use.

## Where it runs and how it is licensed

The simulator runs inside the trial container. Only its download and its license check-ins (at start, every
30 seconds, at exit) leave the container; the app's API calls never do.

- **`lmnr-cli eval run --remote`:** when the Laminar deployment provides Rystic, every trial sandbox already
  has `RYSTIC_TOKEN`, `RYSTIC_RELEASE_ENDPOINT` and `RYSTIC_LEASE_URL` in its environment. They are a
  short-lived token for that trial; Laminar adds its own Rystic key on the way out. Don't declare them in
  `harbor.yaml` or pass them with `--env`, which would replace them. A trial may start up to 20 simulators.
- **Local runs:** the trial needs your own Rystic key (`rys_...`, [rystic.ai/account/api-keys](https://www.rystic.ai/account/api-keys)).
  Pass it for that run only: `lmnr-cli eval run --env RYSTIC_TOKEN`, with `RYSTIC_TOKEN` in your shell or
  `.env`. Never write it into `harbor.yaml`, the image or a task.

If neither applies, `rystic pull` exits 11 (`invalid_token`) or 3 (no key): the case can't use a
simulator here.

## Install the CLI in the image

Install the `rystic` CLI in `agent/Dockerfile`, pinned. The installer needs `curl` and `tar` and puts the
binary in `/usr/local/bin` when run as root. It needs no key:

```dockerfile
RUN curl -fsSL https://www.rystic.ai/install.sh | RYSTIC_VERSION=0.0.93 RYSTIC_NO_LOGIN=1 CI=1 sh
```

Don't download simulators during the build: that needs a key, and a key in a build step ends up in the
image. `seed.py` downloads them when the trial starts (`rystic pull`, about 25 MB each, from GitHub).

## Boot and seed in `seed.py`

Start each simulator before the app, empty, with the case's data as its boot seed, and give the app its URL:

```bash
rystic pull slack-twin --version 0.0.29          # pin it like any other service
rystic run slack-twin -d --name slack --port 0 --json \
  --seed /case/slack.json --clock 2026-11-25T09:00:00Z > /tmp/slack.json
SLACK_URL=$(jq -r .data.url /tmp/slack.json)     # http://localhost:<port>
curl -fsS "$SLACK_URL/_rystic/version" >/dev/null # readiness
```

- **Pull first, then `run --json`**, or the pull's output lands in the JSON. `-d` returns once the simulator
  serves. Copy the instance's log (`.data.log`) into `/logs/agent/` at the end of `run.py`.
- **Seed from the trace, not from a built-in scenario.** The built-in scenarios (`design_studio`,
  `linear_sprint_board`...) are fictional companies with fixed dates; the case needs what the agent read in
  production. Put it in the case's data (`environment/slack.json`) as a register map,
  `{"<register>": {"<key>": {record}}}`, in the vendor's own object shapes. `GET $URL/_rystic/registers`
  lists each simulator's registers and key grammar, e.g. Slack's `state_message` is keyed `channel:ts`.
  `--seed <file>` applies it at boot; `POST $URL/_rystic/seed` merges more into a running simulator.
- **Nothing is derived for you.** Records aren't validated and no field is computed. Seed what the API would
  have stored: on Slack, a `state_membership` row for every channel the bot reads (else `not_in_channel` /
  `channel_not_found`), a thread parent's `reply_count` / `reply_users` / `latest_reply`, reactions both
  on the message and as `state_reaction`; on GitHub, a PR's repository, both branches, their commits and
  the author; on Linear, the team, workflow states and users an issue points at.
- **The acting identity is keyed `member`** in the user register (Slack's `state_user`, Linear's
  `state_user`). Key the bot or user the app acts as there, or every token acts as the simulator's default
  user. GitHub picks the caller per request: `X-Rystic-Actor: <login>` or a token bound to a user; read its
  "Who is calling" docs section.
- **Timestamps.** The clock starts at boot time unless `--clock` sets it. Rebase the case's timestamps the
  way env-evals.md's "Seed data the agent can't tell is a test" says, and boot the clock at the case's
  "now", so new records (a Slack `ts` comes from the clock) fall after the seeded ones.
- **Several services:** one `rystic run` per product, each with its own `--name`.

## Point the app at it

Use the app's existing configuration, as for any service (env-evals.md, "Services"):

- **A base-URL setting exists** (most SDK wrappers have one, or read it from env): set it to the
  simulator's URL and pass a placeholder token of the vendor's shape (`xoxb-...`, `lin_api_...`,
  `ghp_...`). Make placeholders long and distinctive: Harbor redacts env values from its output.
- **The vendor host is hard-coded:** boot with `--intercept` and start the app under
  `rystic exec -- <app command>`. It sets `HTTPS_PROXY` and CA variables so the vendor's real hostnames
  reach the simulator. A start script that uses `env -i` must run `rystic exec` inside it, or forward
  what `rystic env` prints. Clients that ignore the proxy variables reach the real vendor instead: Node
  before 22.21 for `fetch`, Java's `HttpClient`, SDKs with their own HTTP agent. See
  [rystic.ai/docs/intercept](https://www.rystic.ai/docs/intercept).
- Start the app without the `RYSTIC_*` variables in its environment (the `env -i` start script already
  does), and name the URL variable the way the app's config does (`SLACK_API_URL`), not after Rystic.

## Drive and grade

- **Wait for the bot's turn to end.** For a Slack bot driven by an event, `GET $URL/_rystic/settle?quiet_ms=<ms>`
  returns once nothing awaits the bot's ack and no API call landed for `quiet_ms`. Set `quiet_ms` above the
  longest pause inside a turn (a model call). Otherwise wait for the entry point's own terminal state, as
  env-evals.md says.
- **Export what the app did** at the end of `run.py`, including on `failed` / `timeout`:
  `GET $URL/_rystic/calls` is every API call since boot, oldest first (`path`, `args`, `status`, `ok`,
  `principal`, `at`), and `GET $URL/_rystic/inspect` is the whole final state as a register map. Write both
  to `/logs/agent/` (e.g. `slack-calls.json`, `slack-state.json`). The verifier reads the files, so it
  works the same in `shared` and `separate` mode.
- **Grade the effect, deterministically where you can:** a message posted to the right channel, no
  write to an issue the case says to leave alone, no forbidden method called, a PR comment that names the
  failing check. Combine it with the LLM judge only for the judgement part ("Choose the check from the
  failure" in [evals-from-clusters.md](evals-from-clusters.md)).

## Troubleshooting

| Symptom | Cause |
|---|---|
| `rystic pull` exits 11, `invalid_token` | No Rystic access in this trial: a local run without `--env RYSTIC_TOKEN`, or a deployment without Rystic |
| exits 3, no key | `RYSTIC_TOKEN` didn't reach the command (a start script with `env -i`) |
| register answers `429 too_many_simulators` | The trial started 20 simulators; reuse instances instead of restarting them |
| a control route answers `403` with `feature` | The plan doesn't include that route's feature |
| the app gets the real vendor's errors (`invalid_auth` from slack.com) | The call bypassed the simulator: a base URL not applied, or a client ignoring the proxy |
| `not_in_channel` / `channel_not_found` | The seed lacks a `state_membership` row for the bot |
