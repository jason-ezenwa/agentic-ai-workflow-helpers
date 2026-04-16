---
name: code-review
description: Reviews code changes for correctness, quality, security, and rule compliance. Works on local files or a GitHub PR. Produces a structured report with severity levels and a pass/fail verdict.
---

# Code Review Skill

Reviews code for correctness, quality, and compliance with project rules. Works in two modes depending on what is provided.

## Review Mode

### Files mode (default)
Review the specific files or directories mentioned by the user, or the files contextually relevant to the current task (e.g. files just implemented or modified). Read the files directly. No git branch comparison is needed — you may be on any branch or none.

### PR mode
If the user explicitly provides a PR number or URL, fetch the diff first:
```bash
gh pr diff <number-or-url>
```
Then review the diff output instead of reading files directly.

---

## Before Reviewing: Load Rules

You have two sources of rules — apply both, with project rules taking priority where they overlap.

**1. Global rules** — already available to you. Use them as the baseline.

**2. Project rules** — look for a rules directory in the project. Common locations:
- `.agent/rules/`
- `.claude/rules/`
- `.cursor/rules/`

If a project rules directory exists, **list the filenames only** first. Based on the filenames and the files being reviewed, determine which rules are likely relevant — do not read any rule file yet. Then read only the rule files whose names suggest they apply to the current context. For example, a rule named `database-index-creation.md` is irrelevant for a purely frontend change. Use judgment; never load all rules up front.

If no project rules directory exists, proceed with global rules only.

---

## Review Checklist

For each file or diff being reviewed, check:

1. **Correctness**: Does the code do what it's supposed to?
2. **Edge cases**: Are error conditions and boundary inputs handled?
3. **Security**: Are there hardcoded secrets? Are unsanitised inputs passed to queries or shell commands?
4. **Type safety**: Are `any` types introduced without justification? Are type assertions masking real errors?
5. **Performance**: Are there obvious inefficiencies?
6. **Rules compliance**: Does the code comply with the applicable global and project rules identified above?
7. **Style**: Does the code follow the structural and organisational conventions of the codebase — module arrangement, file structure, naming patterns? For UI changes, does the new UI match the visual language of the surrounding product — spacing, typography, component usage, design system tokens, and page layout patterns consistent with the rest of the product?
8. **Dead code**: Are there unused imports, variables, or unreachable branches introduced by this change?
9. **Unnecessary abstraction**: Are there helpers, utilities, or layers introduced for a single use case? Is complexity justified by the actual problem, or speculative?
10. **Test quality**: If tests are present — are they asserting on outcomes (return values, thrown errors) rather than implementation details (which methods were called, in what order)? Do they cover real edge cases rather than just the happy path? Do test names describe observable behaviour? Would these tests catch a regression if the implementation changed?
11. **Code cleanliness**: Are there leftover debug statements, commented-out code, or redundant logic that shouldn't be in the final change? Are variable names descriptive and consistent with codebase conventions? Are functions focused and appropriately sized, or doing too much? Is the module/function hierarchy clear and navigable? Are magic numbers or strings used where named constants should be?

---

## How to Provide Feedback

- Be specific about what needs to change
- Explain why, not just what
- Suggest alternatives when possible
- Tag every finding with a severity:
  - **Blocker** — must be fixed before the review passes (rule violation, bug, broken type safety)
  - **Warning** — should be fixed but won't block
  - **Suggestion** — optional improvement

---

## Report Format

Always produce a report in this exact structure:

```
## Code Review Report

### Blockers
- <file>:<line> — <description>

### Warnings
- <file>:<line> — <description>

### Suggestions
- <file>:<line> — <description>

### Verdict
PASS / FAIL  (FAIL if any Blockers exist)
```

If a section has no findings, write `None` under it. Do not omit sections.
