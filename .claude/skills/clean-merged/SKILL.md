---
name: clean-merged
description: Scan all active cmux workstreams, identify any whose PR has been merged, and clean them up. Run this whenever the user wants to sweep away finished work without naming specific streams.
---

# Clean up merged workstreams

Scan every non-Bosun workspace, detect merged PRs, and tear down the ones that are done.

## Phase 1 — Discover workspaces (inline)

```bash
cmux workspace list
```

Note every workspace ref and name, skipping "Bosun" and any workspace the user has explicitly asked you to leave alone (e.g. an open feature stream the user is monitoring).

## Phase 2 — Detect merged PRs (inline, parallel where possible)

For each workspace, find its PR using **one of two methods**:

### Method A — Browser surface (preferred)

```bash
cmux list-pane-surfaces --workspace workspace:N --json
```

Look for a surface with `"type": "browser"` whose `title` matches the pattern:

```
... Pull Request #<number> · <owner>/<repo>
```

Parse `owner`, `repo`, and `number` from the title, then:

```bash
gh -R <owner>/<repo> pr view <number> --json state,mergedAt,url
```

### Method B — Branch from worktree (fallback)

**Always run this for terminal-only workspaces** — do not skip it just because the terminal title doesn't look like a PR. The worker may have finished and pushed without opening a browser tab.

If no browser surface exists, get the workspace's `current_directory` from `cmux workspace list --json`, then:

```bash
git -C <current_directory> branch --show-current
```

Then look up the PR by head branch. You'll need the repo — infer it from the worktree path (e.g. `~/Projects/sibipro/worktrees/<repo>/...` → `sibipro/<repo>`, or `~/Projects/royvandewater/worktrees/<repo>/...` → `royvandewater/<repo>`):

```bash
gh -R <owner>/<repo> pr list --state all --head <branch> --json number,state,mergedAt,url
```

### No PR found

If neither method surfaces a PR, note it and skip — don't clean up without confirmation.

## Phase 3 — Clean up merged ones (background agents)

For each workspace where the PR state is `MERGED`:

**Spawn a background agent** (one per workspace) with the workspace ref, both surface refs, and these steps. Tell the user which workspaces you're cleaning before dispatching.

Each cleanup agent should:

1. List surfaces to identify Claude surface and setup surface:
   ```bash
   cmux list-pane-surfaces --workspace workspace:N
   ```

2. Halt Claude on the terminal surface (look for `✳` or a terminal type — **not** the browser surface):
   ```bash
   cmux send --workspace workspace:N --surface surface:CLAUDE "/exit"
   cmux send-key --workspace workspace:N --surface surface:CLAUDE Enter
   sleep 2
   ```

3. Delete the worktree. If there's a setup terminal surface, use it:
   ```bash
   cmux send --workspace workspace:N --surface surface:SETUP "git delete-worktree"
   cmux send-key --workspace workspace:N --surface surface:SETUP Enter
   ```
   Poll `cmux read-screen` until "Worktree removed" appears (up to 60s). If no setup surface exists, fall back to removing the directory directly:
   ```bash
   rm -rf <worktree_path>
   ```

4. Close the workspace:
   ```bash
   cmux close-workspace --workspace workspace:N
   ```

5. Report: "Cleaned up workspace:N — PR #X merged, worktree removed."

## Phase 4 — Report

After all agents finish, give the user a summary:
- Which workspaces were cleaned (PR number, repo)
- Which were skipped and why (no PR found, PR open/draft, etc.)

## What not to do

- Never touch the workspace named "Bosun".
- Never clean up a workspace whose PR is open, draft, or closed-unmerged without asking first.
- Don't send `/exit` to a browser surface — only to terminal surfaces.
- Don't `close-workspace` before the worktree is deleted.
