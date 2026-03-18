---
name: figma-to-code
description: Converts Figma designs to production-ready code using the Figma MCP server. Saves a structured spec artifact and implements with 1:1 visual fidelity. Use when implementing UI from Figma files, when user mentions "implement design", "generate code", "implement component", "build Figma design", provides Figma URLs, or asks to build components matching Figma specs.
---

# Figma to Code Conversion

This skill guides the process of extracting design context from Figma using the Figma MCP server and converting it into production-ready code, while maintaining a structured spec record of the outputs.

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

## Step 2: Fetch Design Context

Run `get_design_context` with the file key and node ID:

```
get_design_context(fileKey=":fileKey", nodeId="1-2")
```

This returns layout properties, typography, colors, component structure, spacing, and design tokens.

**If the response is truncated or too large:**
1. Run `get_metadata(fileKey=":fileKey", nodeId="1-2")` to get the high-level node map. Pass the same top-level node ID — metadata returns the node tree without full style data, so it won't be truncated.
2. Identify the specific child nodes needed from the metadata.
3. Fetch each child individually: `get_design_context(fileKey=":fileKey", nodeId=":childNodeId")`.

**Code Connect handling:**
- If the tool returns a script asking to map components, **STOP** and follow the script exactly as instructed.
- Do not proceed with implementation until Code Connect mappings are resolved or explicitly skipped by the user.

---

## Step 3: Capture and Store Visual Reference

The Figma screenshot serves as the visual source of truth throughout implementation and validation. Screenshots are stored using this path structure:

```
/tmp/figma-screenshots/<component-or-page-name>/<node-name>.png
```

- `<component-or-page-name>` — kebab-case name of the page or component being worked on (e.g. `landing-page`, `event-details`)
- `<node-name>` — kebab-case name of the specific node or section (e.g. `hero-section`, `card`)

**Example:**
```
/tmp/figma-screenshots/landing-page/hero-section.png
/tmp/figma-screenshots/event-details/card.png
```

**Before fetching, check if the screenshot already exists at the expected path.** If it does, use it directly — do not call the MCP. Only fetch from the MCP if:
- The file does not exist, or
- A more detailed view is needed (e.g. a zoomed-in screenshot of a specific sub-section for closer comparison)

If fetching is needed, run:

```
get_screenshot(fileKey=":fileKey", nodeId="1-2")
```

Immediately save the result to disk — do not rely on it staying in context. Create the directory if it doesn't exist. Record the saved path — this is the reference used in Step 7b.

---

## Step 4: Download Required Assets

