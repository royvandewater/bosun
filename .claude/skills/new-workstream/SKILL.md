---
name: new-workstream
description: Kick off a new stream of work by spawning a fresh cmux workspace, creating a worktree on a feature branch, and launching a Claude instance briefed on the task. Use whenever the user describes a new piece of work to be done in one of the other repos (RZA, Biggie, etc.) — Bosun's job is to orchestrate, not to do that work inline.
---

# Kicking off a new workstream

Bosun orchestrates other Claude instances. When the user hands you a new task targeting a different repo, do **not** do the work here — spawn a worker.

## Steps

1. **Pick a branch name.** Format: `<project>/<kebab-case-description>`. No semantic prefixes (no `feat/`, `fix/`, etc.). The description should reflect what the change does, not which ticket it's tied to. Examples:
   - `rza/redact-sms-in-aircall-webhook-error-logs`
   - `ordering/add-bulk-reassign-button`

2. **Spawn a cmux workspace** with a short human-readable name:
   ```bash
   cmux new-workspace --name "<project>: <short title>" --focus false
   ```
   Returns a ref like `workspace:17`. Keep that ref for the next steps.

3. **Run `git new-worktree`** inside the new workspace. This is a custom command on the user's system (not stock git) — it creates the worktree, `cd`s into the app dir, and launches Claude in one shot:
   ```bash
   cmux send --workspace workspace:N "git new-worktree <project> <project>/<branch>"
   cmux send-key --workspace workspace:N Enter
   ```

4. **Wait for Claude to come up**, then verify:
   ```bash
   sleep 5
   cmux read-screen --workspace workspace:N --lines 40
   ```
   You should see the Claude banner and a prompt.

5. **Brief the worker.** Send a self-contained task description — the worker has no memory of this conversation. Include:
   - What the problem is and why it matters (the security report, the bug, etc.)
   - Any constraints from the user's global CLAUDE.md that apply (TDD, commit cadence, PR title format)
   - When to commit and open a draft PR

   ```bash
   cmux send --workspace workspace:N "<full briefing text>"
   cmux send-key --workspace workspace:N Enter
   ```

6. **Report back to the user** with the workspace ref and a one-line summary of what was dispatched.

## What not to do

- **Never** touch the workspace named "Bosun" — that's where you (the orchestrator) are running.
- Don't do the actual work in the Bosun repo. If the task is "fix bug in RZA," your job is to dispatch a worker; the worker does the fix.
- Don't skip the briefing. A worker spawned without a clear task will sit idle.

## Checking in on workers

- `cmux list-workspaces` — see all active streams.
- `cmux read-screen --workspace workspace:N --lines 60` — peek at what a worker is doing.
- `cmux send --workspace workspace:N "<followup>"` + `send-key Enter` — answer questions or redirect.
