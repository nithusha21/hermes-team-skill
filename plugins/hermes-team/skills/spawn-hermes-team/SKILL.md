---
name: spawn-hermes-team
description: 'Spawn a parallel team of hermes coding agents in isolated git worktrees. Use when the user wants to implement multiple independent tasks simultaneously using the hermes agent system. Triggers on: "spawn hermes agents", "launch hermes team", "run these tasks in parallel with hermes", "hermes team", "parallel hermes agents". The skill handles git infrastructure setup (bare clone as origin), per-task prompt file generation, staggered agent launch, PID-based monitoring, stale branch cleanup, and wave-based dependency ordering.'
argument-hint: [path/to/TASKS.md or inline task list]
---

# Spawn Hermes Team

This skill orchestrates a parallel hermes agent team across isolated git worktrees.

## Prerequisites

- The `hermes` CLI must be installed and on your `PATH`, configured with at least
  one model. Verify with `hermes --version`.
- You must be inside a git repository (the working tree you want the agents to modify).
- Substitute your own model alias wherever this skill shows `<model>` — the alias
  names below (Step 5) are examples, not guaranteed to exist in your install. Run
  `hermes --help` to see the flags your build supports.

## Overview

Each hermes agent runs `--worktree` which creates its own `hermes/<hex>` branch. They commit independently and their work merges via git. You manage them via PIDs and monitor completion.

## Input Parsing

### If `$ARGUMENTS` points to a file (e.g. `TASKS.md`)
Read it and extract tasks: ID, title, target files, deps, and any implementation notes.

### If tasks are inline in the conversation
Parse them from the user's message. Each task needs at minimum: **ID**, **what to implement**, **files to touch**, **verification commands**, **commit message**.

### Dependency analysis
Group tasks into waves: tasks with no deps are Wave 1; tasks whose deps are all in Wave 1 are Wave 2; etc. Waves execute sequentially, tasks within a wave execute in parallel.

---

## Step 1: Determine directories

```bash
# Repo root (current working directory)
REPO="$PWD"

# Temp dir: use job dir if in a background session, else create one
TMP="${CLAUDE_JOB_DIR:-/tmp}/hermes-team-$(date +%s)"
mkdir -p "$TMP"

# Bare clone path (hermes needs a remote to track "unpushed" branches)
BARE="$TMP/repo.git"
```

---

## Step 2: Set up bare clone as origin

```bash
git clone --bare "$REPO" "$BARE"
git remote set-url origin "$BARE"
```

If `origin` already points somewhere real (GitHub), save it first and restore after the session:
```bash
REAL_ORIGIN=$(git remote get-url origin 2>/dev/null || echo "")
# ... after session: git remote set-url origin "$REAL_ORIGIN"
```

---

## Step 3: Write prompt files

For **each task**, write `$TMP/p_<task-id>.txt`. Use this exact template:

````
You are implementing task <TASK_ID>: <TASK_TITLE>.

You are running in an isolated git worktree. Do NOT run 'git checkout' or create new branches — the branch is already set up for you. Just implement the task, run verification, and 'git add' + 'git commit' in the current directory.

Working directory: <REPO_PATH>

## What to implement

<FULL IMPLEMENTATION DETAILS — be specific: types/signatures, algorithms, edge cases, and any dependencies to add to the project's manifest (e.g. Cargo.toml, package.json, go.mod, pyproject.toml)>

## Key files to read before implementing

<list of files the agent must read to understand context>

## Files to create/modify

<explicit list: CREATE X, MODIFY Y>

## Verification

```
<verification commands that must both pass>
```

## Commit

```
git add <files>
git commit -m "<feat(TASK_ID): description>"
```

Git user: <git config user.name> <<git config user.email>>
````

