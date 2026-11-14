# preloop — agent skills

Agent skills for **Preloop**: GitHub Actions-compatible CI where every job runs in an
isolated microVM, with a pause-and-retry debug loop for agents.

| Skill | What it teaches |
|---|---|
| `preloop` | Sign in to the hosted platform, submit and plan runs, inspect paused failed jobs, sync fixes into the live VM and retry from the failed step, manage secrets, and when a dirty retry is not a clean verdict. |
| `preloop-secure` | Audit and harden workflow security — dangerous triggers and pwn requests, template injection, secret bindings, artifact/cache poisoning, action pinning with `actions.lock`, OIDC trust, runner hijack — and apply the preloop control that closes each finding. |
| `preloop-optimize` | Measure and reduce CI cost — path filters and concurrency, tool-native cache protocols on the shared CAS, observed-input step reuse, affected-test selection, flake triage, DAG and matrix sizing — and prove the savings on the next run. |

## Install

```sh
# All agents detected on this machine
npx skills add preloopdev/skills

# A specific agent
npx skills add preloopdev/skills -a claude-code

# Global scope (available across projects)
npx skills add preloopdev/skills -g -y
```

## Use without installing

```sh
npx skills use preloopdev/skills@preloop
npx skills use preloopdev/skills@preloop-secure
npx skills use preloopdev/skills@preloop-optimize
```

## Layout

```
skills/preloop/SKILL.md           # run and debug CI
skills/preloop-secure/SKILL.md    # audit and harden workflows
skills/preloop-optimize/SKILL.md  # measure and reduce CI cost
```

The skills target `plp`, the Preloop command-line client, against the hosted
platform. Verify a command with `plp --help` before using it.

## Links

- [Preloop](https://app.preloop.dev)
- [Agent Skills specification](https://agentskills.io)
