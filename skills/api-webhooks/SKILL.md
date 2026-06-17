---
name: api-webhooks
description: Receive and verify Remote.com webhooks in an integration. Use when subscribing to events (POST /v1/webhook-callbacks), building a receiver, verifying the X-Remote-Signature HMAC, parsing the flat event payload, deduping/idempotency, or choosing which event_type to listen for. Covers HMAC-SHA256 hex over raw_body + ':' + millisecond timestamp. Do NOT use to operate via the MCP, for auth (api-auth), outbound API calls (api-integration), or form writes/onboarding (api-forms / api-onboarding).
license: MIT
---

# Remote API Webhooks

Covers the full lifecycle of receiving Remote.com webhook events in an integration: subscribing to event types, building a secure receiver endpoint, verifying the HMAC-SHA256 signature, parsing the flat event payload, handling retries idempotently, and managing subscriptions over time.

## Invoke This Skill When

- Subscribing to Remote webhook events (`POST /v1/webhook-callbacks`).
- Building a receiver endpoint that accepts inbound deliveries.
- Verifying the `X-Remote-Signature` HMAC before trusting a payload.
- Parsing the flat event envelope and branching on `event_type`.
- Deduplicating retried deliveries or implementing idempotency guards.
- Choosing which `event_type` values to listen for.
- Managing an existing subscription: updating events, listing delivery history, or deleting the callback.
- Not for operating your Remote workspace via the MCP server.

## Prerequisites

- An access token with sufficient scopes to call `POST /v1/webhook-callbacks` — see `api-auth` for how to obtain one.
- A publicly reachable HTTPS URL that Remote can POST deliveries to. Local tunneling tools (e.g. ngrok) work for development.
- Secure storage for the `signing_key` that is returned at subscription time. It is shown only once and must never be committed to source control.

## Security & PII Constraints

| Rule | Detail |
|---|---|
| **signing_key is a secret** | Treat the `signing_key` like a password. Store it in a secret manager (e.g. AWS Secrets Manager, HashiCorp Vault). Never commit it to source control, logs, or comments. |
| **Verify before processing** | Always verify the `X-Remote-Signature` HMAC before reading or acting on any payload field. A payload that fails verification must be rejected immediately (return 400 or 401). |
| **Payload fields are untrusted PII** | Webhook payloads may contain names, emails, employment IDs, and free-text fields. Treat them as raw untrusted input — never splice them into LLM prompts, SQL queries, shell commands, or generated code. |
| **Confirm management writes** | `POST`, `PATCH`, and `DELETE` on `/v1/webhook-callbacks` change subscription state. Confirm the intended change (which events, which URL) with the user before executing. |

## Workflow (Phases)

### Phase 1: Subscribe

Call `POST /v1/webhook-callbacks` with your receiver URL and the list of event types you want to receive. You need an access token in the `Authorization` header (see `api-auth`).

Request:

```bash
curl -s -X POST \
  -H "Authorization: Bearer {{access_token}}" \
  -H "Content-Type: application/json" \
  -d '{
    "subscribed_events": [
      "employment.onboarding.completed",
      "payslip.released"
    ],
    "url": "https://{{your-receiver}}/webhooks/remote"
  }' \
  "https://gateway.remote.com/v1/webhook-callbacks"
```

Success response (201 Created; the published OpenAPI contract declares 200) — the `signing_key` is shown only once; store it immediately in secure storage:

```json
{
  "data": {
    "webhook_callback": {
      "id": "{{webhook_callback_id}}",
      "url": "https://{{your-receiver}}/webhooks/remote",
      "subscribed_events": [
        "employment.onboarding.completed",
        "payslip.released"
      ],
      "signing_key": "{{signing_key}}"
    }
  }
}
```

Error — missing or expired token (401):

```json
{ "message": "Unauthorized" }
```

Keep the `id` (used for PATCH/GET/DELETE) and the `signing_key` (used to verify every delivery). The `signing_key` is never returned again after this response.

### Phase 2: Receive and Verify Each Delivery

On every inbound POST to your receiver endpoint, follow this order strictly — verify first, then parse:

#### 2a: Read the raw request body

