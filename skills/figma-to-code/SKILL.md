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
- **Figma Frames**: [fileKey and nodeId for each frame, e.g. `fileKey: abc123, nodeId: 573:158941`]

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

Use Playwright to screenshot the implemented component at the same viewport size as the Figma design. Save implementation screenshots with the `-impl` suffix — this is mandatory:

```
/tmp/figma-screenshots/<component-or-page-name>/<node-name>-impl.png
```

**`-impl` suffix is required.** A screenshot without it will be mistaken for a Figma reference.

```
/tmp/figma-screenshots/ticket-purchase-flow/events-page-impl.png  ← correct
/tmp/figma-screenshots/ticket-purchase-flow/events-page.png        ← wrong
```

**IMPORTANT: Always prefer the CLI over writing scripts.** The `npx playwright screenshot` CLI is self-contained and avoids module resolution issues. Only write a script if you need complex interactions (clicking, scrolling to dynamic content, etc.).

#### Setup (one-time)

```bash
npx playwright install chromium
```

#### CLI Screenshots (preferred)

```bash
# Basic screenshot
npx playwright screenshot --viewport-size="<width>,<height>" <local-url> /tmp/screenshot.png

# Full scrollable page
npx playwright screenshot --viewport-size="<width>,<height>" --full-page <local-url> /tmp/screenshot.png

# Wait for content to load (use instead of writing a script just for waits)
npx playwright screenshot --viewport-size="1440,900" --wait-for-timeout=2000 <local-url> /tmp/screenshot.png

# Wait for specific element before capturing
npx playwright screenshot --viewport-size="1440,900" --wait-for-selector=".hero-section" <local-url> /tmp/screenshot.png
```

#### Section-level screenshots (recommended for large pages)

When implementing a page with multiple sections, capture each section individually rather than the entire page. This makes comparison more manageable and discrepancies easier to identify.

```bash
# Navigate to section via URL hash
npx playwright screenshot --viewport-size="1440,900" --wait-for-timeout=1000 "<local-url>#section-id" /tmp/section.png
```

#### Scripts (only when CLI is insufficient)

Only write a Playwright script when you need complex interactions like clicking through a flow, scrolling to dynamic content, or capturing multiple states.

**Critical: Run scripts from the project directory**, not `/tmp/`. Node needs access to `node_modules`.

```bash
# WRONG - will fail with "Cannot find module 'playwright'"
node /tmp/my-script.js

# CORRECT - run from project directory
cd /path/to/project && node scripts/capture-screenshot.js
```

If you must write a script:

```javascript
// Save this IN the project directory, e.g., scripts/capture-screenshot.js
const { chromium } = require('playwright');

(async () => {
  const browser = await chromium.launch();
  const page = await browser.newPage({ viewport: { width: 1440, height: 900 } });
  await page.goto('http://localhost:3000', { waitUntil: 'networkidle' });
  
  // Scroll to section
  await page.locator('text=Section Heading').scrollIntoViewIfNeeded();
  await page.waitForTimeout(500);
  await page.screenshot({ path: '/tmp/section-screenshot.png' });
  
  await browser.close();
})();
```

Then run it:

```bash
cd /path/to/project && node scripts/capture-screenshot.js
```

#### Best practices

- **Prefer CLI over scripts** — fewer things can go wrong
- **Use `--wait-for-timeout` or `--wait-for-selector`** instead of writing scripts just to add waits
- **Run scripts from the project directory** where `node_modules` exists
- **Use CommonJS (`require`)** not ESM (`import`) to avoid syntax issues
- Match the viewport to the Figma frame dimensions from the design context
- For pages with many sections, capture and compare each section separately

### 7b: Compare Against Figma Screenshot

Call `get_screenshot(fileKey, nodeId)` using the values recorded in the spec — the Figma reference appears as an embedded image in context. Then use the `Read` tool on the `-impl.png` file — the implementation screenshot appears in the same context. Compare both images visually. Identify discrepancies in:

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
| Visual mismatch after implementation | Spacing, colour, or typography discrepancy | Re-fetch Figma reference via `get_screenshot` and read the `-impl.png` side-by-side; re-check values in design context |
| Assets not loading | `localhost` URLs being modified | Use the MCP-provided `localhost` URLs directly without modification |
| Design tokens differ from Figma | Project tokens have different values | Prefer project tokens for consistency; adjust spacing/sizing minimally to match visuals |
| Code Connect script appears | Component mapping required | Stop and follow the script exactly; do not proceed until resolved or skipped |
| Playwright not installed | Browser executable missing | Run `npx playwright install chromium` before taking screenshots |
| Screenshot viewport doesn't match Figma | Wrong dimensions used | Check Figma frame width/height in design context and pass to `--viewport-size` |
| Can't screenshot component in isolation | No dedicated route or Storybook | Create a temporary test route or use browser devtools to isolate the element |
| "Cannot find module 'playwright'" | Script running from `/tmp/` or outside project | Run scripts from the project directory where `node_modules` exists, or use the CLI instead |
| No Figma reference for comparison | fileKey/nodeId not recorded | Record fileKey and nodeId in the spec (Step 5); call `get_screenshot` again in Step 7b |
| Implementation screenshot missing `-impl` suffix | Naming convention not followed | All Playwright screenshots must end in `-impl.png` — a file without the suffix will be mistaken for a Figma reference |
| ESM/CJS import errors | Mixing `import` and `require` syntax | Use CommonJS (`require`) syntax; avoid ESM (`import`) for Playwright scripts |
| Page not fully loaded in screenshot | No wait for content | Use `--wait-for-timeout=2000` or `--wait-for-selector=".selector"` CLI flags |
