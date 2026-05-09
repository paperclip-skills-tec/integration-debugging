---
name: integration-debugging
description: Structured playbook for diagnosing and fixing third-party API integration failures — auth errors (401/403), invalid tokens, credential health check failures, connectivity issues, scope/permission problems, SSE/WebSocket streaming contract mismatches, and API contract drift between producers and consumers. Use this skill whenever you encounter an integration that is failing, a credential that is rejected, an "invalid token" or "unauthorized" error, a health check that returns red, any real-time streaming event that is missing/malformed, a consumer receiving unexpected field shapes from a producer, or any task asking you to investigate, fix, or diagnose an external API connection or event stream. Even if the problem seems obvious, invoke this skill first — the checklist catches systemic issues (token rotation, API version drift, scope changes, schema drift, stale consumer assertions) that a quick fix will miss.
---

# Integration Debugging Playbook

When an external integration fails, improvising the diagnosis risks missing root causes. This playbook walks through a consistent 8-step sequence that covers every failure mode seen across JIRA, GitHub, Azure DevOps, and similar integrations — from a wrong credential format through to full end-to-end request tracing.

Work through the steps in order. Each step has a clear stop condition: if you find the root cause, fix it there and then run Step 8 to verify. If not, continue to the next step.

---

## Step 1 — Reproduce and Capture the Error

Before touching any configuration, get a clean error snapshot.

- Trigger the failing operation and capture the full HTTP response: status code, response body, and any relevant response headers (`WWW-Authenticate`, `X-Error`, `x-ms-diagnostics`).
- Note the exact endpoint URL that failed, the HTTP method, and the auth header format (`Bearer`, `Basic`, `token`, PAT).
- If the error is from a health check page or integration settings UI, read the raw error message (not just the status icon).
- Record these facts before changing anything — you'll compare against them at Step 8.

**Stop condition:** you have a precise error (e.g., `401 Unauthorized: token_expired`, `403 Forbidden: insufficient_scope`, `404 Not Found: project_not_found`). If the error is vague (e.g., just "red"), trigger the operation again with verbose logging or check server logs before proceeding.

---

## Step 2 — Identify the Integration Type and Auth Mechanism

Different providers use different token formats and auth flows. Misidentifying the auth type causes every subsequent step to diagnose the wrong thing.

- Confirm the provider: JIRA Cloud, JIRA Data Center, GitHub.com, GitHub Enterprise, Azure DevOps (ADO), GitLab, Bitbucket, etc.
- Identify the auth mechanism in use:
  - **API Key / PAT** — single static token, often base64-encoded with a username prefix for Basic auth
  - **OAuth 2.0** — access token + refresh token pair, provider-issued, time-limited
  - **Service account credentials** — username + password or API token tied to a machine account
  - **App password / App token** — Atlassian-specific, scoped separately from the account password
- Check the integration settings or code to confirm what is actually being sent — what you expect to be there and what is there can differ after a rotation.

**Stop condition:** you know exactly which provider, which auth scheme, and where the credential is stored (env var, secrets manager, database, settings page).

---

## Step 3 — Validate Credential Storage and Format

The credential may be present but malformed or incorrectly encoded.

- Retrieve the credential from its storage location (env var, DB field, secrets store, integration settings).
- Check format:
  - PAT/API key: should be a raw token string — no extra whitespace, newlines, or surrounding quotes.
  - Basic auth: should be `base64(username:token)` — the colon separator is required; email addresses often contain characters that need no escaping here, but double-check.
  - Bearer token: should be the raw JWT or opaque token — no `Bearer ` prefix in the stored value if the header prepends it, or vice versa.
  - OAuth tokens: access token and refresh token stored separately; neither should be the authorization code (one-time use, already consumed).
- Compare the credential length and prefix against a freshly generated one from the provider's UI to spot truncation or corruption.
- Check that the credential is for the correct environment (production vs. staging/cloud vs. on-prem).

**Stop condition:** credential is present, correctly formatted, and matches the expected environment.

---

## Step 4 — Test the Credential Directly Against the Provider API

Isolate whether the credential itself is valid by bypassing the application layer.

```bash
# Generic Bearer test
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: Bearer <token>" \
  https://<provider>/api/v1/me

# Basic auth test (JIRA pattern)
curl -s -o /dev/null -w "%{http_code}" \
  -u "<email>:<api-token>" \
  https://<instance>.atlassian.net/rest/api/3/myself

# GitHub PAT test
curl -s -o /dev/null -w "%{http_code}" \
  -H "Authorization: token <pat>" \
  https://api.github.com/user

# ADO PAT test (Basic auth with empty username)
curl -s -o /dev/null -w "%{http_code}" \
  -u ":<pat>" \
  https://dev.azure.com/<org>/_apis/projects?api-version=7.0
```

