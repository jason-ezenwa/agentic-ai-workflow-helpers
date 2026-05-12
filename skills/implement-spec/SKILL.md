---
name: implement-spec
description: Implements a feature based on a provided technical specification, ensuring build stability and strict adherence to the spec through a sub-agent code review. Creates a feature branch, commits changes, and raises a PR on completion.
---

# Implement Spec Skill

> All paths are relative to the skill's base directory provided when you load the skill.

This skill guides you through implementing a feature defined in a Technical Specification file (usually in `specs/`). It mandates a strict workflow of Worktree Setup -> Analysis & Planning -> Implementation -> Verification -> Code Review -> PR Creation.

## Workflow

### 0. Worktree Setup
Invoke the `/setup-worktree` skill to create an isolated worktree for this task. After the worktree is created, proceed to Analysis & Planning.

### 1. Analysis & Planning
1.  **Read the Spec**: Use the tool available to you for reading files to read the target technical specification artifact (e.g., `specs/feature-name.md`).
2.  **Understand Guardrails**: Pay close attention to the "Goals", "Non-Goals", and "API Design" sections. These are your acceptance criteria.

### 1b. Task List Creation
After reading the spec, create a task list combining skill workflow steps and feature work items. List everything before writing any code. The list must cover both:
1. **Skill workflow items**: Branch Setup, Analysis & Planning, Task List Creation, Implementation (TDD gate, shared deps, parallelisation), Verification, Code Review, PR Creation.
2. **Feature work items**: endpoints, services, models, DTOs, components, pages — drawn from the spec's API Design, Goals, and architecture sections.

### 2. Implementation

#### Parallelisation

Only frontend work is suitable for parallelisation. Backend work (services, repositories, modules) must be implemented sequentially by the main agent following the TDD workflow.

Before writing any code, read the spec and identify independent work streams — frontend components, pages, hooks that don't depend on each other at compile time.

**Shared dependencies first**: any file that other streams will import (DTOs, types, base components, constants) must be implemented in the main session before parallel work begins.

Once shared dependencies exist, delegate frontend streams to subagents — up to a maximum of 2. Do not always spin up 2; use as many as the work genuinely calls for. Parallelise only if there are 2+ independent frontend work streams. If unsure or the spec is small, stick with sequential implementation.

For each subagent, provide:
- The path to the spec file so it can read the full context itself
- The specific files it is responsible for
- A brief note on its role in the overall implementation (e.g. "you are implementing the dashboard page and its components")

**Do not commit during subagent work.** Subagents implement and return results. Review their output, apply any corrections, then commit everything together after verification.

#### Implementation Approach by File Type

**Backend service files and utility functions** — implement them yourself in the main session. Follow the TDD workflow. See [TDD guide](references/tdd.md) for details.

**Frontend with Figma designs** — follow the Figma-to-Code workflow. See [Figma-to-Code guide](references/figma-to-code.md) for details.

**Everything else** (DTOs, models, resolvers, config, pages, components, etc.) — implement directly, file-by-file, following the codebase's existing patterns.

#### Implementation Gate

Before moving to Step 3, confirm:

- [ ] If service files or utility functions were implemented: did I follow the TDD workflow? `references/tdd.md`
- [ ] If Figma designs were provided: did I follow the Figma-to-Code workflow? `references/figma-to-code.md`

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

> **If you find discrepancies:** Fix them now. Do not ask the user for permission to fix bugs you introduced. Once fixes are applied, **send the changes to the same sub-agent for re-review** — not a new instance. If issues persist after this second pass, escalate to the user rather than looping further.

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
