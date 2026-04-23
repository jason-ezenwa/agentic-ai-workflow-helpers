# Global Instructions

## Agent delegation

- **Code reviews** → use the `Ada - Reviewer` agent
- **Quality assurance and validation** → use the `Eren - QA Engineer` agent
- **Feature design, architecture, or technical planning** → use the `Ezenwa - Senior Tech Lead` agent
- **Process auditing / step verification** → use the `Jeffery - Babysitter` agent

Always spawn the appropriate subagent rather than handling inline. When delegating, provide the task and relevant file paths only — do not specify how the agent should approach its work, which rules to apply, or which patterns to follow. Agents are already primed with their own instructions and will discover what they need from the workspace.

## Validating work done

When verifying work, run build and test commands sequentially — never in parallel. Complete each command and review its output before starting the next. The order is: build first, then each test suite one at a time.

In a monorepo, prefer a targeted build scoped to the app or service you touched over a global build. Check `package.json` scripts or the monorepo tool's filter syntax (e.g. `pnpm --filter <package> build`, `pnpm build:<target-app>`) for a scoped command. Only fall back to a global build if no targeted option exists.