A `200` here confirms the credential is valid at the provider. A `401` or `403` confirms the problem is the credential, not the application. Any other code (network error, `404`, `500`) indicates a connectivity or endpoint problem — skip to Step 6.

**Stop condition:** you have a direct HTTP result. If `200`, the credential works; continue to Step 5 to find why the app rejects it. If `401`/`403`, the credential is the problem; fix it and go to Step 8.

---

## Step 5 — Check Expiry, Rotation, and Scopes

Even a credential that once worked can expire, be revoked, or lose required scopes.

- **Expiry:** OAuth access tokens typically expire in 1 hour. Refresh tokens may expire in 90 days. Check `exp` in a JWT payload (`base64 -d <<< $(cut -d. -f2 <<< "$TOKEN") | jq .exp`). For PATs, check the expiry date on the provider's token management page.
- **Rotation:** some providers auto-rotate tokens; some policies require manual rotation on a schedule. Check if the token was rotated recently and the new value hasn't been updated in the application's credential store.
- **Revocation:** tokens can be revoked by the account owner, an admin, or by a security event (password change, suspicious activity). Check the provider's active tokens list.
- **Scopes/permissions:** the integration may have originally worked with broader permissions that were later restricted. For each failing operation, confirm the token has the required scopes:
  - JIRA: `read:jira-work`, `write:jira-work`, `read:jira-user` (see Step 7 for JIRA-specific scope table)
  - GitHub: `repo`, `read:org`, `workflow` (depending on usage)
  - ADO: `Code (Read)`, `Work Items (Read & write)`, `Build (Read & execute)` (depending on usage)
- **Service account status:** confirm the account the token belongs to is still active, not locked, and still has access to the relevant projects/repositories.

**Stop condition:** you've confirmed token validity, rotation state, and required scopes.

---

## Step 6 — Verify API Version Compatibility

The provider may have changed their API or the endpoint structure.

- Check the API version in the request URL or header against the provider's current supported versions.
- JIRA Cloud: REST API v2 vs v3 have different response schemas for some resources.
- ADO: API versions are explicit in the URL (`?api-version=7.0`); some features require a minimum version.
- GitHub: `Accept: application/vnd.github.v3+json` header pinning may conflict with newer API behavior.
- Check provider changelogs or deprecation notices for the specific endpoint that is failing.
- If the application has not been updated in months and uses hardcoded API versions, cross-reference the provider's current docs.

**Stop condition:** API version is current and compatible with the operation being attempted.

---

## Step 7 — Trace the Full Request Path

If Steps 1–6 haven't found the root cause, the problem may be in how the application constructs or sends the request.

- Read the application code or configuration that builds the auth header. Confirm it matches what you validated in Steps 3–4.
- Check for middleware, proxy, or gateway layers that may strip, rewrite, or cache auth headers.
- Verify the base URL is correct (cloud vs. on-prem, correct org/workspace, no trailing slash issues).
- If available, enable request logging or use a network inspection tool to capture the actual outbound request.
- For OAuth flows: confirm the token refresh logic is running before the access token expires, not after a `401` is already returned.
- For webhook-based callbacks (GitHub, JIRA webhooks): confirm the callback URL is reachable and the HMAC or secret validation is using the right key.

**Stop condition:** the full request path is understood and matches the working direct test from Step 4.

---

## Step 8 — Fix and Regression-Test

Apply the fix and verify it resolves the original error without breaking related integrations.

1. Apply the targeted fix (update credential, fix scope, correct API version, fix auth header construction).
2. Re-trigger the originally failing operation. Confirm the error from Step 1 is gone.
3. Run the integration's health check (if one exists) and confirm it shows green.
4. Test at least one additional operation on the same integration to confirm the fix didn't break adjacent functionality.
5. If the root cause was an expired or rotated credential, document where the credential lives and add a note (comment on the issue or in the integration settings) about its rotation schedule to prevent recurrence.
6. Post a structured completion comment with: root cause found, fix applied, evidence of resolution (HTTP status code, health check result), and any follow-up recommendations (e.g., set up token rotation alerts).

---

## Provider-Specific Reference

### JIRA

| Auth type | Format | Notes |
|---|---|---|
| JIRA Cloud API token | Basic: `base64(email:api-token)` | Generated at id.atlassian.com; never expires unless revoked |
| JIRA Cloud OAuth 2.0 | Bearer: `<access-token>` | 1-hour expiry; requires refresh token flow |
| JIRA Data Center PAT | Bearer: `<personal-access-token>` | Generated in JIRA UI under Profile > Personal Access Tokens |
| JIRA Data Center basic | Basic: `base64(username:password)` | Deprecated; prefer PAT |

Key endpoints to test:
- Cloud: `GET /rest/api/3/myself`
- Data Center: `GET /rest/api/2/myself`
- Required scopes (Cloud OAuth): `read:jira-work`, `read:jira-user`, `write:jira-work`, `offline_access` (for refresh)

