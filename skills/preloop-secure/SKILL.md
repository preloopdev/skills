---
name: preloop-secure
description: 'Audit and harden GitHub Actions CI security on preloop — dangerous triggers and pwn requests, script/template injection, secret scoping and bindings, artifact and cache poisoning, action pinning and actions.lock, OIDC trust, workflow-command abuse, runner hijack. Use when the user asks to secure CI, review workflow security, fix zizmor findings, lock down secrets, set fork or deny policies, or harden a preloop deployment. Triggers: "secure my CI", "audit workflows", "zizmor findings", "pull_request_target", "secret bindings", "actions.lock", "cache poisoning", "harden preloop".'
---

# preloop-secure

Audit a repository's GitHub Actions posture against the known CI/CD attack
classes, map every finding to the preloop control that closes it, and apply the
fix — workflow edits, secret bindings, engine policy — then verify on a real run.

Attack surface reference: the GitHub Actions cheat sheet
(`0xn3va.gitbook.io/cheat-sheets/ci-cd/github/actions`) and the zizmor audit
taxonomy (`docs.zizmor.sh/audits`). Every rule below maps to one of those two.

## Non-negotiables

- **Never gate on a control that is not enabled.** `execution_protection` ships
  in `evaluate` (log-only); `token_permissions_ceiling` is inactive unless its
  table is present. Read `security-controls.md` / `SECURITY_MODEL.md` before
  claiming a control covers a finding; if it is opt-in, enabling it is part of
  the fix, not an assumption.
- **`pull_request` does not require approval for all outside collaborators by
  default** — only first-time contributors. A contributor badge is not review.
- **`github.event.*`, `github.head_ref`, `env`, and `steps/needs.*.outputs` are
  attacker-controlled when an outsider can influence them.** Any `${{ }}`
  interpolation of these inside `run:` or a `script:` block is a template
  injection candidate, regardless of quoting style.
- **`head.ref` checks out a branch; `head.sha` checks out the reviewed commit.**
  Any trust gate (label, approval, comment) followed by a `head.ref` checkout is
  bypassable — the attacker pushes after the gate.
- **Fork pull requests must never write shared state.** On preloop this means
  no stored secrets, no OIDC token, cache read-only — do not weaken it.
- **A masked value is only masked if the engine knows it.** Secrets fetched at
  runtime need `::add-mask::` before they touch the log; preloop masking covers
  raw, base64, hex and percent-encoded forms, but nothing it was never told.