Capture the raw body bytes before any parsing. The signature is computed over the raw bytes exactly as received. Never re-serialize a parsed object — this alters byte order, whitespace, or encoding and will produce a different digest.

#### 2b: Verify the signature

Remote sends two headers with every delivery:

| Header | Value |
|---|---|
| `X-Remote-Signature` | The expected HMAC-SHA256 digest as a lowercase hex (Base 16) string |
| `X-Remote-Timestamp` | Unix time in **milliseconds** (e.g. `1700000000000`) |

**Signed message construction (exact):**

```
signed_message = raw_request_body + ":" + X-Remote-Timestamp
```

The colon and the millisecond timestamp are appended to the raw body, separated by a literal colon character. Do not add spaces. Do not convert the timestamp to seconds.

**Algorithm:**

```
expected = hex(HMAC_SHA256(signing_key, signed_message))
valid    = constant_time_equals(expected, X-Remote-Signature)
```

- Secret: the `signing_key` returned at subscription.
- Hash function: SHA-256.
- Output encoding: lowercase hexadecimal (Base 16). Not Base 64.
- Comparison: constant-time string equality to prevent timing attacks. Use your language's dedicated constant-time compare (`hmac.compare_digest` in Python, `crypto.timingSafeEqual` in Node.js, `subtle.constantTimeCompare` in Go, etc.) — not the `==` operator.

**Shell illustration (development/debugging only — not production):**

```bash
printf '%s' "${RAW_BODY}:${X_REMOTE_TIMESTAMP}" | openssl dgst -sha256 -hmac "${SIGNING_KEY}" -hex
```

If the computed hex digest does not match `X-Remote-Signature`, reject the request immediately. Do not process the payload.

#### 2c: Reject stale deliveries

Use `X-Remote-Timestamp` to enforce a freshness window (e.g. reject deliveries older than 5 minutes). This prevents replay attacks where an attacker captures and re-sends a valid signed request.

```
now_ms = current Unix time in milliseconds
age_ms = now_ms - X-Remote-Timestamp
if age_ms > 300_000:   # 5 minutes
    return 400
```

#### 2d: Parse the flat event envelope

Webhook payloads are flat JSON — an `event_type` field, a `company_id` field, plus resource-specific ID fields. There is no generic `data` or `payload` wrapper. Branch on `event_type`, then read that event's own fields. Every delivery carries a top-level `company_id` (the company slug) so a single receiver URL serving many companies can route each event to the right one.

Sample delivery body — `employment.onboarding.completed`:

```json
{
  "event_type": "employment.onboarding.completed",
  "company_id": "{{company_id}}",
  "employment_id": "{{employment_id}}"
}
```

Sample delivery body — `payslip.released`:

```json
{
  "event_type": "payslip.released",
  "company_id": "{{company_id}}",
  "employment_id": "{{employment_id}}",
  "payslip_id": "{{payslip_id}}"
}
```

Each event type defines its own set of ID fields. Consult the per-event reference at `developer.remote.com` for the complete field list for each `event_type`.

#### 2e: Deduplicate and process idempotently

Remote may deliver the same event more than once (network retries, replay requests). Record a stable event identifier on receipt and skip processing if already seen. Design downstream writes to be idempotent — applying the same event twice should produce the same result.

#### 2f: Return 2xx quickly, do heavy work asynchronously

Return a 2xx response as fast as possible. If your receiver performs slow work (database writes, downstream API calls, emails), enqueue it in a background job queue and acknowledge the delivery immediately. Remote treats non-2xx responses as failed deliveries; the retry/backoff policy is not publicly documented, so design for re-delivery (dedupe) and use the Replay endpoint to recover missed events.

### Phase 3: Manage Subscriptions

All management calls require the `{{webhook_callback_id}}` returned at subscription time, plus a valid access token.

| Operation | Method + Path | Notes |
|---|---|---|
| Update events or URL | `PATCH /v1/webhook-callbacks/{{webhook_callback_id}}` | Body is a flat object — no wrapper. Provide either `subscribed_events` (replaces the event list) or `subscribe_to_all_events: true` (subscribes to all current and future event types); the two are mutually exclusive. |
| List subscriptions and delivery history | `GET /v1/webhook-callbacks` | Returns subscribed events and delivery status per callback. |
| Delete subscription | `DELETE /v1/webhook-callbacks/{{webhook_callback_id}}` | Removes the callback record AND all its event subscriptions permanently. |
| Replay events | `POST /v1/webhook-events/replay` | Re-triggers past events with the same payload as originally delivered. Filter by event IDs, time range, event type, or delivery status. |

