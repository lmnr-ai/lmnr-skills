# Simulated SaaS in env evals (Rystic)

Slack, Linear and GitHub can't run in the eval container. Instead of a hand-written mock, run a
[Rystic](https://www.rystic.ai/docs) simulator in the trial: it serves the vendor's API on localhost, keeps
state, boots with the case's data, and records every call the app made for the verifier. Build the rest of
the harness as [env-evals.md](env-evals.md) describes.

A simulator is still a stand-in: under env-evals.md's Principles it counts as a mock, so **tell the user**
which services the cases simulate and which calls the simulator doesn't cover.

## When to use one

| Service | Product | Point the client at |
|---|---|---|
| Slack | `slack-twin` | `$URL/api/` (`slack_sdk` `base_url`, `@slack/web-api` `slackApiUrl`), any `xoxb-` token |
| Linear | `linear-twin` | `$URL/graphql` (`@linear/sdk` `apiUrl`), any non-empty `Authorization` |
| GitHub | `github-pulls-twin` | `$URL` (Octokit `baseUrl`, PyGithub `base_url`), any token |

Don't use one when the app reaches the service through the vendor's hosted MCP server (`mcp.linear.app`
and the like: it can't be pointed at a simulator), or when the case depends on a call the simulator doesn't
serve. Check the trace's methods against `GET $URL/_rystic/routes`; unserved ones answer an error (Slack:
`501`). Report either gap instead of building the case.

Rystic documents each simulator in detail: `rystic docs <product>` lists the sections (`coverage`,
`seeding-state`, `auth`, ...) and `rystic docs <product> <section>` prints one. Read `seeding-state` for the
one you use.

## Where the key comes from

The simulator runs inside the trial; only its download and its license check-ins leave the container.

- **`lmnr-cli eval run --remote`:** when the Laminar deployment provides Rystic, each trial sandbox already
  has `RYSTIC_TOKEN`, `RYSTIC_RELEASE_ENDPOINT` and `RYSTIC_LEASE_URL`: a short-lived token for that trial.
  Don't set them in `harbor.yaml` or with `--env`, which would replace them. A trial may start up to 20
  simulators.
- **Local runs:** pass your own Rystic key for that run only, `lmnr-cli eval run --env RYSTIC_TOKEN`. Never
  write it into `harbor.yaml`, the image or a task.

`rystic pull` exiting 11 (`invalid_token`) or 3 (no key) means this trial has no Rystic access: either
neither case applies, or a start script with `env -i` dropped `RYSTIC_TOKEN`.

## Install the CLI in the image

Pin it in `agent/Dockerfile`. It needs no key. Save the installer before running it: `RUN` has no
`pipefail`, so `curl ... | sh` passes on a failed download. The last command fails the build if `rystic`
doesn't run:

```dockerfile
RUN curl -fsSL https://www.rystic.ai/install.sh -o /tmp/rystic-install.sh \
    && RYSTIC_VERSION=0.0.93 RYSTIC_NO_LOGIN=1 CI=1 sh /tmp/rystic-install.sh \
    && rm /tmp/rystic-install.sh && rystic docs slack-twin >/dev/null
```

Download simulators in `seed.py`, not in the build: the download needs the key.

## Boot and seed in `seed.py`

```bash
rystic pull slack-twin --version <pinned version>    # before run --json, or its output lands in the JSON
rystic run slack-twin -d --name slack --port 0 --json \
  --seed /tmp/slack-seed.json --clock <case's now, RFC 3339> > /tmp/slack.json
SLACK_URL=$(jq -r .data.url /tmp/slack.json)
```

- **Seed from the trace**, not a built-in scenario (those are fictional companies). The seed is a register
  map, `{"<register>": {"<key>": {record}}}`, in the vendor's own object shapes; `GET $URL/_rystic/registers`
  lists the registers and their keys. Keep it in the case's `environment/`.
- **Nothing is derived.** Seed what the API would have stored. The usual misses: a Slack `state_membership`
  row for every channel the bot reads (else `not_in_channel`), and the records a GitHub PR or Linear issue
  points at (repo, branches, commits, author; team, states, users).
- **Key the identity the app acts as under `member`** in the user register (Slack, Linear). GitHub picks
  the caller per request; see its `auth` section.
- **Rebase the case's timestamps** to now (env-evals.md, "Seed data the agent can't tell is a test") and
  set `--clock` to the same now, so records the app creates fall after the seeded ones.
- One `rystic run` per product, each with its own `--name`.

## Point the app at it

Through the app's existing config, as for any service: the simulator's URL as its base URL, a long,
distinctive placeholder token of the vendor's shape (Harbor redacts env values from its output), and the
URL variable named the way the app's config names it, not after Rystic. Start the app without the
`RYSTIC_*` variables. If the vendor host is hard-coded, boot with `--intercept` and start the app under
`rystic exec -- <command>` ([intercept](https://www.rystic.ai/docs/intercept); some clients ignore its proxy).

## Grade

- For a Slack bot driven by an event, `GET $URL/_rystic/settle?quiet_ms=<ms>` returns once its turn is over;
  set `quiet_ms` above the longest pause in a turn (a model call).
- At the end of `run.py`, including on `failed` / `timeout`, write `GET $URL/_rystic/calls` (every API call:
  `path`, `args`, `ok`, ...) and `GET $URL/_rystic/inspect` (the final state) to `/logs/agent/`, with the
  simulator's log (`.data.log`). The verifier reads those files, so it works in `shared` and `separate` mode.
- Grade the effect deterministically where you can (the right message posted, no forbidden write), and use
  an LLM judge only for the judgement part ([evals-from-clusters.md](evals-from-clusters.md)).
