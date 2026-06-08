---
name: cleanup-workstream
description: Tear down a finished workstream — halt the Claude session, remove the git worktree, and close the cmux workspace. Use when the user asks to clean up, close, tear down, or remove a workstream/workspace. Verifies the PR is merged before destructive cleanup and asks for confirmation otherwise.
---

# Cleaning up a workstream

When a workstream is done, this is how to take it down safely. The destructive parts (deleting the worktree, deleting the branch, closing the workspace) are not reversible, so verify the PR is merged or get explicit confirmation first.

## Identify the workstream

The user will name a workspace by ref (`workspace:17`), by cmux name ("the henchman one"), or by branch. Resolve to:
- The **workspace ref** (e.g. `workspace:17`).
- The **worktree path** (the cwd of the setup tab — grab it with `cmux read-screen` on tab 2, or `cmux list-pane-surfaces --workspace workspace:N` and infer).
- The **branch name** (`git -C <worktree-path> branch --show-current`).

```bash
cmux workspace list                                   # find the workspace
cmux list-pane-surfaces --workspace workspace:N       # see the tabs
git -C <worktree-path> branch --show-current          # confirm branch
```

## Check the PR

The branch name is the lookup key. Use `gh` from inside the worktree (it picks up the repo):

```bash
gh -R <owner>/<repo> pr list --state all --head <branch> --json number,state,url,mergedAt
```

Or, simpler — if there's a PR currently associated:
```bash
cd <worktree-path> && gh pr view --json number,state,url,mergedAt
```

Three outcomes:

### A. PR is MERGED → proceed to cleanup, no confirmation needed
The work shipped; the branch and worktree are dead weight. Tell the user "PR #N is merged, cleaning up" and continue to the cleanup section.

### B. PR exists but is OPEN / DRAFT / CLOSED-unmerged → ask the user
Use AskUserQuestion. Make the PR state clear in the question. Two decisions to surface:
1. Proceed with cleanup anyway?
2. If yes, what to do with the PR — leave it open, or close it?

Options to offer:
- "Leave PR open and skip cleanup" (default safe)
- "Close PR and cleanup workstream" — runs `gh pr close <num>` then proceeds
- "Cleanup workstream but leave PR open" — for cases where the user wants to keep the diff but is done with the worktree

### C. No PR found → ask the user
Cleanup deletes the branch too. If there's no PR, the work either never got pushed or got pushed without a PR. Ask:
- "Cancel cleanup" (default safe)
- "Cleanup anyway (deletes local branch — anything not pushed is gone)"

Before asking, run `git -C <worktree-path> log @{u}..HEAD --oneline 2>/dev/null` to see if there are unpushed commits. Mention them in the question if so.

## Cleanup steps

Once you've decided to proceed:

### Resolve surfaces inline (fast)

Run `cmux list-pane-surfaces --workspace workspace:N` to identify:
- The **Claude surface** (tab 1) — look for the one with a `✳` or whose title matches the task.
- The **setup surface** (tab 2) — the terminal showing the worktree path.

### Then background the destructive work

**Spawn a background agent** (Agent tool, `run_in_background: true`) with the workspace ref, both surface refs, and these steps. Tell the user: **"Cleaning up in the background — I'll notify you when it's done."**

The background agent should:

1. **Halt Claude in tab 1:**
   ```bash
   cmux send --workspace workspace:N --surface surface:CLAUDE "/exit"
   cmux send-key --workspace workspace:N --surface surface:CLAUDE Enter
   sleep 2
   ```

2. **Delete the worktree** from the setup tab (`git delete-worktree` removes the worktree AND the local branch):
   ```bash
   cmux send --workspace workspace:N --surface surface:SETUP "git delete-worktree"
   cmux send-key --workspace workspace:N --surface surface:SETUP Enter
   ```
   Poll `cmux read-screen` until it prints "Worktree removed" and "Deleting branch". RZA is large — allow up to ~60s. If it refused (e.g. uncommitted changes), surface the error; don't retry with `--force`.

3. **Close the workspace:**
   ```bash
   cmux close-workspace --workspace workspace:N
   ```

4. **Report back:** "Cleaned up workspace:N — worktree removed, branch deleted, PR #X merged" (or whatever applied).

## What not to do

- Don't skip the PR check. The user may genuinely have lost track of whether something shipped.
- Don't `git delete-worktree --force` unless the user explicitly approves it.
- Don't `cmux close-workspace` before deleting the worktree — once the workspace is gone you can't run commands in it.
- Never close the workspace named "Bosun".
