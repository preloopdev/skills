---
name: preloop
description: 'Run and debug GitHub Actions workflows on the hosted preloop platform — jobs execute in isolated microVMs with per-second billing. Use when the user asks to sign in, run or plan a workflow, submit CI, debug a failed job, retry a failed step in its live VM, open a shell in a preserved VM, delegate a scoped credential to an agent, manage secrets, or work with the preloop CLI. Triggers: "preloop ci", "run this workflow", "CI failed", "debug the failure", "retry the step", "agent loop", "preloop login".'
---

# Preloop

Preloop is GitHub Actions-compatible CI where every job runs in an isolated
microVM, billed per second. The control plane is **hosted** — the agent talks
to `PRELOOP_URL` over a bearer token and never administers an engine. The same
CLI can also self-host (`preloop serve`), but the default path is the hosted
platform.

Source of truth for this skill: the CLI at `crates/preloop-cli/src/main.rs`
(current subcommands only — verify a command exists before using it) and
`docs/control-plane/`.

## Non-negotiables

- **Hosted session, not a local engine.** `preloop login` (RFC 8628 device
  flow) + `preloop org use <ORG>`. Do not ask the agent to handle the GitHub
  credential or a user session; there is no engine administrator credential
  on hosted preloop.
- **Delegated tokens are short and narrow.** `preloop agent exec` yields a
  session whose permissions are the intersection of the user's grants, the
  agent's ceiling, and requested scopes; max TTL 15 minutes. Never mint or
  pass a broader credential than needed.
- **`ci-agent` cannot approve itself.** It can create/read runs, cancel its
  own runs, control its own debug sessions, and read artifacts — it cannot
  use secrets, use OIDC, publish runs, approve environments or forks, or
  administer agents. A refusal for out-of-scope action is expected; do not
  retry it with a broader credential.
- **Push-back is human-owned.** Agents do not use `preloop ci push` or the
  auto-PR path. A run is evidence for the person; the person publishes.
- **Dirty retry is not a clean verdict.** A fix proven inside a paused VM
  must be confirmed by a clean run from a fresh VM before calling the job
  fixed.
- **Secrets are reference-based.** `preloop ci secrets list` returns names,
  tiers and bindings — never values.

## Sign in

```sh
preloop login --platform-url https://<hosted-control-plane>
preloop whoami
preloop org use org_01H...
preloop doctor        # verify credential/config
```

Scripts and agents use `PRELOOP_URL` + `PRELOOP_TOKEN` instead of a session.

## Agent credentials (human sets up)

```sh
preloop agent create ci-bot --roles ci-agent
preloop agent credentials ci-bot
preloop agent login --agent ci-bot --oidc-token "$OIDC_ID_TOKEN"   # or --cid/--assertion (Ed25519, single-use)
preloop agent exec --agent ci-bot --scopes runs:read,artifacts:read --ttl 15m -- <command>
preloop agent grant-job RUN_ID/JOB --agent ci-bot                  # task grant on one live job
```

## Core loop

```text
preloop ci plan -f <workflow> --json   # expanded job DAG + matrix, no execution
preloop ci run -f <workflow>           # snapshot working tree, run; TTY runs pause on failure
preloop ci status --json               # active/recent runs
preloop ci logs RUN_ID --job <job> [--step N] [-f]
preloop ci debug --json RUN_ID         # paused session as JSON (agents/scripts)
<edit source on the host>
preloop ci debug RUN_ID --verdict retry --sync --from <step>
preloop ci debug RUN_ID --verdict continue|abort
preloop ci rerun RUN_ID --failed       # replays captured tree — for flakes, not source fixes
```

Interactive `preloop ci run` **pauses** at a failed step and holds the VM.
Non-interactive runs (`--detach`, piped stdout, agents) never pause; pass
`--debug` to opt into a debug session, or `--preserve-on-failure` to keep the
VM for `preloop ci shell`.

### Running checks before a push

- Commit first where possible — `run` snapshots the working tree, but a
  commit makes the tested tree explicit.
- Select only check jobs (test/lint/build). Skip jobs that deploy, publish,
  read secrets beyond `GITHUB_TOKEN`, or have `write`/`write-all` token
  permissions. `--job` pulls in its `needs:` closure — inspect every job in
  it.
- `pull_request` events need `--base <ref>` for `branches:`/`paths:` filters
  and `--payload <file>` when conditions read PR fields — otherwise skipped
  steps can fake green.
- `preloop ci rerun --failed` replays the *captured* tree — good for flakes,
  useless for testing a source fix. A fix needs a fresh `run`.

## Move a repository from GitHub Actions

Run `preloop ci migrate guide` and follow it exactly. `preloop ci migrate apply`
is one end-to-end migration with three phases: workflows (copy into
`.preloop/workflows/` and open the pull request), secrets (the GitHub OIDC
import after gate `set_secret`), and OIDC (per-provider trust check with the
per-organization issuer and exact snippet, after gate `configure_trust`).
`plan --json` and `verify --json` report all three phases; `--skip-secrets` /
`--skip-oidc` leave one for later. Stop at every human gate it names; never
push, disable a workflow, import a secret, or add a cloud trust without
approval.

