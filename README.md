# preloop — agent skills

Agent skills for **Preloop**: GitHub Actions-compatible CI that runs each job in an
isolated microVM (smolvm/libkrun), with a pause-and-retry debug loop for agents.

| Skill | What it teaches |
|---|---|
| `preloop` | Run workflows locally, inspect paused failed jobs, sync fixes into the live VM and retry from the failed step, manage secrets and GitHub credentials, and when a dirty retry is not a clean verdict. |
| `preloop-secure` | Audit and harden workflow security — dangerous triggers and pwn requests, template injection, secret bindings, artifact/cache poisoning, action pinning with `actions.lock`, OIDC trust, runner hijack — and apply the preloop control that closes each finding. |
| `preloop-optimize` | Measure and reduce CI cost — path filters and concurrency, tool-native cache protocols on the shared CAS, observed-input step reuse, affected-test selection, flake triage, DAG and matrix sizing — and prove the savings on the next run. |

## Install

```sh
# All agents detected on this machine
npx skills add preloopdev/preloop-skill

# A specific agent
npx skills add preloopdev/preloop-skill -a claude-code

# Global scope (available across projects)
npx skills add preloopdev/preloop-skill -g -y
```

## Use without installing

```sh
npx skills use preloopdev/preloop-skill@preloop
npx skills use preloopdev/preloop-skill@preloop-secure
npx skills use preloopdev/preloop-skill@preloop-optimize
```

## Layout

```
skills/preloop/SKILL.md           # run and debug CI
skills/preloop-secure/SKILL.md    # audit and harden workflows
skills/preloop-optimize/SKILL.md  # measure and reduce CI cost
```

The skills target the current Preloop CLI (`crates/preloop-cli/src/main.rs` in
the [preloop repo](https://github.com/preloopdev/preloop)); commands are
verified against the code, not the aspirational docs.

## Links

- [Preloop product](https://github.com/preloopdev/preloop)
- [smolvm — microVM runtime](https://github.com/preloopdev/smolvm)
- [Agent Skills specification](https://agentskills.io)
