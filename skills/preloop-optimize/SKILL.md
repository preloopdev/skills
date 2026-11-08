---
name: preloop-optimize
description: 'Measure and reduce CI cost on preloop — path filters and concurrency, tool-native cache protocols on the shared CAS (GOCACHEPROG, sccache WebDAV, REAPI, Turbo/Nx/Gradle HTTP), observed-input step reuse with explain, affected-test selection through framework adapters (pytest, Go, Vitest, Jest, Playwright, Bun, nextest), flake triage, DAG and matrix sizing, matrix dedup, per-second billing. Use when the user asks to make CI faster, cut CI cost, reduce queue time, set up caching, select affected tests, find flakes, or prune unnecessary jobs. Triggers: "optimize CI", "CI is slow", "CI costs too much", "cache my builds", "only run affected tests", "which jobs can I skip", "flaky tests".'
---

# Preloop CI Optimization

Make a workflow do less work — then prove it on the next run. Optimization on
preloop is not changing the YAML and hoping; it is reading what a run actually
cost (per job, per step, per test, per cache restore) and removing the work
whose outcome cannot change, with the platform's own accounting as the check.

## Non-negotiables

- **No unverified claims.** An optimization exists only after two comparable
  runs show it: same workflow, same branch class, measurable delta — wall
  time, compute seconds, cache hit rate, or selection rate. Never report a
  "win" from a diff.
- **Skipped ≠ free.** A skipped test or reused step is only an optimization
  when the skip is *correct* — backed by a dependency map or an observed
  input set. Never narrow a safety rule to inflate the savings number.
- **Do not double-cache.** Tools that already do incrementality (Gradle,
  Cargo, Bazel, sccache-covered toolchains) are *refused* by the reuse
  layer with an `external_effect`/`unproven_determinism` barrier — that is
  the right answer, not a miss. The win there is keeping the tool's own
  cache warm, not wrapping the step in another cache. Fixing this is
  regression, not optimization.
- **Reuse is bounded by input breadth.** A step whose observed inputs are
  "the whole repo" can only reuse on non-code commits. Measured data: a
  prettier step reading 2 213 files reused 1/5 commits; an eslint step
  reading 227 reused 2/5. If a step re-executes, read `explain` — check
  whether the reason is a changed input (correct) or a barrier that can be
  removed (contract fix).
- **Barriers are findings, not misses.** Every refusal to reuse names a
  `Barrier`: `external_effect`, `filesystem_write`, `volatile_input`,
  `incomplete_observation`, `unsuccessful_exit`, `forced_execution`,
  `cost_policy`, `unsupported_*`, `unproven_determinism`,
  `cache_unavailable`. Some are removable (declaring state trees), most are
  honest. Report each for what it is.
- **Never cancel in-flight deploys.** `cancel-in-progress: true` is safe on
  build/test groups; cancelling a deploy mid-flight can leave an
  environment half-released. Gate concurrency by job, not by workflow
  blanket.
- **Quarantine is metadata.** A quarantined flaky test still runs and its
  failure still fails the check. It never suppresses, skips, or reclassifies
  — it marks the test as known-flaky so a failure budget can be applied
  deliberately.
- **Selection is fail-safe, not predictive.** A test is skipped only when
  its detected dependency map is complete, nothing in it changed, it hit no
  barrier, and it last passed. New, failed, flaky, and uncertain tests
  always run. No statistics, no ML prediction — detected dependencies and
  recomputed digests.

## Step 1 — Measure before touching anything

Establish the baseline from the run record, not from anecdotes.

- `plp ci status --json` / `GET /api/v1/runs/{run_id}` — per-run
  wall time and job conclusions.
- `GET /api/v1/runs/{run_id}/events` (or `events.ndjson`) — per-job and
  per-step start/finish; compute job and step durations.
- `plp ci logs --job <job> --step <n>` — where the seconds went inside
  the slowest steps (restore, install, build, test).
- `GET /api/v1/runs/{run_id}/manifest` and
  `.../jobs/{job_id}/manifest` (+ `manifest-facts`) — the signed record of
  what each job ran, per-step reuse decisions, and the inputs the step
  observed.
- `GET /api/v1/tests` and `GET /api/v1/tests/{key}/explain` — per-test
  duration, outcome, selection decision, and the dependency evidence behind
  it.
- `GET /api/v1/scheduler/history` — queue wait vs. execution time; a job
  that waits 90 s for a warm slot and runs 10 s has a scheduling problem,
  not a compute problem.
- `plp ci plan` (dry-run of a workflow submission) — the expanded job
  DAG and matrix before paying for a run.

Report the baseline as a table: job | wall_s | queue_s | top step | step_s |
cache hit? | tests run/skipped.

## Step 2 — Remove runs that should never start

Cheapest seconds are the ones never billed.

- **`paths:` / `paths-ignore:`** on `push`/`pull_request` — a docs-only
  commit should not trigger the test workflow.
- **`concurrency:`** group + `cancel-in-progress: true` on `push` to
  non-default branches and PRs — a newer commit supersedes the in-flight
  run. Scope the group per-workflow + ref
  (`${{ github.workflow }}-${{ github.ref }}`). Never on deploys.
- **Redundant triggers** — `push` *and* `pull_request` on the same branch
  runs every PR commit twice. Pick one per lifecycle stage.
