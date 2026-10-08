---
name: new-workstream
description: Kick off a new stream of work by spawning a fresh cmux workspace, running git new-worktree for setup, and launching a Claude instance in tab 1 briefed on the task. Use whenever the user describes a new piece of work to be done in one of the other repos (RZA, Biggie, etc.) — Bosun's job is to orchestrate, not to do that work inline.
---

# Kicking off a new workstream

Bosun orchestrates other Claude instances. When the user hands you a new task targeting a different repo, do **not** do the work here — spawn a worker.

`git new-worktree` only creates the worktree and runs the repo's `setup.sh` (pnpm install, doppler setup, etc.). It does **not** launch Claude anymore — that's your job.

Target layout in the new cmux workspace:
- **Tab 1**: the Claude instance (where you brief the worker). This is the only tab.

## Steps

### Phase 1 — Do inline (fast, needs your judgment)

1. **If the task comes from a Linear ticket, call dibs first:**
   ```bash
   linear dibs <number>
   ```

2. **Pick a branch name.**
   - **Monorepo (rza, biggie, etc.):** prefix with the **app** being changed — `<app>/<kebab-case-description>` (e.g. `ordering/add-metadata`).
   - **Single-package repo (trim-the-sails, git-tools, stack-cleanup, etc.):** use a bare `<kebab-case-description>` — **do not** prefix with the repo name. The branch already lives in that repo; `trim-the-sails/run-in-parallel` is redundant, just use `run-in-parallel`.
   - No semantic prefixes (no `feat/`, `fix/`, etc.).

3. **Write the worker briefing.** Compose the full briefing text you'll send to Claude (see "Brief the worker" below). You need this ready before spawning.

### Phase 2 — Run in background

