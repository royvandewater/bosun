---
name: clean-merged
description: Scan all active cmux workstreams, identify any whose PR has been merged, and clean them up. Run this whenever the user wants to sweep away finished work without naming specific streams.
---

# Clean up merged workstreams

Scan every non-Bosun workspace, detect merged PRs, and tear down the ones that are done.

## Phase 1 — Single-pass discovery (inline)

One command gets everything:

```bash
cmux tree --all --json
```

This returns all workspaces and their surfaces in one shot, including a `url` field on browser surfaces. Use it to build a map of `workspace_ref → surfaces[]`.

Skip the workspace named "Bosun" and any the user has explicitly asked to leave alone.

## Phase 2 — Detect merged PRs (inline)

For each non-Bosun workspace, find its PR:

### Method A — Browser surface URL (preferred)

Look for surfaces with `"type": "browser"` and a non-null `url`. GitHub PR URLs look like:

```
https://github.com/<owner>/<repo>/pull/<number>
```

Parse `owner`, `repo`, and `number` directly from the URL, then:

```bash
gh -R <owner>/<repo> pr view <number> --json state,mergedAt,url
```

### Method B — Branch from worktree (always run for terminal-only workspaces)

**Do not skip this step** just because the terminal title doesn't look like a PR — the worker may have finished without opening a browser tab.

Get the workspace's `current_directory` from the tree output, then:

```bash
git -C <current_directory> branch --show-current
```

Infer the repo from the worktree path:
- `~/Projects/sibipro/worktrees/<repo>/...` → `sibipro/<repo>`
- `~/Projects/royvandewater/worktrees/<repo>/...` → `royvandewater/<repo>`

```bash
gh -R <owner>/<repo> pr list --state all --head <branch> --json number,state,mergedAt,url
```

### No PR found

If neither method surfaces a PR, note it and skip — don't clean up without confirmation.

## Phase 3 — Clean up merged ones (background agents)

For each workspace where the PR state is `MERGED`:

**Spawn a background agent** (one per workspace) with the workspace ref, surface refs from the tree output, and these steps. Tell the user which workspaces you're cleaning before dispatching.

Each cleanup agent should:

1. Identify the Claude terminal surface (look for `✳` or terminal type) from the already-known surface list. If unsure, run `cmux list-pane-surfaces --workspace workspace:N`.

2. Halt Claude on the terminal surface (**not** the browser surface):
   ```bash
   cmux send --workspace workspace:N --surface surface:CLAUDE "/exit"
   cmux send-key --workspace workspace:N --surface surface:CLAUDE Enter
   sleep 2
   ```
   If there is no terminal surface (browser-only workspace), skip this step.

3. Delete the worktree. Use `current_directory` from the tree as the worktree path:
   ```bash
   rm -rf <current_directory>
   git -C <parent-repo-path> worktree prune 2>/dev/null || true
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