### Right-size each job first

Pick a runner size per job from its GitHub Actions history, not by guessing.

1. Collect the last 30 successful runs of each workflow:
   `gh run list --workflow <file> --status success --limit 30 --json databaseId`,
   then `gh api repos/<owner>/<repo>/actions/runs/<id>/jobs` for each job's
   `started_at`, `completed_at` and `labels`.
2. For each job, compute the median and p90 duration and note any run that
   ended with exit code 137, `Killed` or an out-of-memory message.
3. Size it:
   - **Smallest (2 vCPU):** median under 2 minutes, or lint, format, typecheck
     and other single-threaded work.
   - **Middle (4 vCPU):** the default for builds and tests that use some
     parallelism.
   - **Largest (8 vCPU):** median over 10 minutes *and* the tool actually uses
     the cores (`cargo`, `make -j`, Gradle, `pytest -n`, Jest/Vitest workers,
     `nproc` in the script), or any memory failure in the history.
   A job that does not parallelise gets no faster on more cores, only dearer.
4. Show the human one row per job: current label, median, p90, proposed size
   and the reason. Put the sizes in the migration diff only after approval.
5. After a week on preloop, compare durations. If a size step cut the job's
   time by less than a third, step back down: cost is size × time.

## Command reference (`preloop ci`)

```text
preloop ci run [-f WF] [--job ID] [--event T] [--payload F] [--base REF]
               [--secret NAME=V] [-d|--detach] [--debug] [--preserve-on-failure]
preloop ci plan [-f WF] [--json]
preloop ci status [RUN_ID] [--json]
preloop ci logs [RUN_ID] [--job J] [--step N] [-f]
preloop ci cancel [RUN_ID]
preloop ci rerun [RUN_ID] [--failed | --job J]
preloop ci push RUN_ID                 # human path: push + PR + check runs
preloop ci manifest RUN_ID [--job J] [--verify] [--dsse --export F]
preloop ci debug [RUN_ID] [--json] [--verdict retry|continue|abort] [--sync]
               [--from STEP | --from-start] [--revert none|untracked|all]
               [--export] [--takeover]
preloop ci shell [last-failed]         # PTY into a preserved VM
preloop ci dap RUN_ID                  # DAP debug adapter session
```

Secrets:

```text
preloop ci secrets list [--repo o/r | --org O]      # names/tier/bindings, never values
preloop ci secrets set NAME [--stdin|--from-file F|--from-env V]
                      [--repo o/r [--env E] | --org O]
                      [--branch G] [--workflow P] [--job K] ...   # bindings
preloop ci secrets scope NAME ...                   # replace bindings only
preloop ci secrets delete NAME
preloop ci secrets import [--repo o/r] [--secrets NAME...] [--scope repo|env|org]
                          [--push]                  # value-free GitHub OIDC import
preloop ci migrate plan|apply|verify [--skip-secrets] [--skip-oidc] [--json]
preloop ci migrate guide                            # full agent playbook
```

## Consume the event stream

`GET /api/v1/runs/RUN_ID/events.ndjson` with the session bearer returns a
snapshot then live NDJSON: `run_accepted`, `run_status`, `job_status`, `log`,
`annotation`, `job_completed`, `check_run_created`. Reconnect + de-duplicate
— a new connection starts from a fresh snapshot. Point-in-time:
`preloop ci status RUN_ID --json` or `GET /api/v1/runs/RUN_ID`.

## Debug sessions over HTTP

`GET/POST /api/v1/debug/sessions[/{id}[/lease|/events|/operations|/audit]]`.
One mutating lease at a time; operations carry `request_id` +
`expected_version` (idempotent, stale versions rejected). `retry_from` takes
a zero-based `step_index`. Pause credit is capped at 20 minutes per job —
release the lease when done so unattached time doesn't bill as pause.
`preloop ci debug --takeover` replaces a stranded lease.

## Failure classification

| Classification | Action |
|---|---|
| `test_failure` / `compile_failure` | edit source, retry step |
| `missing_secret` | bind/store it via `ci secrets`; don't work around |
| `network_blocked` | inspect egress policy |
| `cache_miss` | continue, or warm the cache |
| `resource_oom` / `disk_quota` | raise resources or fix the test |
| `timeout` | optimize or raise the timeout |
| `infra_failure` | retry job or report a bug |
| `fidelity_unsupported` | record the gap, don't fake it |

## Environment

| Variable | Meaning |
|---|---|
| `PRELOOP_URL` | control-plane base URL (hosted default; `http://127.0.0.1:9090` for a local engine) |
| `PRELOOP_TOKEN` | bearer for API calls (session token or delegated agent token) |
| `PRELOOP_HOME` | state dir; default `~/.preloop` |

Local/self-host knobs (`PRELOOP_RUNNER_*`, `serve`) apply only when running
your own engine — not part of the hosted agent path.

## Verification rule

Never report "fixed" on a dirty retry alone. The verdict needs a clean run:
fresh VM, strict network/secrets/cache policy. And a paused-session success
is evidence for the user — pushing and PRs stay human-gated.
