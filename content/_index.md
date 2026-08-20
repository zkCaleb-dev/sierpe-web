---
title: Home
---

No forks, no custom code, no vendor. Configuration is data: register
contracts at runtime through an authenticated API, and Sierpe classifies
them, backfills their **full history** — including ranges no RPC serves
anymore — and follows the tip.

```text
1. Deploy the container next to an empty Postgres
2. POST /v1/contracts {"contract_id": "C...", "from": "genesis"}
3. Sierpe discovers the contract's events from its on-chain spec
   and walks its history backwards, honestly declaring coverage
4. GET /v1/contracts/C.../events?topic0=...&after=<cursor>
```

**Honest by construction.** Coverage and gaps are first-class data, declared
in every API response. An empty page always tells you whether there is
nothing — or whether it just hasn't been indexed yet.
