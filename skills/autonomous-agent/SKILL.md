---
name: autonomous-agent
description: Autonomously plans, specs, implements, and raises a PR for a task — without requiring the human to be present for the planning phase. Use this skill when the user hands off a task and asks Claude to handle it end-to-end, or when phrases like "auto-implement", "just do it", "handle this autonomously", "fire and forget", or "work on this without me" appear. Also use when the user provides a task description and signals they want to review the output rather than participate in the planning. The skill replaces the human in the grill-me session by using a tech lead subagent instead.
---

# Autonomous Agent Skill

This skill runs the full engineering workflow autonomously — from raw task description to merged PR — without requiring the human in the planning loop.

It mirrors my standard human-in-the-loop workflow exactly, except the grill-me session targets a **tech lead subagent** instead of the human.

---

## Workflow Overview

```
Phase 0: Notify Jeffery
    ↓
Phase 1: Self-Grilling (with tech lead subagent)
    ↓
Phase 2: Create spec
    ↓
Phase 3: Implement spec
    ↓
Phase 4: PR with audit trail + testing guide
    ↓
Phase 5: Report to Jeffery, then to human
```

---

## Phase 0 — Notify Jeffery

Before doing any work, spin up the `Jeffery - Babysitter` agent and inform it that you are beginning an autonomous session using this skill. Pass it this skill file as its reference so it knows what to audit.

Tell Jeffery you will report back once all phases are complete.

---

## Phase 1 — Self-Grilling with Tech Lead Subagent

### Goal
Reach the same shared understanding you would have after a grill-me session with the human — but autonomously, by discussing with the tech lead subagent.

### Running the session

Follow the `grill-me` skill exactly, but direct every question at the tech lead subagent instead of the human.

### Termination condition

Continue until you have no remaining open questions and feel you have reached a clear, shared understanding with the tech lead subagent. At that point, communicate that you are satisfied and ready to proceed to spec.

If there are unresolved blockers that genuinely require human input, **stop here** and surface them clearly before proceeding.

---

## Phase 2 — Spec Creation

Follow the `create-technical-spec` skill exactly.

The spec should reflect the decisions and understanding reached in Phase 1. Do not re-open questions that were already resolved.

Save to: `specs/<feature-name>.md`

---

## Phase 3 — Implementation

You act as orchestrator with a lean context window.

### Worktree setup
First, follow the `setup-worktree` skill to create an isolated worktree for this task. The branch will be created during this step.

### Implementation
Follow the `implement-spec` skill, **skipping Step 0 (Branch Setup)** — the branch and worktree are already in place from the previous step.

Hand off the actual code writing to an implementation subagent for each phase of `implement-spec`. Your role is to:

- Spawn the implementation subagent with the spec and relevant context for each phase
- Receive the output and decide whether to proceed, retry, or escalate
- Spawn the code review subagent at the appropriate checkpoint
- Iterate until code review comes back clean

The implementation subagent should receive: the spec, the relevant repo(s), and the specific phase it is responsible for. It should not carry context from prior phases unless explicitly passed.

---

## Phase 4 — Pull Request

Only once the build is clean and the code review passes, raise the PR following the `implement-spec` skill's PR creation step (Step 5), with the following body structure:

```
## Summary
<Goals from the spec>

## Changes Made
<Bullet list of files/modules touched>

## Decisions
<Structured audit trail of key decisions made during the tech lead subagent session, with reasoning for each>

## Assumptions
<Only include this section if there were assumptions that could not be verified from the codebase, or questions that were deferred. List each with a brief note on why it could not be confirmed. Omit this section entirely if there are none.>

## Code Review Report
<Output from the code review sub-agent>

## Test Plan
<Step-by-step guide describing the app's expected state after this change, tailored to the nature of the task:>
- Bug fix: the app should behave as if the issue never existed
- Feature: describe the new behaviour end-to-end; a reviewer should be able to exercise the full feature from this guide alone
- Refactor/non-functional: describe what should be observably unchanged, and any new behaviour introduced

Each step: [Action] → [Expected result], covering happy path and relevant edge cases.
```

### Task references
If this task originated from a Jira ticket or ClickUp task, reference it explicitly in the PR body (e.g. `Closes SCOOLER-123` or the ClickUp task URL). If neither was used, omit this.

---

## Phase 5 — Jeffery Report

Before reporting completion to the human, report back to Jeffery with a structured summary of what happened in each phase:

- **Phase 1**: how the grill-me session concluded and what the tech lead session resolved
- **Phase 2**: path to the spec file created
- **Phase 3**: worktree path, branch name, confirmation that implement-spec was followed with branch setup skipped
- **Phase 4**: PR URL and full PR body

Wait for Jeffery's response. If he flags any gaps or missed steps, address them and re-report. Only once Jeffery is satisfied, report completion to the human — including the PR URL and Jeffery's sign-off.

---

## Output Summary

| Artifact | Location |
|---|---|
| Technical spec | `specs/<feature-name>.md` |
| Implementation | Feature branch, worktree |
| Pull request | Raised, with audit trail, assumptions (if any), and testing guide |

---

## Notes

- If at any point during Phase 1 you encounter a genuine blocker that cannot be resolved without human input, **stop and surface it**. Do not guess past a hard blocker.
- The decisions section in the PR is not a formality — it should contain real reasoning, not platitudes.