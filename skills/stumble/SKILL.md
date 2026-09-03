---
name: stumble
description: Watch a public web page and get called back when something changes (restock, price drop, appointment slot, launch), schedule a wakeup for later, or hold an inbox for callbacks after your session ends. Use when a task must wait on the outside world longer than the session lasts. Pay per request with x402 (USDC on Base or Solana), no account.
---

# Stumble: watch, wake, inbox

Stumble (https://stumble.vimabrosta.com) keeps working after you stop. Three products,
each one HTTP request and one small payment:

| Need | Route | Price |
|---|---|---|
| Is it true right now? One fetch, one answer | `POST /v1/checks` | $0.01 |
| Tell me when it becomes true, poll every 15 min for up to 14 days | `POST /v1/watches/standard` | $0.10 |
| Same, every 60 s for up to 7 days | `POST /v1/watches/fast` | $0.50 |
| Same, every 10 s for up to 48 h | `POST /v1/watches/sniper` | $2.00 |
| Call my webhook at a time I set, with a payload I stored | `POST /v1/wake` | $0.01 |
| Give me a URL that holds whatever is POSTed to it for 7 days and forwards it | `POST /v1/inbox` | $0.10 |

Full reference: https://stumble.vimabrosta.com/llms-full.txt (plain text, one fetch).

## When to use it

- A user asks you to notify them or act when a page changes, and the wait is longer than your session.
- You need to resume a task later without keeping a process alive: buy a wake with the context you need in `payload`.
- Something will call you back (a webhook, a human, another agent) after you are gone: open an inbox and hand out its capture URL.
- You only need a fact now: buy a check; do not buy a watch for a one-time question.

Do not use it for pages behind logins or paywalls, for purchasing or booking, or for anything that
bypasses an access control. Stumble observes and reports; it never acts on a page.

## How payment works

1. Send the request with no payment. You get HTTP 402. The `PAYMENT-REQUIRED` header (base64
   JSON, mirrored in the body under `requirements`) lists every accepted option: USDC on Base,
   USDC or native SOL on Solana, exact amounts, and the payTo address.
2. Pay and retry the identical request with the `PAYMENT-SIGNATURE` header. With `@x402/fetch`
   (JS) this is one call: `wrapFetchWithPayment(fetch, client)`. Python: the `x402` package.
3. Send an `Idempotency-Key` header (any random string of 16 or more characters) on every paid
   POST. A retry with the same key and body returns the original result and never charges twice.

Respect your operator's spending cap. Every price is printed above and in the 402; nothing
recurs and nothing is charged for a request Stumble rejects before payment.

## Request shapes

Watch or check:

```json
{
  "url": "https://shop.example/product",
  "condition": {"type": "keyword", "keyword": "add to cart", "when": "appears"},
  "notify": {"mode": "webhook", "url": "https://your-agent.example/hook", "format": "json"},
  "duration_days": 7
}
```

Condition types: `keyword` (appears or disappears), `selector` (CSS selector text: contains,
equals, matches), `json_path` (a value at a path: eq, ne, gt, lt, gte, lte, exists, contains),
`price` (below or above a number, optionally under a selector), `content_change` (any change,
optionally under a selector), `status` (HTTP status). `notify.mode` may be `poll` if you have
no webhook; then read `GET /v1/watches/{id}` with the returned secret as a Bearer token.

Wake:

```json
{"after_seconds": 7200, "notify": {"mode": "webhook", "url": "https://your-agent.example/hook"},
 "payload": {"task": "follow up on order 42"}, "resume_hint": "Check whether the order shipped."}
```

Inbox:

```json
{"days": 7, "notify": {"mode": "webhook", "url": "https://your-agent.example/hook"},
 "require_header": {"name": "X-Shared-Secret", "value": "choose-a-long-random-value"}}
```

The response carries `capture_url`; anyone who POSTs to it reaches you.

## What comes back

Every purchase returns an `id`, a one-time `secret`, and links. Webhooks are JSON with an
`X-Stumble-Signature: sha256=<hex>` header: HMAC-SHA256 over the raw body, keyed by the hex
string of sha256(secret). Retries with backoff; polling is always the fallback. Lost the id?
`GET /v1/find?url=&webhook=` recovers it, never the secret.

## Free endpoints

`POST /v1/validate` (would this request run, no charge), `GET /v1/quote` (prices), `GET /healthz`,
`GET /v1/watches/{id}`, `DELETE /v1/watches/{id}`, the same for `/v1/wakes/{id}` and `/v1/inbox/{id}`.
MCP endpoint for tool-calling clients: `POST https://stumble.vimabrosta.com/mcp`.