- **Submodules / fetch-depth** — `actions/checkout` defaults to full
  history on some paths; `fetch-depth: 1` unless the job needs history
  (changelog, `git describe`). Preloop's checkout-object cache already
  dedupes blobs — do not also ship a hand-rolled git cache on top.

## Step 3 — Warm the caches, one CAS underneath

preloop hosts one content-addressable store and speaks the cache protocols
toolchains already have. Do not invent a cache; plug the tool into the CAS
adapter it was built for.

| Toolchain | Protocol | preloop surface |
|---|---|---|
| Go 1.24+ | `GOCACHEPROG` (JSON Lines) | helper binary → CAS |
| sccache | WebDAV / S3 / Redis | WebDAV endpoint → CAS |
| Bazel / Buck2 | REAPI (CAS + ActionCache + ByteStream) | REAPI services → CAS |
| Turborepo / Nx / Gradle | HTTP remote/build cache | HTTP endpoint → CAS |
| Cargo | **none** — `target/` fingerprints only | CAS snapshot/restore pointer |
| pip / npm / bun / vendor dirs | `actions/cache` protocol | cache API → CAS |

Rules:

- Key on *content*, not on mtime — checkout bumps mtimes; the incremental
  layer deliberately ignores them. A same-size same-mode content change a
  step only stat()s is the accepted residual risk.
- Scope keys to what the step reads — over-broad keys (OS only, no lockfile
  digest) thrash and poison across PRs; over-narrow keys never hit.
- `actions/setup-*` and `rust-cache` write into tool-native locations; on
  preloop they persist via the host CAS. Keep them; do not wrap them in a
  second `actions/cache` layer that serializes the same bytes.
- Cache restore is counted in step time — a 20 s restore for a 5 s task is
  negative. Measure before keeping a cache.

## Step 4 — Make tests select themselves

Test Intelligence is the multiplier on suites: detected inputs per test →
skip the ones the change cannot reach.

- Adapters exist for **pytest, Go, Vitest, Jest, Playwright, Bun, and
  nextest** — each detects real reads, not declared ones.
- Selection reasons to read on `GET /api/v1/tests/{key}/explain`:
  `unaffected`, `dep_changed`, `module_changed`, `unknown_test`,
  `previous_failure`, `flaky`, `barrier`, `test_removed`,
  `sensitive_path_changed`, `forced_full_run`.
- **Sensitive-path rule:** lockfiles, test config, package manifests,
  CI definitions, and `.env` files run the *entire* session — a change to
  `package-lock.json` cannot be proven narrow.
- **Flakes:** confirmed only by a pass and a fail on identical input
  digests with no barrier. A confirmed flake always runs; quarantine is
  a label for the budget, never a verdict change.
- If a suite shows low selection rate, the cause is almost always a broad
  input surface (test fixtures globbing the whole repo, a shared
  `conftest.py` reading the world). Narrowing fixtures raises the rate more
  than any CI setting.

## Step 5 — Reuse steps correctly, then stop

- A step is reused only when its observed input set validates unchanged.
  `GET /api/v1/runs/{run}/steps/{step}/reuse` shows the decision; the
  manifest records `explain`.
- Removable barrier: a step that writes to its own input set
  (`incomplete_observation`) can become reusable by declaring the mutated
  paths `state_trees` *and* excluding them from inputs — the django finding
  (`__pycache__`, `.pytest_cache`) is the template. Everything else —
  daemons, sockets, nondeterminism — stays executed; that is correct.
- Process-level reuse (skip a `pytest` invocation inside a step) is the
  documented next lever; treat the ~99 % of re-executed step compute it
  projects as skippable as a ceiling estimate, not a feature you can call
  today.

## Step 6 — Structure the DAG and the matrix

- Parallelize what does not depend: lint, typecheck, test, build run
  together; only `deploy` gets `needs:`. Sequential `needs:` chains are
  the most common self-inflicted slowdown.
- Matrices: preloop spawns each cell on its own microVM, billed per
  second — matrix cells are cheap to fan out but not free. Fan out only
  where coverage adds value; prune redundant OS×version cells that run
  identical test bodies.
- `fail-fast` on the matrix cancels siblings on first failure — keep it on
  for PR feedback, off for a nightly full-coverage run.
- Long suite → split with `matrix` + the framework's native shard flag
  (`pytest --shard`, `jest --shard`), not N hand-copied jobs.

## Step 7 — Prove it

1. Rerun the same workflow on a comparable commit.
2. Diff: run wall time, per-job seconds, queue wait, cache hit/miss lines,
   tests run vs skipped, reused vs executed steps.
3. Accept the change only if the number moved *and* every skip is
   explain-backed. If a step you expected to reuse executed, read the
   barrier before touching the contract.

Anti-targets that do not count as proof: faster YAML, "should be cached
now", a rerun on an unrelated commit, a local `act` run, timing the same
step twice inside one job.

## Report format

| Change | Evidence | Baseline | After | Δ | Correct-skip check |
|---|---|---|---|---|---|
| concurrency group added | run events | 2×8m runs | 1 run | −50 % | n/a |
| GOCACHEPROG → CAS | step logs | 45 s build | 9 s | −80 % | hits logged |
| pytest adapter on | tests API | 9 579 ran | 24 ran | −99.7 % | `unaffected` w/ maps |

Plus a **do-not-optimize** list: steps that correctly re-execute (barrier
named), tools whose own incrementality beats ours, and caches whose
restore costs more than the work.
