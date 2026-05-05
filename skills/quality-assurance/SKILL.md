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

## Authentication

### Browser tasks (UI validation and browser-driven test plans)

Playwright connects to the user's real Chrome browser via the Playwright MCP Bridge extension. The correct Chrome profile is already configured per project — no setup needed.

If the task requires an authenticated session, just proceed — the connected profile already has the user's existing sessions. Do not attempt to log in manually.

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