For any images, icons, or SVGs returned by the Figma MCP server, download them into `src/assets/` (create the directory only if it doesn't already exist) and reference them by their local path in the code.

**Rules:**
- If the Figma MCP server returns a `localhost` source for an image or SVG, fetch it from that URL and save it locally — do not modify the URL when fetching, and do not embed the localhost URL in the final code.
- **Do NOT** add new icon packages. All assets must come from the Figma payload.
- **Do NOT** use placeholders if a `localhost` source is available.

---

## Step 5: Save Spec File

Before writing any code, save a structured spec to `figma-outputs/` at the root of the workspace.

- **Target directory:** `figma-outputs/` (create it if it doesn't exist)
- **Filename format:** `<component-or-page-name>-<node-id>.md` (e.g., `figma-outputs/event-details-205-57932.md`)

### Spec Template

```markdown
# Figma Design Spec: [Component/Page Name]

**Source**: [Figma URL — or "Figma Desktop selection" if no URL was provided]
**Node ID**: [Node ID]
**Date**: [YYYY-MM-DD]

---

## Design Summary
[Brief description of the component or page]

## Assets & Resources
- **Images**: [List of images]
- **Icons**: [List of icons]
- **Fonts**: [Font families used]
- **Figma Screenshots**: [List of saved screenshot paths, e.g. `/tmp/figma-screenshots/landing-page/hero-section.png`]

## Code Connect Status
[Did we use Code Connect? Yes / No / Skipped — reason]

## Raw/Processed Context
[Insert pertinent JSON or code snippets from the MCP tool output.
If too large, summarize the structure or key style tokens.]

## Implementation Plan
1. [Step 1]
2. [Step 2]
3. [Step 3]
```

---

## Step 6: Implement the Code

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

## Step 7: Validate and Fix Visual Discrepancies

After implementation, perform automated visual comparison and fix any discrepancies.

### 7a: Capture Live Screenshot

Use Playwright to screenshot the implemented component at the same viewport size as the Figma design. Save implementation screenshots alongside the Figma reference using the `-impl` suffix:

```
/tmp/figma-screenshots/<component-or-page-name>/<node-name>-impl.png
```

```bash
# Full page screenshot
npx playwright screenshot --viewport-size="<width>,<height>" --full-page <local-url> /tmp/figma-screenshots/<component-or-page-name>/<node-name>-impl.png
```

**Section-level screenshots (recommended for large pages):**

When implementing a page with multiple sections, capture each section individually rather than the entire page. This makes comparison more manageable and discrepancies easier to identify.

```bash
# Scroll to section and capture viewport
npx playwright screenshot --viewport-size="1440,900" <local-url>#section-id /tmp/figma-screenshots/<component-or-page-name>/<node-name>-impl.png
```

Or use a Playwright script to scroll and capture specific regions:

```javascript
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage({ viewport: { width: 1440, height: 900 } });
  await page.goto('<local-url>', { waitUntil: 'networkidle' });
  
  // Scroll to section
  await page.locator('text=Section Heading').scrollIntoViewIfNeeded();
  await page.waitForTimeout(500);
  await page.screenshot({ path: '/tmp/figma-screenshots/<component-or-page-name>/<node-name>-impl.png' });
  
  await browser.close();
})();
```

**Best practices:**
- Match the viewport to the Figma frame dimensions from Step 2
- If Playwright is not installed, run `npx playwright install chromium` first
- For component-level screenshots, navigate to a route or Storybook page that isolates the component
- For pages with many sections, capture and compare each section separately — this makes the fix loop faster and more focused

### 7b: Compare Against Figma Screenshot

Load the Figma reference screenshot from its saved path (recorded in the spec in Step 5). Place it side-by-side with the implementation screenshot. Only re-fetch from the MCP if the file is missing or a more detailed/zoomed view of a specific area is needed for closer comparison. Identify discrepancies in:

1. **Layout** — spacing, alignment, sizing, overflow
2. **Typography** — font family, size, weight, line height, color
3. **Colors** — backgrounds, borders, shadows, text colors
4. **Assets** — images, icons, illustrations rendering correctly
5. **Borders & Radii** — corner radius, border width/style/color

### 7c: Fix Discrepancies Iteratively

For each discrepancy found:

1. Identify the specific CSS property or value that differs
2. Cross-reference the Figma design context from Step 2 for the correct value
3. Update the code to match the Figma specification
4. Re-capture the implementation screenshot
5. Re-compare until the implementation matches the design

**Repeat the capture → compare → fix cycle until no visual discrepancies remain.**

### 7d: Final Validation Checklist

Only mark complete when ALL items pass:

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
| Visual mismatch after implementation | Spacing, colour, or typography discrepancy | Compare side-by-side with the Step 3 screenshot; re-check values in design context |
| Assets not loading | `localhost` URLs being modified | Use the MCP-provided `localhost` URLs directly without modification |
| Design tokens differ from Figma | Project tokens have different values | Prefer project tokens for consistency; adjust spacing/sizing minimally to match visuals |
| Code Connect script appears | Component mapping required | Stop and follow the script exactly; do not proceed until resolved or skipped |
| Playwright not installed | Browser executable missing | Run `npx playwright install chromium` before taking screenshots |
| Screenshot viewport doesn't match Figma | Wrong dimensions used | Check Figma frame width/height in design context and pass to `--viewport-size` |
| Can't screenshot component in isolation | No dedicated route or Storybook | Create a temporary test route or use browser devtools to isolate the element |
