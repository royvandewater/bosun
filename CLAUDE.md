# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Purpose

Bosun is a meta-project: it holds CLAUDE.md files and skills used to manage *other* Claude Code instances running in cmux workspaces. The code/configuration here orchestrates fleets of Claude workers — it is not itself an application.

## Workspace rules

- This repo is checked out inside a cmux workspace named **Bosun**. Never close, rename, or send input to that workspace — it's the one you're running in.
- For each distinct stream of work, spawn a *new* cmux workspace. See the `new-workstream` skill (`.claude/skills/new-workstream/`) for the full procedure.
- Use `cmux list-workspaces` to see existing streams before spawning duplicates. Drive spawned workspaces with `cmux send`, `cmux send-key`, `cmux read-screen`, etc.

## Git workflow (overrides global rules)

- Commit and push after **every** change to files in this repo. No batching.
- Commit directly to `main`. Do not create feature branches in this repo.
- Push immediately after committing so other Claude instances can pull the latest skills/configs.

## Conventions

- Skills go in `.claude/skills/<skill-name>/` with a `SKILL.md` frontmatter block describing when to trigger. (The repo's `.gitignore` un-ignores `.claude/` — the user's global gitignore excludes it by default.)
- CLAUDE.md files at the repo root apply to this project; nested CLAUDE.md files apply to their subtree.
