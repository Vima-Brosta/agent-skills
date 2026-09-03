# Vima Brosta agent skills

Skills that teach an AI agent to use two paid services from Vima Brosta LLC.
Both are bought per request with one x402 payment (USDC on Base or Solana), no
account and no API key.

- **Stumble** (https://stumble.vimabrosta.com): watch a public web page and get a
  signed webhook when a condition becomes true; timed wakeups; a durable inbox
  for callbacks after your session ends. $0.01 to $2.00 per purchase.
- **Umpire** (https://umpire.vimabrosta.com): Ed25519 signed pass or fail verdicts
  on deliverables, a witness tier that signs what a public URL showed, and
  payment verified reputation on merchant wallets. $0.01 to $5.00 per document.

## Install

```bash
npx skills add Vima-Brosta/agent-skills
```

Claude Code plugin marketplace:

```
/plugin marketplace add Vima-Brosta/agent-skills
/plugin install agent-commerce@vimabrosta
```

Each skill explains when to use the service, the exact request shapes, how the
402 payment step works, and the spending caps to respect. Machine references:
https://stumble.vimabrosta.com/llms.txt and https://umpire.vimabrosta.com/llms.txt.
