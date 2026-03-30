# UI Validation

You will receive a local URL, Figma node IDs, frame dimensions, and a description of what to validate.

> **Tooling:** Prefer computer use if available. Fall back to Playwright MCP otherwise. All steps apply to either.

## Process

### 1. Set Up the Browser

Resize the browser to match the Figma frame dimensions provided.

### 2. Navigate and Orient

Navigate to the provided URL and take an initial screenshot to understand the current state of the UI before validating.

### 3. Wait for Content

If the UI has dynamic content (loading states, animations, async data), wait for it to settle before capturing.

### 4. Capture the Implementation Screenshot

Take a screenshot of the current state. For pages with multiple sections, navigate to each section individually (e.g. via URL hash) and capture separately. This makes comparison more manageable.

### 5. Fetch the Figma Reference

Call `get_screenshot(fileKey, nodeId)` using the values provided — the Figma reference appears as an embedded image in context alongside the implementation screenshot. Compare both visually.

### 6. Validate Against Criteria

Work through each criterion provided. For each one, check:

1. **Layout** — spacing, alignment, sizing, overflow
2. **Typography** — font family, size, weight, line height, color
3. **Colors** — backgrounds, borders, shadows, text colors
4. **Assets** — images, icons, illustrations rendering correctly
5. **Borders & Radii** — corner radius, border width/style/color

### 7. Validate Interactive States

Drive interactions to reach states that need validating — hover, focus, active, disabled, open/closed modals, form errors, etc. Capture a screenshot at each meaningful interaction state.

### 8. Check for Errors

After exercising the UI, review console output and network activity for unexpected failures.

## Output

```
## UI Validation Report

### Criteria Results
- [criterion]: PASS / FAIL — [specific finding, e.g. "gap between heading and subtext is 24px, expected 16px"]

### Console Errors
- [any unexpected errors observed, or "None"]

### Network Failures
- [any failed requests, or "None"]

### Overall Verdict
PASS / FAIL
```

Be specific. "Button label is `#1A1A1A`, Figma shows `#000000`" is better than "color is slightly off". Include screenshots as evidence for each failure.