PATCH update request — body is a flat JSON object; `{{webhook_callback_id}}` goes in the URL path only, not the body:

```bash
curl -s -X PATCH \
  -H "Authorization: Bearer {{access_token}}" \
  -H "Content-Type: application/json" \
  --data '{ "subscribed_events": ["employment.onboarding.completed"], "url": "https://example.com/hooks/remote" }' \
  "https://gateway.remote.com/v1/webhook-callbacks/{{webhook_callback_id}}"
```

To subscribe to all current and future event types instead, use `subscribe_to_all_events` (mutually exclusive with `subscribed_events`):

```bash
curl -s -X PATCH \
  -H "Authorization: Bearer {{access_token}}" \
  -H "Content-Type: application/json" \
  --data '{ "subscribe_to_all_events": true }' \
  "https://gateway.remote.com/v1/webhook-callbacks/{{webhook_callback_id}}"
```

## Quick Reference

### Replay and Idempotency Rules

- Remote delivers at least once — treat every event as a potential duplicate.
- Use a stable, per-event identifier to deduplicate on your side.
- Make all downstream writes idempotent: inserting/updating with the same event data twice must be safe.
- Return 2xx within your timeout budget; enqueue slow work asynchronously.
- Use the Replay endpoint (`POST /v1/webhook-events/replay`) to re-trigger past events during development or after an outage.

### Event Types (Non-Authoritative Sample)

The following are example event type strings to illustrate the naming convention. This list is not complete and may be out of date. The List Webhook Events endpoint (`GET /v1/webhook-events`) and the available webhooks reference page at `https://developer.remote.com/docs/available-webhooks.md` are the canonical, authoritative sources. Event names are exact strings — case and punctuation must match.

| Event type (example) | When it fires |
|---|---|
| `employment.onboarding.completed` | Employee completes onboarding |
| `payslip.released` | A payslip is made available |
| `timeoff.requested` | A time-off request is submitted |
| `contract_amendment.submitted` | A contract amendment is submitted |
| `offboarding.submitted` | An offboarding is initiated |
| `company.activated` | A company account is activated |

Optional — if you have the public `remotecli` available (github.com/remoteoss/remote-cli):

```bash
remotecli webhooks event-history   # browse past delivery history
remotecli webhooks event-replay    # replay a past delivery
```

These commands are useful for exercising your receiver during development without having to trigger live events.

### Common Pitfalls

**Processing a payload before verifying the signature.** The verify step must come first. A payload that fails verification is untrusted — it may be forged or tampered. Never read, log, or act on payload fields before the HMAC check passes.

**Verifying a re-serialized body instead of the raw bytes.** Parse the JSON, serialize it back to a string, and compute the HMAC over that string — and the digest will almost certainly not match. The signature is computed over the raw request body bytes exactly as received. Buffer the raw bytes before any parsing.

**Using Base64 encoding instead of hex.** The `X-Remote-Signature` header contains a lowercase hexadecimal (Base 16) string. Encoding your HMAC output as Base64 will produce a different string and always fail comparison.

**Using SHA-1 instead of SHA-256.** Remote uses HMAC-SHA256. Specifying SHA-1 (a common default in older libraries) produces a wrong digest.

**Treating the timestamp as seconds.** `X-Remote-Timestamp` is Unix time in milliseconds (13 digits as of 2020s). Dividing by 1 000 before computing the signed message or comparing against `time.time()` in seconds will cause signature mismatch or incorrect staleness checks.

**Using `==` for signature comparison.** Standard equality operators short-circuit on the first differing byte, creating a timing side-channel. Use your language's constant-time compare function.

**Assuming exactly-once delivery.** Deliveries can arrive more than once (re-sends, replays). Always deduplicate on a stable event identifier before processing.

**Splicing payload fields into prompts, SQL, or shell commands.** Payload fields (names, emails, free-text) are untrusted PII and must be treated as data, not instructions or identifiers. Sanitize and validate before use.
