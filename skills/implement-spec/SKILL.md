---
name: implement-spec
description: Implements a feature based on a provided technical specification, ensuring build stability and strict adherence to the spec through a sub-agent code review. Creates a feature branch, commits changes, and raises a PR on completion.
---

# Implement Spec Skill

This skill guides you through implementing a feature defined in a Technical Specification file (usually in `specs/`). It mandates a strict workflow of Branch Setup -> Planning -> Implementation -> Verification -> Code Review -> PR Creation.

## Workflow

### 0. Branch Setup (Before Any Code Changes)
**CRITICAL**: All work must happen on a dedicated branch. Never commit implementation work directly to a base branch.

1. **Check working tree**: Run `git status` first. If there are uncommitted changes:
   - **If creating a new branch** (step 3 will show the branch doesn't exist): require a clean tree before checking out and pulling the base branch. Tell the user what is dirty and ask whether to stash those changes or abort.
   - **If continuing on an existing branch** (step 3 will show the branch exists): inspect whether the dirty files overlap with the task at hand. If they do, ask the user how to proceed. If they are unrelated, continue without touching them. Never silently revert changes you didn't make.

2. **Derive expected branch name**: Slugify the spec filename or task description into kebab-case. Choose a prefix that reflects the type of change:
   - New feature → `feat/`
   - Bug fix → `fix/`
   - Refactor / maintenance → `chore/`
   - Hotfix → `hotfix/`
   - Fallback when type is unclear → `feat/`

   Example: `feat/contractor-onboarding`, `fix/invoice-rounding-logic`

3. **Check if the branch already exists** (locally or remotely):
   ```bash
   git show-ref --verify refs/heads/<branch-name>          # local
   git show-ref --verify refs/remotes/origin/<branch-name> # remote
   ```
   - **If it exists**: check it out and pull any remote changes. This is a continuation of previous work — do not create a new branch.
     ```bash
     git checkout <branch-name>
     git pull origin <branch-name>   # no-op if no remote yet
     ```
   - **If it does not exist**: proceed to steps 4–5 to create it fresh.

4. **[New branch only] Detect base branch** in priority order (`develop` > `development` > `main`):
   ```bash
   for branch in develop development main; do
     if git show-ref --verify --quiet refs/remotes/origin/$branch; then
       echo $branch; break
     fi
   done
   ```

5. **[New branch only] Switch to base branch, pull latest, and create the branch**:
   ```bash
   git checkout <base-branch> && git pull origin <base-branch>
   git checkout -b <branch-name>
   ```

### 1. Analysis & Planning
1.  **Read the Spec**: Use the tool available to you for reading files to read the target technical specification artifact (e.g., `specs/feature-name.md`).
2.  **Understand Guardrails**: Pay close attention to the "Goals", "Non-Goals", and "API Design" sections. These are your acceptance criteria.

### 2. Implementation

#### Parallelisation

Before writing any code, read the spec and identify independent work streams — logical units of work that don't depend on each other at compile time. Both backend streams (service, repository, module) and frontend streams (components, pages, hooks) are valid candidates.

**Shared dependencies first**: any file that other streams will import (DTOs, types, base components, constants) must be implemented in the main session before parallel work begins.

Once shared dependencies exist, delegate independent streams to subagents — up to a maximum of 3. Do not always spin up 3; use as many as the work genuinely calls for. Parallelise only if there are 3+ logically independent work streams that are self-contained enough to implement without constant iteration between streams. If unsure or the spec is small, stick with sequential implementation in the main session.

For each subagent, provide:
- The path to the spec file so it can read the full context itself
- The specific files it is responsible for
- A brief note on its role in the overall implementation (e.g. "you are implementing the backend service layer; the DTOs it depends on are already in place at `src/dto/foo.dto.ts`")

**Do not commit during subagent work.** Subagents implement and return results. Review their output, apply any corrections, then commit everything together after verification.

#### Implementation Approach by File Type

**Backend service files and utility functions** — do not delegate to a subagent. These require close iteration and are not suitable for parallel work. Implement them yourself in the main session. Follow the TDD workflow from `/implement-spec/references/tdd.md` for all service methods and utility functions.

**Frontend with Figma designs** — if the spec includes Figma design references or URLs, follow the Figma-to-Code workflow from `/implement-spec/references/figma-to-code.md` for the implementation of all UI components and pages.

**Everything else** (DTOs, models, resolvers, config, pages, components, etc.) — implement directly, file-by-file, following the codebase's existing patterns. These files can be delegated to subagents if parallelising.

#### Implementation Gate

Before moving to Step 3, confirm:

- [ ] If service files or utility functions were implemented: did I follow the TDD workflow from `/implement-spec/references/tdd.md`?
- [ ] If Figma designs were provided: did I follow the Figma-to-Code workflow from `/implement-spec/references/figma-to-code.md`?

### 3. Verification (The "Build" Check)
**CRITICAL**: You must ensure the application builds successfully after your changes.
1.  **Determine the build command**: Check `package.json` scripts first. Look for `build`, `type-check`, or `typecheck` in that order. Use the first one found. Only fall back to running `tsc` directly if none exist.
2.  **Run it**: Execute the command found above.
3.  **Fix Errors**: If the build fails, you **MUST** fix the errors immediately. Do not proceed until the build is clean.
4.  **Fix Lints**: For lint fixes, fix them manually — do not run `npm lint --fix` or similar automation. This ensures only files in the spec are touched.

### 4. Code Review (The "Guardrails" Check)
Before communicating completion to the user, delegate a code review to a **sub-agent**. Using a sub-agent avoids bias from the implementing agent reviewing its own work.

Pass the sub-agent the list of files changed and the spec content. It returns a report. Using the report, ask yourself:
1.  **Completeness**: Did I implement every endpoint defined in "API Design"?
2.  **Compliance**: Did I meet all "Goals"? Did I avoid all "Non-Goals"?
3.  **Quality**: Are there any linting errors or obvious bugs?
4.  **Consistency**: Does the code match the "Proposed Architecture"?

> **If you find discrepancies:** Fix them now. Do not ask the user for permission to fix bugs you introduced. Once fixes are applied, **spawn the sub-agent one more time** to confirm the issues are resolved. If issues persist after this second pass, escalate to the user rather than looping further.

### 5. PR Creation (After Clean Code Review)
Only once the build is clean and the code review passes:

1. **Commit** following Conventional Commits format. The type must match the branch prefix:
   ```bash
   git add -A
   git commit -m "<type>: <short imperative description>"
   ```
   Examples: `feat: add contractor onboarding flow`, `fix: correct invoice rounding logic`

2. **Push branch**:
   ```bash
   git push -u origin <branch-name>
   ```

3. **Check if a PR already exists** for this branch:
   ```bash
   gh pr view <branch-name> --json url -q .url
   ```
   - **If a PR exists**: skip creation. The new commit is already on the branch and the existing PR is updated. Capture the existing PR URL.
   - **If no PR exists**: create one using the GitHub CLI. The body must include the spec summary, a list of files changed, the code review report, and the test plan (if present in the spec):
   ```bash
   gh pr create \
     --base <base-branch> \
     --title "<type>(<optional scope>): <short description>" \
     --body "$(cat <<'EOF'
   ## Summary
   <Goals from the spec>

   ## Changes Made
   <Bullet list of files/modules touched>

   ## Code Review Report
   <Output from the code review sub-agent>

   ## Test Plan
   <From the spec's verification section, if present>
   EOF
   )"
   ```

   Examples of PR titles: `feat(onboarding): add contractor onboarding flow`, `fix(invoice): correct invoice rounding logic`, `docs: update API usage guide`

4. **Capture the PR URL** from the output.

5. **Fallback**: If `gh` CLI is not available, warn the user, skip PR creation, and provide the branch name so they can open a PR manually.

### 6. Completion
Only when you are **confident** that:
1.  The code is implemented.
2.  The app builds without errors.
3.  The implementation matches the Spec.
4.  A sub-agent has conducted a code review.
5.  A PR has been raised (or the user has been notified of the branch if `gh` is unavailable).

Then notify the user with:
- The **code review report** (findings + what was fixed).
- The **PR URL** for them to review.
- Which **base branch** the PR targets.
