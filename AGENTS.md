# Global Instructions

## Agent delegation

- **Code reviews** → use the `Ada - Reviewer` agent
- **UI/browser/API validation** → use the `Eren - QA Engineer` agent; provide the URL, the Figma node IDs to validate against, and a description of what to validate
- **Feature design, architecture, or technical planning** → use the `Ezenwa - Senior Tech Lead` agent

Always spawn the appropriate subagent rather than handling inline. When delegating, provide the task and relevant file paths only — do not specify how the agent should approach its work, which rules to apply, or which patterns to follow. Agents are already primed with their own instructions and will discover what they need from the workspace.