Once you have the branch name and briefing ready, **spawn a background agent** to handle all the cmux work. Use the Agent tool with `run_in_background: true`. **Do not pass `isolation: "worktree"`** — a worktree-isolated agent gets hard-blocked by the sandbox on any `cmux send` command containing `git` text (it can't verify a relayed git command stays scoped to its own worktree), which kills `git new-worktree` and any later `cd`-into-worktree steps. If a background agent reports hitting this block anyway, finish its remaining steps yourself from the main session rather than retrying through it — and if a blocked `cmux send` left garbled leftover text in the target terminal's input line, clear it with `cmux send-key --workspace <ref> ctrl+c` before resending.

Pass the agent a self-contained prompt that includes:

- The repo name, branch name, workspace display name, and worktree path
- The full briefing text to send to the worker Claude
- The complete procedure below

Tell the user: **"Workstream spinning up in the background — I'll notify you when it's ready."** Then stop — don't run any cmux commands yourself.

---

## Procedure for the background agent

The background agent should execute these steps:

1. **Spawn the cmux workspace** (this creates the default setup tab):
   ```bash
   cmux new-workspace --name "<app>: <short title>" --focus false
   ```
   Capture the returned ref (e.g. `workspace:17`).

   **Set the workspace color right after creation** (always — not just CVE fixes):
   ```bash
   cmux workspace-action set-color <Color> --workspace workspace:N
   ```
   - **CVE fix streams:** always `Amber`.
   - **Everything else:** pick a color by vibes from `Red, Crimson, Orange, Olive, Green, Teal, Aqua, Blue, Navy, Indigo, Purple, Magenta, Rose, Brown, Charcoal` (Amber reserved for CVE fixes). Base the pick on the feel of the task — e.g. a bugfix/investigation might feel Red or Crimson, a new feature might feel Green or Blue, docs/cleanup might feel Charcoal or Olive.
   - **Related streams share a color:** if this workstream is part of the same effort as another currently-active one (same ticket broken into parts, same repo's related follow-up work, etc.), reuse that other stream's color instead of picking a new one — check `cmux workspace list` for existing workspace names/colors first when in doubt.

   **CVE fix streams also join a shared workspace group** so they collapse together in the sidebar:
   ```bash
   cmux workspace-group list --json
   ```
   - If a group named "CVE fixes" already exists, add the new workspace to it:
     ```bash
     cmux workspace-group add --group <group-ref> --workspace workspace:N
     ```
   - If no such group exists yet, create one seeded with just this workspace:
     ```bash
     cmux workspace-group create --name "CVE fixes" --from workspace:N
     ```

2. **Run `git new-worktree`** in the default tab:
   ```bash
   cmux send --workspace workspace:N "git new-worktree <repo> <branch>"
   cmux send-key --workspace workspace:N Enter
   ```
   `git new-worktree <project> <branch>` auto-detects whether `<branch>` already exists (locally, on `origin`, or neither) and does the right thing — checks out the existing branch (e.g. to resume/fix up an already-open PR) or creates a fresh one off the remote default branch. You never need to pass extra flags to target an existing PR's branch — just pass its branch name.

3. **CD the setup tab** into the app directory if the task targets a specific app, otherwise the worktree root for repo-wide tasks:
   ```bash
   cmux send --workspace workspace:N "cd <app-dir-or-worktree-root>"
   cmux send-key --workspace workspace:N Enter
   ```
   For monorepos like rza/biggie the app dir is `~/Projects/sibipro/worktrees/<repo>/<branch>/apps/<app>`.

4. **Wait for setup to finish.** Poll with `cmux read-screen` until `Worktree ready:` appears:
   ```bash
   sleep 5
   cmux read-screen --workspace workspace:N --lines 30
   ```
   Repeat if still installing.

5. **Create the Claude surface** and move it to tab 1:
   ```bash
   cmux new-surface --type terminal --workspace workspace:N --focus false
   cmux reorder-surface --surface surface:M --index 0 --workspace workspace:N
   ```

6. **Launch Claude** in tab 1, `cd`-ing into the app directory when the task targets a specific app (same rule as step 3 — use the worktree root only for repo-wide tasks):
   ```bash
   cmux send --workspace workspace:N --surface surface:M "cd <app-dir-or-worktree-root> && claude"
   cmux send-key --workspace workspace:N --surface surface:M Enter
   sleep 6
   cmux read-screen --workspace workspace:N --surface surface:M --lines 40
   ```
   **CVE fix streams:** launch with Opus 5.5 instead — use `claude --model claude-opus-5-5` in place of `claude` in the command above. All other streams use the default model.

   **Confirm you see the Claude banner AND the `❯` prompt before continuing.** If you send the briefing before Claude's input is ready, the text is swallowed silently.

   **If Claude prompts to enable/connect any MCP servers on startup, always decline** — `cmux send-key --workspace workspace:N --surface surface:M escape`. Never approve or enable an offered MCP server for a worker instance.

   **If Claude shows a "trust this folder" prompt, select "Yes, I trust this folder"** (it's the second option — send `down` then `enter`).

6a. **Close the setup surface** (the original tab from step 1 — now that Claude is running, it's no longer needed):
   ```bash
   cmux close-surface --surface <original-surface-ref> --workspace workspace:N
   ```

7. **Send the briefing:**
   ```bash
   cmux send --workspace workspace:N --surface surface:M "<full briefing text>"
   cmux send-key --workspace workspace:N --surface surface:M Enter
   ```

8. **Report back** with the workspace ref and a one-line summary of what was dispatched.

---

## Brief the worker

The briefing must be self-contained — the worker has no memory of this conversation. Include:
- What the problem is and why it matters.
- The branch already exists (don't create it).

Do **not** repeat constraints that are already in the user's global CLAUDE.md (TDD, commit cadence, PR cadence, no rebase/amend, `git switch`/`git restore`, PR title format, etc.) — the worker's Claude instance loads that file automatically and will follow those rules without being told.

## What not to do

- **Never** touch the workspace named "Bosun".
- Don't do the actual work inline. Your job is to dispatch; the worker does the fix.
- Don't run cmux commands yourself — delegate all of Phase 2 to the background agent.
- Don't skip the briefing. A worker spawned without a clear task will sit idle.

## Checking in on workers

- `cmux workspace list` — see all active streams.
- `cmux list-pane-surfaces --workspace workspace:N` — see the tabs within a worker's workspace.
- `cmux read-screen --workspace workspace:N --surface surface:M --lines 60` — peek at what a worker is doing.
- `cmux send --workspace workspace:N --surface surface:M "<followup>"` + `send-key Enter` — answer questions or redirect.