Common JIRA gotchas:
- Cloud API tokens are for the account, not the site — one token works across all your JIRA Cloud instances.
- The `accountId` field replaced `username` in v3 API responses; code using `name` from v2 breaks on v3.
- App passwords (Atlassian account setting) are different from API tokens.

---

### GitHub

| Auth type | Format | Notes |
|---|---|---|
| PAT (classic) | `token <pat>` or `Bearer <pat>` | Expires optionally; check for expiry date |
| Fine-grained PAT | `Bearer <pat>` | Resource-scoped; org approval may be required |
| GitHub App installation token | `Bearer <installation-token>` | 1-hour expiry; refresh via app auth flow |
| OAuth App token | `Bearer <oauth-token>` | Does not expire unless revoked |

Key endpoint to test: `GET https://api.github.com/user`

Common GitHub gotchas:
- Fine-grained PATs require explicit org approval before they can access org resources — a valid token still gets `403` if unapproved.
- Classic PATs with SSO-enabled orgs need SSO authorization on the token (Settings > Authorized OAuth Apps).
- Rate limits (`429`) look like auth failures in some error handlers — check `X-RateLimit-Remaining` header.
- GitHub Apps: the installation token endpoint requires a signed JWT from the app's private key, not the installation token itself.

---

### Azure DevOps (ADO)

| Auth type | Format | Notes |
|---|---|---|
| PAT | Basic: `base64(:<pat>)` (empty username, colon prefix) | Scoped to org; explicit expiry date required |
| OAuth 2.0 | Bearer: `<access-token>` | 1-hour expiry; `offline_access` scope for refresh |
| Service principal | Bearer: `<aad-access-token>` | Azure AD-issued; requires ADO org-level AAD integration |

Key endpoint to test: `GET https://dev.azure.com/<org>/_apis/projects?api-version=7.0`

Common ADO gotchas:
- PATs are org-scoped by default; a PAT for `org-A` will get `401` on `org-B` even if the same user has access.
- The `All Organizations` PAT scope is a separate checkbox — not selected by default.
- ADO API version must be in the URL (`?api-version=7.0`); omitting it falls back to a very old version that may behave differently.
- Service principal access requires an explicit step to add the SP to the ADO org (Organization Settings > Users) — Azure RBAC alone is not sufficient.
- `203 Non-Authoritative Information` from ADO often means the request was redirected to a sign-in page — treat as a `401`.

---

## SSE / WebSocket Streaming Contract Debugging

Real-time streaming failures are distinct from REST auth failures. The connection may succeed (HTTP 200, WebSocket handshake OK) while the data arriving is wrong — a mismatched event name, a missing field, or a schema that diverged between producer and consumer. Use this section when a streaming integration is connected but the consumer isn't processing events correctly, events appear to be silently dropped, or the consumer crashes on arrival.

### Step A — Confirm the Transport Layer Is Healthy

Before examining the data, verify the connection itself is working.

**SSE (Server-Sent Events):**
```bash
# Verify Content-Type and streaming response
curl -N -H "Accept: text/event-stream" https://<producer>/stream
# Expect: Content-Type: text/event-stream, chunked transfer, lines starting with "data:"
```

**WebSocket:**
```bash
# Use wscat or websocat for a quick smoke test
npx wscat -c wss://<producer>/ws
# Expect: 101 Switching Protocols, then JSON/text frames
```

Check for:
- Correct `Content-Type: text/event-stream` (SSE) or `101 Switching Protocols` (WS)
- Heartbeat frames arriving on schedule (ping/pong for WS, `: keepalive` comment lines for SSE)
- Proxy/load-balancer timeout — long-lived connections are frequently killed by 60-second idle timeouts; confirm the producer is sending keepalives within that window

### Step B — Capture and Inspect the Raw Event Stream

Intercept the exact bytes the producer is sending before any consumer parsing occurs.

**SSE — dump raw frames:**
```bash
curl -N -H "Accept: text/event-stream" https://<producer>/stream 2>/dev/null | head -50
```

**WebSocket — log incoming frames:**
```js
// Minimal browser snippet — paste in DevTools console
const ws = new WebSocket('wss://<producer>/ws');
ws.onmessage = (e) => console.log('frame:', e.data);
ws.onerror = (e) => console.error('error:', e);
```

From the raw output, extract:
- **Event type** — SSE uses `event: <type>` lines; WS events carry a `type` or `event` field in the JSON payload
- **Field names and shapes** — record the exact keys and value types you see in the raw frame

### Step C — Compare Raw Output Against Consumer Expectations

This is where most streaming bugs live. The producer's schema changed, or the consumer was written against a stale contract.

