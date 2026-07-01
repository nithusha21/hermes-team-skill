# Hermes Team

A [Claude Code](https://claude.com/claude-code) skill that spawns a **parallel team of
[hermes](https://github.com/) coding agents** in isolated git worktrees to work through a
`TASKS.md` file wave by wave.

Each agent runs in its own `hermes/<hex>` worktree branch, implements one task, runs
verification, and commits independently. The skill handles dependency ordering (waves),
staggered launch, PID-based monitoring, stale-branch cleanup, and post-wave verification.

## Requirements

- [Claude Code](https://claude.com/claude-code) v2.1+ (for the plugin system).
- The `hermes` CLI installed and on your `PATH`, configured with at least one model.
- A git repository to work in.

## Install

```
/plugin marketplace add CipherraAI/hermes-team-skill
/plugin install hermes-team@hermes-skills
```

Replace `CipherraAI/hermes-team-skill` with wherever you host this repo (`owner/repo` on
GitHub). To try it locally without pushing anywhere:

```
/plugin marketplace add /path/to/hermes-team-skill
/plugin install hermes-team@hermes-skills
```

## Usage

Invoke the skill and point it at a task file (or describe tasks inline):

```
/hermes-team:spawn-hermes-team TASKS.md
```

See [`plugins/hermes-team/examples/TASKS.md`](plugins/hermes-team/examples/TASKS.md) for the
expected task-file shape. The skill:

1. Parses tasks and groups them into dependency **waves**.
2. Sets up a bare clone as `origin` so hermes can track "unpushed" branches.
3. Writes a self-contained prompt file per task.
4. Launches each wave's agents in parallel (2s stagger), monitors PIDs to completion.
5. Runs your project's build/type/test commands and reports per-task commit status.
6. Retries failed tasks, then advances to the next wave.

## How it works

The full orchestration procedure lives in
[`plugins/hermes-team/skills/spawn-hermes-team/SKILL.md`](plugins/hermes-team/skills/spawn-hermes-team/SKILL.md).
It is written to be run by Claude Code, but reads as a plain runbook if you want to drive it
by hand.

## Notes

- The skill is language-agnostic. Each task supplies its own `Verify:` commands, and Step 7
  auto-detects the project's build/test commands from marker files (`Cargo.toml`,
  `package.json`, `go.mod`, `pyproject.toml`, `pom.xml`, `build.gradle`, …). The example
  `TASKS.md` happens to be Rust, but nothing in the skill assumes it.
- Model aliases are install-specific — the skill uses `<model>` as a placeholder. Run
  `hermes --help` to see what your build supports.

## License

MIT — see [LICENSE](LICENSE).
