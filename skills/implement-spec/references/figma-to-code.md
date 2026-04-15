# Figma to Code

**Follow these steps in order. Do not skip steps.**

---

## Step 1: Get Node ID

### Option A: Parse from Figma URL

When the user provides a Figma URL, extract the file key and node ID.

**URL format:** `https://figma.com/design/:fileKey/:fileName?node-id=1-2`

- **File key:** the segment after `/design/`
- **Node ID:** the value of the `node-id` query parameter (e.g., `42-15`)

**Example:**
- URL: `https://figma.com/design/kL9xQn2VwM8pYrTb4ZcHjF/DesignSystem?node-id=42-15`
- File key: `kL9xQn2VwM8pYrTb4ZcHjF`
- Node ID: `42-15`

### Option B: Figma Desktop App Selection (`figma-desktop` MCP only)

When using the `figma-desktop` MCP and the user has NOT provided a URL, tools automatically use the currently selected node from the open Figma file. Only `nodeId` is needed — the server infers the file automatically.

---

## Steps 2 & 3: Fetch Design Context + Preview (Run in Parallel)

Use sub-agents to do this work concurrently — all fetching happens simultaneously before implementation begins.

### For a single frame

Run directly (no sub-agent needed):

```
get_design_context(fileKey=":fileKey", nodeId="1-2")
get_screenshot(fileKey=":fileKey", nodeId="1-2")
```

Alongside this, spawn one **codebase scan sub-agent**: give it a plain-language description of the UI being implemented and ask it to find existing components, patterns, and design tokens that are likely reusable. It returns file paths and a summary.

### For multiple frames

Spawn one **sub-agent per frame** in parallel. Each sub-agent:
1. Calls `get_design_context` for its assigned frame — returns the full context (layout, typography, colors, spacing, tokens)
2. Calls `get_screenshot` for its assigned frame — previews the visual reference
3. Returns the complete design context and any Code Connect findings

Simultaneously, spawn one **codebase scan sub-agent** as above.

Collect all results before proceeding to Step 5.

### Handling large design contexts

**If `get_design_context` is truncated or too large** (single frame or within a sub-agent):
1. Run `get_metadata(fileKey=":fileKey", nodeId="1-2")` to get the high-level node map.
2. Identify the specific child nodes needed.
3. Fetch each child individually: `get_design_context(fileKey=":fileKey", nodeId=":childNodeId")`.

### Code Connect handling
- If `get_design_context` returns a script asking to map components, **STOP** and follow the script exactly.
- Do not proceed with implementation until Code Connect mappings are resolved or explicitly skipped by the user.

---

## Step 4: Download Required Assets

For any images, icons, or SVGs returned by the Figma MCP server, download them into `src/assets/` (create the directory only if it doesn't already exist) and reference them by their local path in the code.

**Rules:**
- If the Figma MCP server returns a `localhost` source for an image or SVG, fetch it from that URL and save it locally — do not modify the URL when fetching, and do not embed the localhost URL in the final code.
- **Do NOT** add new icon packages. All assets must come from the Figma payload.
- **Do NOT** use placeholders if a `localhost` source is available.

---

## Step 5: Implement the Code

Translate the Figma output into the project's framework, styles, and conventions.

**Key principles:**
- Treat the Figma MCP output as a design + behaviour reference, not final code style.
- **Always** check for existing components before creating new ones — reuse over recreation.
- Replace Tailwind utility classes with the project's design system tokens and conventions where they differ.
- Use the project's color system, typography scale, and spacing tokens consistently.
- Respect existing routing, state management, and data-fetching patterns.
- When project design tokens differ from Figma values, prefer project tokens but adjust spacing/sizing minimally to maintain visual fidelity.
- Follow WCAG accessibility requirements.
- Add TypeScript types for all component props.
- Add JSDoc comments for exported components.
- Avoid hardcoded values — extract to constants or design tokens.

---

## Step 6: Validate and Fix Visual Discrepancies

Delegate this step to a **QA agent**. Do not perform browser validation yourself.

Provide the QA agent with:
- The **local URL** of the implemented component or page
- The **Figma fileKey and nodeId(s)** from Step 1
- The **Figma frame dimensions** (width × height) from the design context
- A description of what to validate: layout, typography, colors, interactive states, responsive behaviour, and assets

The QA agent will return a structured validation report with a pass/fail verdict per criterion and screenshot evidence.

### 6b: Fix Discrepancies

For each failure in the report:

1. Identify the specific CSS property or value that differs
2. Cross-reference the Figma design context from Step 2 for the correct value
3. Update the code to match

Once fixes are applied, **delegate to the QA agent again** to re-validate. Repeat until the report is a full pass.

### 6c: Final Validation Checklist

Only mark complete when the QA report shows ALL passing:

- [ ] Layout matches (spacing, alignment, sizing)
- [ ] Typography matches (font, size, weight, line height)
- [ ] Colors match exactly
- [ ] Interactive states work as designed (hover, active, disabled)
- [ ] Responsive behaviour follows Figma constraints
- [ ] Assets render correctly
- [ ] Accessibility standards met (WCAG)

---

## Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Design context is truncated | Design is too complex for a single response | Use `get_metadata` to get node structure, then fetch specific child nodes individually |
| Visual mismatch after implementation | Spacing, colour, or typography discrepancy | Fix the flagged CSS properties, then re-delegate to the QA agent for re-validation |
| Assets not loading | `localhost` URLs being modified | Use the MCP-provided `localhost` URLs directly without modification |
| Design tokens differ from Figma | Project tokens have different values | Prefer project tokens for consistency; adjust spacing/sizing minimally to match visuals |
| Code Connect script appears | Component mapping required | Stop and follow the script exactly; do not proceed until resolved or skipped |
| No Figma reference for comparison | fileKey/nodeId not captured | Note the fileKey and nodeId from Step 1 — pass them to the QA agent when delegating Step 6 |
| Can't screenshot component in isolation | No dedicated route | Create a temporary test route to isolate the component, then provide that URL to Eren |
