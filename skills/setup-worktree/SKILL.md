---
name: setup-worktree
description: Sets up a Git worktree for a task or feature. Derives a branch name from the task description, creates an isolated worktree directory, copies env files, and installs dependencies. Use when starting work on a new feature or task in isolation.
---

# Setup Worktree

Creates an isolated Git worktree for a task or feature so work happens independently of the main working tree.

**Prerequisite**: You must be inside the repository directory (or have navigated to it) before starting. The worktrees folder will be created as a sibling to that repository.

**Follow these steps in order. Do not skip steps.**

---

## Step 1: Derive Branch Name

Slugify the task or feature description into kebab-case. Choose a prefix that reflects the type of work:

- New feature → `feat/`
- Bug fix → `fix/`
- Refactor / maintenance → `chore/`
- Hotfix → `hotfix/`
- Fallback when type is unclear → `feat/`

Example: `feat/order-tracking`, `fix/invoice-rounding-logic`

---

## Step 2: Detect Base Branch

Detect the base branch in priority order (`develop` > `development` > `main`):

```bash
for branch in develop development main; do
  if git show-ref --verify --quiet refs/remotes/origin/$branch; then
    echo $branch; break
  fi
done
```

---

## Step 3: Determine Worktree Path

Worktrees are organized in a dedicated folder alongside the main repository. Get the repo name from the current directory:

```bash
REPO_NAME=$(basename $(git rev-parse --show-toplevel))
```

Flatten the branch name for the directory by replacing `/` with `-`:
`feat/order-tracking` → `feat-order-tracking`

Construct the worktree path inside a dedicated worktrees folder:

```
../<repo-name>-worktrees/<branch-slug>
```

Example: repo `api-service`, branch `feat/order-tracking` → `../api-service-worktrees/feat-order-tracking`

### Expected Directory Structure

After setup, the layout looks like:

```
.
├── api-service/                    # main repo
└── api-service-worktrees/          # dedicated worktrees folder (sibling of main repo)
    ├── feat-order-tracking/        # worktree for feat/order-tracking
    ├── fix-invoice-rounding/       # worktree for fix/invoice-rounding
    └── chore-update-deps/          # worktree for chore/update-deps
```

All worktrees share the same `.git` repository metadata from the main repo. Running `git worktree list` from `api-service/` will show all of them.

---

## Step 4: Create the Worktree

**Check if the branch already exists** (locally or remotely):

```bash
git show-ref --verify refs/heads/<branch-name>          # local
git show-ref --verify refs/remotes/origin/<branch-name> # remote
```

### Branch does not exist — create it

Pull the latest base branch and create a new branch in the worktree:

```bash
git fetch origin <base-branch>
git worktree add ../<repo>-worktrees/<branch-slug> -b <branch-name> origin/<base-branch>
```

### Branch already exists — resume it

First check whether the branch is currently checked out in the main working tree:

```bash
git branch --show-current
```

**If the current branch matches** — the branch is checked out here and cannot be simultaneously checked out in a worktree. Stop and warn the user:

> "Branch `<branch-name>` is currently checked out in this working tree. Switch to `<base-branch>` first, then re-run:
> ```bash
> git checkout <base-branch>
> ```"

**If the current branch does not match** — the branch is free to be resumed in a worktree:

```bash
git worktree add ../<repo>-worktrees/<branch-slug> <branch-name>
```

---

## Step 5: Copy Env Files

Find all `.env*` files up to 3 directories deep in the current repository:

```bash
find . -maxdepth 3 -name '.env*' -type f
```

Copy each one to the corresponding relative path inside the worktree, preserving directory structure:

```bash
cp --parents <env-file> ../<repo>-worktrees/<branch-slug>/
```

If no `.env*` files are found, note this in the report — the user may need to add them manually.

---

## Step 6: Install Dependencies

Detect the package manager by checking for lockfiles in priority order:

| Lockfile | Package manager | Install command |
|----------|----------------|-----------------|
| `bun.lockb` | Bun | `bun install` |
| `pnpm-lock.yaml` | pnpm | `pnpm install` |
| `yarn.lock` | Yarn | `yarn install` |
| `package-lock.json` | npm | `npm install` |

Fallback if no lockfile is found: `npm install`

Run the install command from inside the worktree directory:

```bash
cd ../<repo>-worktrees/<branch-slug> && <install-command>
```

If the install fails, warn the user — common causes are missing `.npmrc`, private registry tokens, or unavailable network. Note the failure in the report and move on.

---

## Step 7: Report

Confirm what was set up:

- **Worktree path** — full path to the new worktree
- **Branch** — name of the branch checked out
- **Base branch** — what it branched from (or "existing branch resumed" if applicable)
- **Env files copied** — list of files copied, or a note if none were found
- **Dependencies** — package manager used and whether install succeeded, or failure reason if it failed
- **Next step** — the `cd` command to enter the worktree:

```bash
cd ../<repo>-worktrees/<branch-slug>
```
