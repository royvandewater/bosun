---
name: fix-cve
description: Spin up a workstream to fix a CVE/Dependabot security alert. Fetches alert details, determines the right fix strategy (pnpm update → parent upgrade → overrides as last resort), then dispatches a worker via new-workstream.
---

# Fixing a CVE

This skill is for addressing Dependabot security alerts. The goal is to land a PR that resolves the vulnerability with the **minimal possible diff** — ideally only a lockfile change.

## Steps

### 1. Fetch the alert

Grab the alert details from GitHub:

```bash
gh api repos/sibipro/<repo>/dependabot/alerts/<number>
```

Note the vulnerable package, the affected `manifest_path` (which subdirectory/app), and the first patched version from `security_vulnerability.first_patched_version.identifier`.

### 2. Dispatch a worker via new-workstream

Follow the `new-workstream` skill to spawn a cmux workspace and Claude instance. Branch naming: `<app>/upgrade-<package>-security` or `<app>/fix-cve-<cve-id>`.

### 3. Brief the worker with the CVE fix strategy below

The briefing must include the vulnerability context **and** the ordered fix strategy. Workers should follow this strategy top-down, stopping as soon as one step resolves the vulnerability.

---

## CVE Fix Strategy (include verbatim in worker briefing)

The goal is a lockfile-only diff where possible. Follow these steps in order:

### Step 1 — Try a direct update

```bash
pnpm update <package>
```

Then verify the installed version is no longer vulnerable:

```bash
pnpm why -r <package>
```

Check that every resolved version in the output is >= the first patched version. If so, you're done — commit and open a PR.

### Step 2 — Check if a parent upgrade fixes it

If `pnpm update <package>` alone doesn't reach a patched version (e.g. a parent pins the vulnerable range), look at `pnpm why -r <package>` to identify which parent packages are pulling in the vulnerable version. Try updating those:

```bash
pnpm update <parent-package>
```

Re-run `pnpm why -r <package>` to confirm the vulnerable version is gone. If resolved, commit and open a PR.

Repeat for any additional parents as needed before moving on.

### Step 3 — Check if the vulnerable package can be removed

Sometimes the parent that pulls in the CVE'd package no longer needs it in a newer version. Check the parent's changelog/release notes. If upgrading the parent removes the vulnerable transitive dep entirely, that's a win — verify with `pnpm why -r <package>` returning nothing.

### Step 4 — Use pnpm overrides (last resort)

Only reach for overrides if no combination of direct or parent upgrades resolves the vulnerability:

```json
// package.json
"pnpm": {
  "overrides": {
    "<package>": ">=<first-patched-version>"
  }
}
```

Run `pnpm install` after adding the override, then verify with `pnpm why -r <package>`. Document in the PR description why overrides were necessary (i.e., which parent pins the vulnerable range and why it can't be upgraded).

---

## PR

- Title: `Upgrade <package> to <patched-version> to address <CVE-ID>`
- Ideally the diff is **lockfile only**. Call this out explicitly in the PR description.
- If overrides were used, explain why in the PR body.
- Commit immediately after the fix; open a draft PR after the first commit.

## What not to do

- Don't manually edit the lockfile.
- Don't bump the vulnerable package in `package.json` unless `pnpm update` alone doesn't do it — let pnpm resolve it naturally.
- Don't add overrides before exhausting Steps 1–3.