**Prompt quality rules:**
- Include all type/interface signatures the agent needs — don't make it guess
- List exact files to read first (always include the project's core model/type module and relevant existing modules)
- Specify which deps to add if needed
- Verification must be runnable commands, not descriptions
- The commit message format must match the project convention

---

## Step 4: Clean stale branches

Stale `hermes/<hex>` branches from previously failed agents cause UUID collisions. Always clean before launch:

```bash
git branch | grep 'hermes/' | grep -v '\*' | tr -d ' ' | xargs git branch -D 2>/dev/null || true
```

---

## Step 5: Launch agents with 2-second stagger

For each task in the current wave:

```bash
hermes --worktree -z "$(cat "$TMP/p_<task-id>.txt")" -m <model> &
PIDS+=($!)
echo "Launched <task-id>: PID $!"
sleep 2
```

Record PID → task-id mapping. Show the mapping to the user:
```
Launched Wave N (N tasks):
  PID 12345  →  A4-1: MetricsComputer
  PID 12347  →  A5-1: L1 Preconditions
  PID 12349  →  B3-1: FeatureFlagDetector
```

**Model selection:**
- Pick a fast model as the default for well-specified tasks.
- Pick a stronger/slower model for tasks with complex algorithm design or multiple
  interdependent structs.
- Use whatever aliases your hermes install exposes (`hermes --help` to list them).
- Override: the user can specify `--model <name>` when invoking the skill.

---

## Step 6: Monitor to completion

Use a background task or inline monitor loop. The monitor should:

1. Poll PIDs every 30 seconds: `ps -p <pid1> <pid2> ... > /dev/null 2>&1`
2. Each iteration: clean stale branches again (agents exit dirty after network failures):
   ```bash
   git branch | grep 'hermes/' | grep -v '\*' | tr -d ' ' | xargs git branch -D 2>/dev/null || true
   ```
3. When all PIDs are gone: run completion check (Step 7)

While waiting: show the user active worktrees: `git worktree list`

---

## Step 7: Verify on completion

Verification has two layers; both must pass. **Do not hardcode a language** — derive the
commands from the repo and from each task.

**7a. Detect the project profile.** Inspect the repo root for a marker file and use the
matching build/test commands (the user may override any of these when invoking the skill):

| Marker file | Build / typecheck | Test |
|-------------|-------------------|------|
| `Cargo.toml` | `cargo check --workspace` | `cargo test --workspace` |
| `package.json` | `npm run build` (or `npx tsc --noEmit`) | `npm test` |
| `go.mod` | `go build ./...` | `go test ./...` |
| `pyproject.toml` / `setup.py` | `ruff check .` (or `mypy .`) | `pytest` |
| `pom.xml` | `mvn -q compile` | `mvn -q test` |
| `build.gradle` | `gradle build -x test` | `gradle test` |

If several markers exist (e.g. a Rust + TS repo), run each stack's commands. If none match,
fall back to whatever build/test commands the task file or the user specifies.

**7b. Per-task verification.** Each task already carries its own `Verify:` commands (Step 3
template). Those are the source of truth for that task — run them as written.

```bash
# 1. Check what was committed
git log --oneline <base-sha>..HEAD

# 2. Project-level build/typecheck (from the profile table above)
<detected build command> 2>&1 | tail -20

# 3. Project-level tests (from the profile table above)
<detected test command> 2>&1 | tail -20
```

Report results in a table:

```
Wave N results:
  A4-1  ✓ committed  (sha abc1234)
  A5-1  ✗ not found in git log
  B3-1  ✓ committed  (sha def5678)

Build:  ✓ build passed
Tests:  127 passing, 0 failed
```

---

## Step 8: Handle failures

A task failed if its commit is **not** in `git log <base>..HEAD`.

For each failed task:
1. Check if agent left a worktree: `git worktree list | grep hermes`
2. If worktree exists, read the agent's output there to understand the error
3. Clean stale branches: same command as Step 4
4. Optionally revise the prompt (add missing context or fix type/interface signatures)
5. Relaunch just the failed task (same Step 5, single agent)

If the same task fails 3 times, escalate to the user with the error output.

---

## Step 9: Launch next wave (if any)

After Wave N fully passes verification:
1. Announce: "Wave N complete. Launching Wave N+1: [task list]"
2. Repeat Steps 4–8 for the next wave

---

## Completion

When all waves are done and verification passes:
1. Run final full verification
2. Report total commits, test count, and any soundness gaps noted
3. Restore real origin if it was replaced: `git remote set-url origin "$REAL_ORIGIN"`
4. Optionally clean tmp dir: `rm -rf "$TMP"`

---

## Quick reference: hermes flags

| Flag | Purpose |
|------|---------|
| `--worktree` / `-w` | Isolated git worktree per agent |
| `-z <prompt>` | One-shot: send prompt, print response, exit |
| `-m <model>` | Model override |
| `--accept-hooks` | Auto-approve shell hooks (use with --worktree) |

Confirm the exact flags with `hermes --help`; older/newer builds may differ.

## Common failure modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| Agent exits immediately, no commit | Broken pipe / network | Relaunch; transient |
| `fatal: A branch named 'hermes/abc...' already exists` | Stale branch collision | Clean branches + relaunch |
| Build errors after all agents finish | One agent introduced a compile/type error | Read the erroring file, fix manually or relaunch that task |
| Agent commits but wrong files | Prompt too vague | Revise prompt with exact file paths and type/interface signatures |
