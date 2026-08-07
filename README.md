# preloop — agent skill

Agent skill for **Preloop**: GitHub Actions-compatible CI that runs each job in an
isolated microVM (smolvm/libkrun), with a pause-and-retry debug loop for agents.

Teaches agents how to run workflows locally, inspect paused failed jobs, sync
fixes into the live VM and retry from the failed step, manage secrets and GitHub
credentials, and when a dirty retry is not a clean verdict.

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
```

## Layout

```
skills/preloop/SKILL.md   # the skill
```

The skill targets the current Preloop CLI (`crates/preloop-cli/src/main.rs` in
the [preloop repo](https://github.com/preloopdev/preloop)); commands are
verified against the code, not the aspirational docs.

## Links

- [Preloop product](https://github.com/preloop/preloop)
- [smolvm — microVM runtime](https://github.com/preloopdev/smolvm)
- [Agent Skills specification](https://agentskills.io)
