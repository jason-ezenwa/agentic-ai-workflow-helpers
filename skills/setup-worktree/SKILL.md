---
name: setup-worktree
description: Sets up a Git worktree for a task or feature. Derives a branch name from the task description, creates an isolated worktree directory, copies env files, and installs dependencies. Use when starting work on a new feature or task in isolation.
---

# Setup Worktree

Creates an isolated Git worktree for a task or feature so work happens independently of the main working tree.

## Prerequisites

Run the command while your shell working directory is the target git repository root.

The helper script lives in this skill bundle at:
`scripts/setup-worktree.sh`

The script auto-detects the package manager by checking for lockfiles in priority order:
`bun.lockb` > `pnpm-lock.yaml` > `yarn.lock` > `package-lock.json`

> Script path is relative to the skill's base directory provided when you load the skill.

## Usage

```bash
scripts/setup-worktree.sh <verb> <feature-description>
```

| Argument | Required | Values |
|----------|----------|--------|
| verb | Yes | feat, fix, chore, hotfix |
| feature-description | Yes | kebab-case (e.g., reference-fix, invoice-rounding-logic) |

### Example

```bash
scripts/setup-worktree.sh feat order-tracking
```

## Execution Rules

- Resolve `scripts/setup-worktree.sh` relative to this skill's directory.
- Execute the script with the target repository root as the shell working directory.
- Do not assume the target repository contains its own `scripts/setup-worktree.sh` unless explicitly stated elsewhere.

## What It Does

The script automatically:

1. Detects package manager from lockfiles.
2. Derives branch name from verb + feature description, for example `feat/order-tracking`.
3. Detects base branch in this priority order: `develop` > `development` > `main`.
4. Checks whether the branch exists locally or remotely, and resumes it if it does.
5. Creates a worktree at `../<repo>-worktrees/<branch-slug>`.
6. Copies `.env*` files up to 3 directories deep.
7. Installs dependencies using the detected package manager.
8. Outputs a JSON report with path, branch, and next command.