Checklist:
- **Event type name** — does the consumer listen for `"message"` but the producer sends `"update"`? SSE consumers using `addEventListener('update', …)` receive nothing if the producer sends unnamed `data:` lines (which default to `"message"`).
- **Field casing** — `userId` vs `user_id`, `eventType` vs `event_type`
- **Nested vs flat** — producer wraps payload in `{ "data": { … } }`; consumer reads top-level fields directly
- **Optional fields treated as required** — producer stopped sending a field that the consumer doesn't null-check
- **Numeric vs string IDs** — producer changed `id: 123` to `id: "123"` after a DB migration; consumer does strict equality

Look at the consumer code that handles incoming frames. Confirm every field it reads by name against the fields actually present in the raw stream.

### Step D — Check Reconnection and State Recovery

Many streaming bugs only appear after a disconnection, not on first connect.

- **SSE `Last-Event-ID`** — the browser sends this header on reconnect; the producer must replay missed events based on it. If the producer ignores the header or the consumer doesn't set `EventSource` with correct credentials, events are lost on reconnect.
- **WebSocket reconnect logic** — confirm the client re-subscribes (sends a `subscribe` message) after reconnecting; the server does not automatically re-send subscriptions from the previous session.
- **Event ordering** — if the consumer processes events out of order (e.g., due to a queue), check whether events carry a sequence number or timestamp for ordering. Missing sequence numbers can cause stale state to overwrite fresh state.
- **Buffer overflow/backpressure** — a slow consumer may cause the producer to drop events. Check producer logs for backpressure warnings or buffer-full errors.

### Step E — Fix and Verify

1. Identify whether the mismatch is in the **producer** (wrong event name/shape) or **consumer** (stale parsing logic). Fix at the correct layer.
2. After fixing, use the `curl -N` or `wscat` test from Step A to confirm the raw stream matches what the consumer expects.
3. Simulate a reconnect to confirm event recovery works: disconnect mid-stream and verify the consumer resumes correctly.
4. Post a completion comment noting: which field/event name mismatched, which side was fixed, and whether reconnection was tested.

---

## API Contract Drift Detection

Use this checklist when touching an integration boundary — any time you modify a producer endpoint or a consumer that calls it. The goal is to catch drift before it ships, not after a consumer breaks in production.

### Pre-Touch Checklist (before changing producer or consumer code)

- [ ] **Locate the contract definition** — find the OpenAPI spec, GraphQL schema, or equivalent contract document for the endpoint you're about to change. If one doesn't exist, snapshot the current request/response shapes now before editing.
- [ ] **Identify all consumers** — search for all code that calls this endpoint (grep for the path or method name). List them. A change that looks local to the producer may break multiple consumers.
- [ ] **Identify all producers** — for consumer-side changes, confirm which producer version/environment you're coding against. `staging` and `production` may have different schemas.

### Change Classification

Classify the change before making it:

| Change type | Breaking? | Action required |
|---|---|---|
| Add optional response field | No | Document; consumers can ignore |
| Remove response field | **Yes** | Update all consumers before or with producer |
| Rename field | **Yes** | Update all consumers; consider aliasing during migration |
| Change field type (e.g., int → string) | **Yes** | Update consumers; coordinate deploy order |
| Add required request field | **Yes** | All callers must supply the new field |
| Add optional request field with default | No | Document default behavior |
| Change enum values | **Yes** | Search for hardcoded enum values in consumers |
| Change HTTP status code | **Yes** | Search for status-code-specific handling in consumers |
| Add SSE event type | No (additive) | Document; consumers using `addEventListener` ignore unknown types |
| Rename SSE event type | **Yes** | Update all `addEventListener` calls |
| Change SSE field names | **Yes** | Full Step B–C audit above |

### Post-Change Verification

- [ ] **Run contract tests or schema validation** — if the project has Pact tests, OpenAPI schema validation, or similar contract-testing tooling, run it now. Don't skip this if it exists.
- [ ] **Check test assertions against live shapes** — search the test suite for hardcoded field names, values, or status codes from this endpoint. Stale assertions pass even when the integration is broken because they're asserting against mocked responses that were never updated.
- [ ] **Verify mock/stub accuracy** — any test mock for this endpoint should reflect the current producer shape. If it doesn't, update it.
- [ ] **Smoke test the integration end-to-end** — run one real request through the full path (producer → network → consumer) in a non-production environment. Don't rely solely on unit tests for integration boundary changes.
- [ ] **Check deployment order** — for breaking changes, the consumer that handles both old and new shapes should deploy before the producer change. Or use a feature flag to cut over atomically.
- [ ] **Update the contract document** — if an OpenAPI spec or contract file exists, update it as part of the same PR. Out-of-date specs are worse than no spec.

---

*TEC Custom Skill — maintained by the Deltek Technical Services Engineering team.*
