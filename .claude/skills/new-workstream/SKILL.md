---
name: new-workstream
description: Kick off a new stream of work by spawning a fresh cmux workspace, running git new-worktree for setup in tab 2, and launching a Claude instance in tab 1 briefed on the task. Use whenever the user describes a new piece of work to be done in one of the other repos (RZA, Biggie, etc.) — Bosun's job is to orchestrate, not to do that work inline.
---

# Kicking off a new workstream

Bosun orchestrates other Claude instances. When the user hands you a new task targeting a different repo, do **not** do the work here — spawn a worker.

`git new-worktree` only creates the worktree and runs the project's `setup.sh` (pnpm install, doppler setup, etc.). It does **not** launch Claude anymore — that's your job.

Target layout in the new cmux workspace:
- **Tab 1**: the Claude instance (where you brief the worker).
- **Tab 2**: the setup terminal (where `git new-worktree` ran and stays for follow-up shell work).

## Steps

1. **Pick a branch name.** Format: `<project>/<kebab-case-description>`. No semantic prefixes (no `feat/`, `fix/`, etc.). Describe what the change does, not which ticket. Examples:
   - `rza/redact-sms-in-aircall-webhook-error-logs`
   - `ordering/add-bulk-reassign-button`

2. **Spawn the cmux workspace.** This gives you one default surface (tab) which will become the setup tab:
   ```bash
   cmux new-workspace --name "<project>: <short title>" --focus false
   ```
   Capture the returned ref (e.g. `workspace:17`).

3. **Run `git new-worktree` in the default tab** to create the worktree and kick off setup:
   ```bash
   cmux send --workspace workspace:N "git new-worktree <project> <project>/<branch>"
   cmux send-key --workspace workspace:N Enter
   ```
   Setup (pnpm install, doppler, etc.) can take a minute or two — don't block on it.

4. **Wait for the worktree directory to exist**, then read the screen to confirm `git new-worktree` finished creating it. Path convention (verify against the screen output — `git new-worktree` prints `Worktree ready: <path>` near the end):
   ```
   ~/Projects/sibipro/worktrees/<project>/<branch>
   ```
   For monorepos like biggie/rza, the app subdir under that path is usually `apps/<app-name>`.
   ```bash
   sleep 5
   cmux read-screen --workspace workspace:N --lines 30
   ```

4b. **`cd` the setup tab into the new worktree** so it's a useful shell for follow-up commands. Do this even if setup.sh is still running — the `cd` will queue up and apply once the prompt returns. Use the worktree root (not the app subdir) so the tab can poke at workspace-level things:
   ```bash
   cmux send --workspace workspace:N "cd ~/Projects/sibipro/worktrees/<project>/<branch>"
   cmux send-key --workspace workspace:N Enter
   ```

5. **Create a new surface (tab) for Claude** in the same workspace and move it to position 0 so it becomes tab 1:
   ```bash
   cmux new-surface --type terminal --workspace workspace:N --focus false
   # Note the returned surface ref, e.g. surface:52
   cmux reorder-surface --surface surface:M --index 0 --workspace workspace:N
   ```

6. **Launch Claude in the new tab 1.** `cd` to the worktree app dir (the path you found in step 4) and run `claude`:
   ```bash
   cmux send --workspace workspace:N --surface surface:M "cd <worktree-app-dir> && claude"
   cmux send-key --workspace workspace:N --surface surface:M Enter
   sleep 6
   cmux read-screen --workspace workspace:N --surface surface:M --lines 40
   ```
   **Confirm you see the Claude banner *and* the input prompt (`❯`) before continuing.** If you send a brief before Claude's input is ready, the text disappears into the shell or splash screen and the worker silently sits idle. When spawning many workers in parallel, verify each one individually — they boot at different speeds.

7. **Brief the worker.** Send a self-contained task description — the worker has no memory of this conversation. Include:
   - What the problem is and why it matters (security report, bug, ticket context).
   - Constraints from the user's global CLAUDE.md that apply (TDD, commit cadence, PR title format like `[HED-1234]`, no rebase/amend, branch already exists).
   - When to commit and open a draft PR (per global rules: after the first commit).

   ```bash
   cmux send --workspace workspace:N --surface surface:M "<full briefing text>"
   cmux send-key --workspace workspace:N --surface surface:M Enter
   ```

8. **Report back to the user** with the workspace ref and a one-line summary of what was dispatched.

## What not to do

- **Never** touch the workspace named "Bosun" — that's where you (the orchestrator) are running.
- Don't do the actual work in the Bosun repo. If the task is "fix bug in RZA," your job is to dispatch a worker; the worker does the fix.
- Don't wait for `setup.sh` to finish before launching Claude — they can run in parallel. Claude will block on `pnpm` commands itself if it needs them before setup completes.
- Don't skip the briefing. A worker spawned without a clear task will sit idle.

## Checking in on workers

- `cmux list-workspaces` — see all active streams.
- `cmux list-pane-surfaces --workspace workspace:N` — see the tabs within a worker's workspace.
- `cmux read-screen --workspace workspace:N --surface surface:M --lines 60` — peek at what a worker is doing.
- `cmux send --workspace workspace:N --surface surface:M "<followup>"` + `send-key Enter` — answer questions or redirect.
