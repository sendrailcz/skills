---
name: sendrail
description: Send SMS, verification codes (OTP) and check phone numbers through the Sendrail API (sendrail.cz), or write an integration with it. Use when the user wants to send a text message, verify a phone number, add SMS notifications to an application, handle delivery webhooks or STOP opt-outs, or mentions Sendrail.
---

# Sendrail

Sendrail is an SMS API (Czech provider, own network of GSM gateways). Base URL `https://api.sendrail.cz`, JSON in and out, key in `Authorization: Bearer <key>`. Prices are in CZK without VAT, per segment.

The complete reference is one Markdown file: fetch `https://sendrail.cz/docs.md` whenever you need a field, an error code or anything not written below. OpenAPI: `https://api.sendrail.cz/v1/openapi.json`.

## Before you send anything

1. **Find the key.** Look for `SENDRAIL_API_KEY` in the environment. Never ask the user to paste a key into the conversation and never write a key into a file that is committed; read it from the environment or a secret store.
2. **Prefer a test key while developing.** A key starting with `sr_test_` behaves exactly like a live one but sends nothing and charges nothing. A key starting with `sr_live_` sends real messages for real money.
3. **A real SMS reaches a real person and cannot be taken back.** With a live key, confirm the recipient and the text with the user before the first send, unless they already gave both explicitly. Use `"dry_run": true` to check and price a message without sending.
4. If the Sendrail MCP server is connected (tools `send_message`, `estimate_message`, `check_number`, …), use the tools instead of raw HTTP.

## Send a message

```
curl -sS -X POST https://api.sendrail.cz/v1/messages \
  -H "Authorization: Bearer $SENDRAIL_API_KEY" -H "Content-Type: application/json" \
  -d '{"to": "+420777123456", "message": "Your order is ready.", "idempotency_key": "order-1042-ready"}'
```

- `to`: international format. Nine digits without a prefix are taken as Czech.
- Answer `202` with `id`, `status: "queued"`, `segments`, `price`. Typical time to `sent` is about ten seconds.
- **Always send an `idempotency_key`** derived from the business event (order id + purpose). After a timeout repeat the same request; the same key returns the first message with `"duplicate": true` and nothing is sent twice.
- The same text to the same number from the same key within five minutes is not sent again either.
- Give the HTTP client a timeout of at least 20 seconds. Limit: 60 messages a minute per key.

Segments: 160 characters in GSM-7 (153 per part when split). One character outside GSM-7 (Czech diacritics, emoji) switches to UCS-2: 70 characters (67 per part). At most 10 segments. If cost matters, tell the user that dropping diacritics more than halves the number of segments.

## Status

`GET /v1/messages/{id}` → `scheduled` | `queued` | `sent` (final: handed to the mobile network) | `failed` | `cancelled` | `rejected` (final, refunded). `POST /v1/messages/{id}/cancel` works until the message leaves.

Do not poll in production. Set a webhook URL on the key in the dashboard; Sendrail posts `{"event":"message.final","message":{…}}` and signs the raw body:

```
expected = "sha256=" + hmac_sha256_hex(signing_secret, raw_body)
# compare with header X-Sendrail-Signature in constant time
```

Answer `2xx` within 5 seconds. The same event can arrive more than once; make the handler idempotent by message `id`. Other events on the same URL: `message.received`, `call.missed`, `optout.created`.

## Verification codes (OTP)

Do not generate and store codes yourself; Sendrail does it.

```
POST /v1/verify              {"to": "+420777123456", "app": "My Shop", "lang": "en"}   → 202 {"id": "ver_…", "status": "pending", …}
POST /v1/verify/{id}/check   {"code": "482913"}                                        → 200 {"valid": true|false, "status": …, "attempts_left": n}
```

- `valid` is `true` exactly once: right code, before expiry (`ttl` 60–900 s, default 300), within five tries.
- Do the check on the server. Never return the code to the browser or app of the person being verified. With a test key the answer contains `test_code` for automated tests; never forward it either.
- At most 3 codes per number in 10 minutes (`429 verify_rate_limited`). Rate-limit your own endpoint that triggers a code as well, per number and per IP, or somebody will spend the account's credit.

## Check a number

`POST /v1/numbers/check` with `{"number": "777 123 456"}` (or `{"numbers": [...]}` up to 100). Free. Returns the normalized number, country, `type` (`mobile`/`landline`/`special` for CZ and SK), the price per segment and whether the key may write there. It is not a live network lookup; it cannot tell whether the phone is switched on.

## Mass mailings and opt-outs

- Newsletters, campaigns and any larger batch not triggered by a single user action **must** be sent as bulk: `"type": "bulk"` on a message, or `POST /v1/messages/batch` with up to 500 recipients. Bulk messages are priced by the bulk line of the price list, have the lowest priority and leave slowly in daytime hours. Sending a campaign as ordinary messages breaks the terms.
- The sender needs the recipients' consent and must offer a way out. A person who replies `STOP`, `Odhlásit` or `Unsubscribe` is put on the account's opt-out list (bulk scope) automatically; a message to a listed number answers `403 recipient_opted_out`. Manage the list with `GET/POST /v1/optouts` and `DELETE /v1/optouts/{number}`.

## Templates

Stored texts with `{{name}}` places: `PUT /v1/templates/{name}` with `{"message": "…"}`; send with `{"to": "…", "template": "order-ready", "variables": {"name": "Jana"}}`.

## Errors

`{"error": {"code": "…", "message": "…"}}`. What to do:

| HTTP | Meaning | Action |
|---|---|---|
| 400 | bad request (`invalid_number`, `message_too_long`, `missing_variable`, …) | fix it; a retry fails again |
| 401 | `invalid_api_key` | wrong or paused key |
| 402 | `insufficient_credit`, `trial_exhausted` | the owner tops up credit in the dashboard; do not retry |
| 403 | `country_not_allowed`, `recipient_not_allowed`, `recipient_opted_out`, `trial_recipient` | a rule of the key or account; do not work around it |
| 429 | `rate_limited`, `daily_limit_reached`, `verify_rate_limited` | slow down, retry later |
| 502 | `gateway_error` | nothing was charged; retry with the same `idempotency_key` |

A new account is in trial mode: 100 free messages, only to the phone number verified at sign-up (`403 trial_recipient` otherwise) until the first top-up.

## Writing an integration: checklist

- Key from the environment, never in the repository; a test key in CI.
- `idempotency_key` on every send; timeout ≥ 20 s; retry only on network errors, 429 and 5xx.
- Final status through the signed webhook; verify the signature on the raw body before parsing.
- OTP through `/v1/verify`, checked on the server, with your own rate limit in front.
- Campaigns as bulk, with consent and opt-out handling.
- For a key given to a script or an AI agent, suggest that the owner sets limits on it in the dashboard (messages and spend per 24 hours, allowed recipients).

## MCP server

`https://api.sendrail.cz/mcp` (Streamable HTTP). Claude Code:

```
claude mcp add --transport http sendrail https://api.sendrail.cz/mcp --header "Authorization: Bearer $SENDRAIL_API_KEY"
```

Connectors that cannot send a header (Claude on the web, ChatGPT) sign in with OAuth: enter only the address. Details: https://sendrail.cz/ai
