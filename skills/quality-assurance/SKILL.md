---
name: quality-assurance
description: QA skill for browser-based UI validation and functional test plan execution. Handles authentication and routes to the appropriate reference based on the task.
---

# Quality Assurance

> All paths are relative to the skill's base directory provided when you load the skill.

## Determine your mode

**UI validation** — you have been given a URL, Figma node IDs, and frame dimensions to validate against. Follow [UI validation guide](references/ui-validation.md).

**Test plan execution** — you have been given a feature description and a test plan with scenarios and expected outcomes. Follow [test plan guide](references/execute-test-plan.md).

---

## Pre-flight check (browser tasks)

Before any browser interaction, snapshot the entry page and confirm:

1. Which user (if any) is signed in.
2. Whether that matches the account named in the task.

If they don't match — or if no user is signed in and the task names a specific account — **stop and ask** rather than improvising a manual login. The connected Chrome profile is the source of truth for session state.

---

## Authentication

### Browser tasks (UI validation and browser-driven test plans)

Playwright connects to the user's real Chrome browser via the Playwright MCP Bridge extension. The correct Chrome profile is already configured per project — no setup needed.

If the task requires an authenticated session, just proceed — the connected profile already has the user's existing sessions. Do not attempt to log in manually.

**Credentials in the prompt are informational, not an instruction to log in.** If the task prompt provides credentials for a browser test, the connected profile may already hold that session. Run the pre-flight check first; if not signed in as that user, ask before logging in manually.

**Seed state in the same channel you'll test in.** If the test runs in the browser, create preconditions through the browser UI. Do not authenticate via curl and expect that session to carry into the connected browser — they are separate sessions, and data created via curl will appear missing from the browser's perspective.

If the specific browser auth flow has not been specified and is needed, **ask before proceeding**:
- Which flow? (Google SSO, Gmail OTP, Maildrop OTP, or other)
- Which email or account to use?

**Google SSO** — click the "Sign in with Google" button and let the connected Chrome handle it using the existing Google session. Do not enter credentials manually.

**Gmail OTP** — trigger the OTP from the app, then use Gmail MCP (`gmail_search_messages`, `gmail_read_message`) to find and read the OTP email. Paste it into the browser and continue.

**Maildrop OTP** — use `<username>@maildrop.cc` as the email in the app, then navigate to `maildrop.cc/<username>` in the browser to read the OTP. Paste and continue.

### API tasks (curl-based scenarios)

For API-only scenarios, authentication is handled via curl before making requests. If the auth method has not been specified, **ask before proceeding**:
- Which method? (Bearer token, cookie-based, or none)
- Which credentials or endpoint to use?

**Bearer token** — obtain a token via the login endpoint, then pass it as a header on subsequent requests:
```bash
TOKEN=$(curl -s -X POST "https://api.example.com/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "password"}' \
  | jq -r '.token')

curl -s -X GET "https://api.example.com/endpoint" \
  -H "Authorization: Bearer $TOKEN"
```

**Cookie-based** — the server sets a session cookie via a `Set-Cookie` response header after login. Use `-c` to capture it and `-b` to send it on subsequent requests:
```bash
# Log in — curl saves Set-Cookie headers automatically
curl -s -X POST "https://api.example.com/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "password"}' \
  -c /tmp/cookies.txt

# Inspect headers if needed
curl -s -X POST "https://api.example.com/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com", "password": "password"}' \
  -c /tmp/cookies.txt -D -

# Use the cookie in subsequent requests
curl -s -X GET "https://api.example.com/endpoint" \
  -b /tmp/cookies.txt
```

---

## Browser Tooling Note

When launching Playwright or claude-in-chrome, if you see a "Playwright Extension started debugging this browser" page showing "unknown" connected and the connection seems stuck, do a single page reload — that completes the connection. **Do NOT** relaunch the browser in a retry loop; just refresh once and proceed.

## Playwright MCP gotchas

- **Snapshot refs are single-use.** Refs like `e18` are valid only for the snapshot they were captured from. After any `click`, `navigate`, `fill`, key press, or wait, take a fresh snapshot before using a ref. Never retry the same ref after a "not found" error.
- **No new tabs in bridge mode.** `browser_tabs new` / `newPage` is not supported when Playwright is bridged to the user's Chrome. Reuse the active tab and `navigate` to change page.
- **Two strikes rule.** If the same call fails twice with the same error, stop and report. Do not iterate on permutations.

## Input Handling

Prefer using the `fill` method over typing character by character unless the input form doesn't propagate the `fill` as expected. Use `type` or `press` only when `fill` fails to trigger the expected behavior. As a fallback, use `run code` to execute JavaScript that sets the value directly on the input element.

Re-snapshot between fallbacks. A previous `fill` or `type` attempt may have shifted focus or changed the DOM, invalidating any refs you were about to reuse.
