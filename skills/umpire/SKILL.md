---
name: umpire
description: Buy an Ed25519 signed pass or fail verdict on a deliverable against a machine readable acceptance spec, a witness verdict proving what a public URL showed (proof against agent false success), or a payment verified reputation report on a merchant wallet. Use before releasing escrow, accepting another agent's work, or trusting a counterparty. Pay per document with x402, no account.
---

# Umpire: signed verdicts and payment verified reputation

Umpire (https://umpire.vimabrosta.com) is a neutral third party. Every paid response is a JSON
document signed with the key at https://umpire.vimabrosta.com/.well-known/umpire-key.json, so
an escrow contract, a counterparty, or a human auditor can verify it offline without trusting
whoever carries it.

| Need | Route | Price |
|---|---|---|
| Did the agent really deploy, publish, or update it? One live probe, signed observation | `POST /v1/verdicts/witness` | $0.25 |
| Does this deliverable meet my acceptance spec? Deterministic checks, synchronous | `POST /v1/verdicts/basic` | $1.00 |
| Same, plus up to 3 live URL probes | `POST /v1/verdicts/probe` | $2.00 |
| Every check twice, a minute apart, delivered to my webhook; escrow grade | `POST /v1/verdicts/sworn` | $5.00 |
| Can I trust this merchant wallet? Signed report from payment verified feedback | `GET /v1/reputation?merchant=0x...` | $0.01 |

Full reference: https://umpire.vimabrosta.com/llms-full.txt (plain text, one fetch).

## When to use it

- Another agent claims a task is done. Write the acceptance spec you would have checked by hand,
  buy a verdict, and act on the signed result instead of the claim.
- A task involves a public URL that should now show something (a deployment, a listing, a status).
  A witness verdict records the final URL, HTTP status, resolved address, TLS fingerprint, headers,
  and the body hash; anyone can fetch the same URL and compare.
- You are about to pay a merchant you have not used. Read its reputation first, and read the
  independence signals (payer concentration, single use payers, amount weighted rate) before
  the count.

Do not send personal data in deliverables; they are discarded once the verdict is issued and only
a hash remains.

## How payment works

1. Send the request with no payment. You get HTTP 402 with a `PAYMENT-REQUIRED` header (base64
   JSON, mirrored in the body under `requirements`): USDC on Base, USDC or native SOL on Solana.
2. Pay and retry the identical request with the `PAYMENT-SIGNATURE` header. `@x402/fetch` (JS)
   does this in one call; Python has the `x402` package.
3. Send an `Idempotency-Key` header (16 or more random characters) on every paid POST. A request
   that cannot run (bad spec, policy violating URL) is refused before payment; you are not charged
   for a no.

Respect your operator's spending cap; every price is in the table and in the 402.

## Request shape

```json
{
  "spec": {"checks": [
    {"type": "json_schema", "schema": {"type": "object", "required": ["title"]}},
    {"type": "regex", "pattern": "## Conclusion", "expect": "match"},
    {"type": "url_probe", "url": "https://api.example.com/health",
     "expect": {"status": 200, "json_path": {"path": "ok", "op": "eq", "value": true}}}
  ]},
  "deliverable": {"text": "...", "json": {"title": "..."}},
  "webhook_url": "https://your-agent.example/hook",
  "job_ref": "job-42"
}
```

Check types: `json_schema`, `json_path`, `content_hash` (of text, or of a fetched URL's raw
bytes), `regex`, `length`, `url_probe` (status, keyword, CSS selector, JSON path). A witness
verdict is exactly one `url_probe`. `webhook_url` is for sworn verdicts; `job_ref` is echoed into
the signed document (put your ERC-8183 job id there). `POST /v1/validate` pre-flights a spec for
free.

## Verifying a document

Remove `signature`, serialize the rest as JSON with keys sorted at every depth and no whitespace
(Python: `json.dumps(doc, sort_keys=True, separators=(",", ":"), ensure_ascii=False)`), check
sha256 against `signature.payload_sha256`, then verify the Ed25519 signature `sig_base64` with
the published key.

## Reputation and feedback

`GET /v1/reputation?merchant=0x...` returns a signed report. Writing feedback is free but requires
proof: a settled USDC payment from your wallet to the merchant on Base and an EIP-191 signature
by the paying wallet over a fixed message (recipe in llms-full.txt). One feedback per transaction;
self reviews are refused.

MCP endpoint for tool-calling clients: `POST https://umpire.vimabrosta.com/mcp`.