- **Report evidence, not vibes.** For every finding: the file, the line, the
  trigger that makes it reachable, and whether a base-branch clone of the repo
  is affected (a vulnerable `pull_request_target` in a non-default branch runs
  its own branch's copy — patching only `main` does not fix it).

## Audit sequence

Run the categories in order — each finding feeds the next (a dangerous trigger
makes every downstream injection reachable).

### 1. Triggers and trust boundary

- `pull_request_target`, `issue_comment`, `workflow_run` run with write
  `GITHUB_TOKEN` and secrets on base-repo trust. Flag every one that checks out
  or executes PR-controlled content (`ref:`/`ref.sha`/`` refs/pull/N/merge ``,
  `git fetch`/`merge` of `head_ref`, downloading and running PR artifacts).
- `workflow_run` consumers: untrusted data still crosses through
  `github.event.workflow_run.*` context and downloaded artifacts. Both must be
  validated, never interpolated or executed.
- Check `branches:`/`branches-ignore` filters on every dangerous trigger —
  unfiltered `pull_request_target` also runs copies living in non-default
  branches.
- `workflow_dispatch`/`repository_dispatch` with secret or token inputs: inputs
  land in logs unless masked.
- Label/approval/`if:` gates: flag `github.event.pull_request.head.ref`
  checkouts behind them (see non-negotiables). `head.sha` or a reviewed merge
  SHA is the only safe ref.

### 2. Script and template injection

- `${{ }}` inside `run:` is evaluated *before* the shell runs — quoting does not
  help. Move untrusted values through `env:` instead.
- `actions/github-script` `script:` + `${{ }}` = JS injection;
  `octokit/graphql-action` `query:` = GraphQL injection; `octokit/request-action`
  `route:` = request injection. Pass values via `env`/inputs, read them inside
  the script.
- `stop-commands`/`add-mask` injection: an attacker who controls logged text can
  stop command processing. On preloop, `::add-path::`/`::set-env::` are gated
  (pentest PR 913) — do not enable `ACTIONS_ALLOW_UNSECURE_COMMANDS`.
- Deprecated commands: `set-env`, `add-path`, `set-output`, `save-state` — the
  env-file equivalents (`$GITHUB_ENV`, `$GITHUB_OUTPUT`) are the safe path.

### 3. Secrets

- Never put a secret in a `${{ }}` that feeds `run:`/`script:` directly —
  deliver via `env:` and read inside the process.
- Reusable workflows: `jobs.<id>.secrets` bundles stringified secrets; if the
  callee re-splits them the parts are unmasked. Prefer OIDC or individually
  mapped secrets over packed `creds` strings.
- Scope every secret. On preloop, bind by ref glob, workflow path, job key and
  caller chain:

  ```sh
  preloop ci secrets scope DEPLOY_KEY --repo org/repo \
    --branch 'refs/heads/main' --workflow deploy.yml --job deploy
  ```

  Conditions inside one binding AND; bindings OR across. No bindings =
  unrestricted — every production secret should carry at least one.
- Production environment secrets: environment approvals are human-only on
  preloop, so gating deploy secrets on `env + branch + job` is enforceable.
- Audit for literals: `grep -rIEn '(SECRET|TOKEN|KEY|PASSWORD)[^:]*:' .github/`
  plus `git log -p .github/workflows` — values committed then "deleted" still
  leak; they need rotation, not removal.

### 4. Actions and supply chain

- Pin every `uses:` to a commit SHA (`@sha`, 40 hex). Tags are mutable.
- Enforce with `.github/workflows/actions.lock` — on preloop the lock records
  the resolved SHAs and a workflow whose actions drift from it loses its OIDC
  token. PR 1030 (`fix/require-lockfiles`) tightens this to a hard requirement.
- Typosquatting/unclaimed namespaces, composite actions running arbitrary
  shell, JavaScript actions with attacker-reachable inputs, Docker actions
  pulling mutable images — each is a distinct finding class (zizmor:
  `unpinned-uses`, `typosquat-uses`, `impostor-commit`, `unpinned-images`,
  `known-vulnerable-actions`).
- Run `zizmor .github/workflows --persona auditor` where available; feed its
  output through this same map instead of re-deriving severity.

### 5. Cache and artifacts

- Cache scope: entries are namespaced per repository **and ref** on preloop;
  forks read and cannot write; low-trust writes land in scoped keys that cannot
  shadow base. A workflow that writes cache from PR content on a base-branch
  ref is the classic poison — find `actions/cache` save steps reachable from
  `pull_request_target` + PR checkout.
- Artifacts are mutable across runs: anything downloaded from another run
  (dawidd6/action-download-artifact, `gh run download`, `gh api`
  artifacts_url, `listWorkflowRunArtifacts`) must be validated — checksum,
  signature, or at minimum never executed directly.
- On preloop, artifacts and logs are scanned for exfiltration at finalize
  (512 MiB budget, a match is refused) — that catches leaks, not logic; the
  workflow still shouldn't run downloaded binaries.

### 6. OIDC and cloud trust

- `permissions: id-token: write` is required for OIDC; audit every job that has
  it. The cloud-side trust must pin `aud` + `sub` (repo + environment/branch),
  or any repo minting the right claims can assume the role.
- preloop OIDC tokens live 300 s and are bound to the repo's immutable numeric
  ID plus `actions.lock`; forks get none.

### 7. GITHUB_TOKEN and permissions

- Default `permissions:` should be `permissions: {}` or `contents: read` at the
  top level; grant write only per-job where needed.
- `token_permissions_ceiling` (opt-in TOML table) caps what any workflow can
  request — set `default = "read"` and explicit grants per permission so a
  workflow cannot escalate past policy.
- `pull-requests: write` + `issues: write` + `contents: write` on a job that
  runs PR content = maintainer impersonation (approve/merge/comment).

### 8. Runners

- Self-hosted runners are the biggest trust break: any workflow can reach the
  host. Non-ephemeral runners persist state across jobs — a malicious job
  implants a poisoned runner image or tooling cache.
- Registration-token disclosure lets an attacker register a runner and take
  jobs (code + secrets). Treat registration tokens as secrets; rotate on
  exposure.
- preloop's managed pool is the mitigation story: one job per microVM,
  destroyed at end, claimed warm from a paused pool — no shared daemon, no
  residue. Recommend managed runners for anything handling secrets; keep
  self-hosted pools for low-trust, secret-free work.

## Engine policy (operator side)

On the hosted platform these are API-managed policies under
`/api/v1/policies/...` (`execution-protection`, `token-ceiling`,
`namespace`, per-repo and per-environment rules); the system bearer or an
org-admin session sets them. Self-hosted engines accept the same policy in
`preloop.toml`:

```toml
[execution_protection]
mode = "enforce"            # evaluate only logs — a finding is not a fix

[[execution_protection.event_rules]]
event = "pull_request_target"
workflows = ["deploy.yml"]  # omit to match every workflow
action = "deny"

[[execution_protection.actor_rules]]
actor = "untrusted-bot"
action = "deny"

[token_permissions_ceiling]
default = "read"
contents = "read"
pull-requests = "write"
```

Also: `fork_policy` for approval-gated fork runs (24 h fail-closed on
`POST /api/v1/runs/{id}/approve-fork`), and environment rules for deploy jobs.

## Output format

One finding per row, ordered by reachability × impact:

| # | Severity | File:line | Attack class | Why it is exploitable | Fix applied |
|---|----------|-----------|--------------|-----------------------|-------------|

Then a posture delta: which engine controls the audit enabled (`enforce`,
ceiling, bindings, approval), and what remains open with the reason.

## Verify

- A fix is verified when a real run exercises it: submit the workflow
  (`preloop ci run`) and confirm the hardened path behaves —
  a denied event is skipped and logged, a bound secret does not reach an
  unbound job, a drifted action loses its OIDC token.
- `preloop ci secrets list` to confirm bindings landed; the audit log records
  the secret-release decision per job.
- Re-run `zizmor`/`actionlint` after edits; do not report a finding as fixed
  from the diff alone.
- A dirty retry inside a paused VM proves the mechanism, not the shipped
  state — the final claim needs a clean run.
