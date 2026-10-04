---
name: builder-cli
description: Hand implementation tasks to headless builder CLIs (opencode, agy/Antigravity, codex) driven by Claude, one git worktree per task, builder never commits, Claude reviews and the owner approves. Use when the user says "use opencode", "use agy", "use codex", "send it to a builder", or wants to parallelise work across agents.
---

# Builder CLIs

Claude orchestrates and reviews. Builders write code. Never let a builder review itself,
and never let it commit.

## The rule

1. **Headless only, driven by Claude.** Launch the CLI from Bash with `run_in_background`;
   Claude is notified when it exits.
2. **One git worktree per task**, never two builders in one worktree:
   `git worktree add ../<repo>-<task> -b <branch> <base>` (base = the repo's default branch
   unless the user names another). Launch the CLI **from inside that worktree**.
3. **The builder never commits, pushes, merges, rebases or stashes.** Say so in the prompt.
4. **The prompt is self-contained:** files to read in full, the exact task, the only paths it
   may change, the gate commands with real exit codes (no pipes hiding them), and the report
   shape (changed files + gate output tails). Write it to `"$TMPDIR/<task>-brief.txt"`.
5. **Owner approves.** After review Claude stops and reports. No commit, push or merge until
   the task owner says so.

## Launch (from inside the worktree)

| CLI | Command |
|---|---|
| opencode | `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS=900000 opencode run --model <model> "$(cat "$TMPDIR/<task>-brief.txt")"` |
| agy | `agy -p "$(cat "$TMPDIR/<task>-brief.txt")" --dangerously-skip-permissions` |
| codex | `codex exec -s workspace-write "$(cat "$TMPDIR/<task>-brief.txt")"` |

### opencode
- **Check the model first:** `opencode models`, `opencode auth list`, then
  `opencode run --model <model> "Reply READY"`. If it can't answer that, it can't build.
- `No payment method` = the provider has no billing; `User not found` = dead key. Not a code
  problem: stop and tell the user.
- **Sandboxed to its working directory.** Reads outside it are auto-rejected in headless mode.
  Every file the brief points at must be inside the worktree (copy specs in if needed).
- **Bash tool dies at 120s** unless `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS` is set,
  as in the command above.

### agy
- `-p` runs one turn and exits; `--print-timeout 0` (default) waits for it to finish.
- It may only write inside the worktree. Add `--add-dir <path>` only for extra read access
  the task really needs.

### codex
- `workspace-write` keeps writes inside the worktree. Use
  `--dangerously-bypass-approvals-and-sandbox` only when the gate needs Docker/network and the
  user has OK'd it.

## Prepare the worktree first

A fresh worktree has no installed dependencies. Install or copy them (`node_modules/`,
`vendor/`, build caches) before launch, so the builder spends its budget on the task.

## Review gate (Claude)

Read the full diff, re-run every gate yourself, confirm nothing changed outside the worktree,
then stop and report: changed files, gate results, anything the builder got wrong.
