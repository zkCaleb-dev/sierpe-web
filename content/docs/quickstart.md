---
title: Quickstart
weight: 20
description: Register a contract and query its complete history in five minutes.
---

This assumes a running instance — see [Installation](/docs/install/).

## 1. Register a contract

POST the contract id. Sierpe reads its on-chain spec, classifies it (SAC
by executable, wasm events from `contractspecv0`), and starts a
descending backfill while following the tip:

```bash
curl -X POST localhost:8080/v1/contracts \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"contract_id": "CBMLLYBH...", "from": "genesis",
       "kinds": ["events", "state", "movements"]}'
```

`kinds` picks what gets derived — `events`, `state`, `transfers`,
`trustlines`, `movements`. Omitting it gives `events` and `state`
(`transfers` too for a SAC). **If you care about deposits — anything
entering or leaving the contract — include `movements`**; see step 5.
`from` takes `"genesis"` or a ledger number. Adding a kind later reopens
the history walk for it, so nothing is lost by starting small.

Registration is idempotent. `DELETE` stops indexing but keeps the data;
re-registering resumes where it left off. If `HTTP_BASIC_AUTH` is set,
add `-u user:password` to every request in this guide.

## 2. Watch it work

```bash
curl localhost:8080/v1/contracts/CBMLLYBH...
```

The response includes the classification, discovered event names, backfill
progress and **derived coverage** — the exact ledger ranges Sierpe can
answer for.

## 3. Query events

Filters follow the proposed `getEvents` v2 semantics — positional topic
filters, opaque cursors:

```bash
curl "localhost:8080/v1/contracts/CBMLLYBH.../events?topic0=<base64-scval>&limit=100"
```

Every page declares `coverage` and a `scanStatus`:

| scanStatus | Meaning |
|---|---|
| `COMPLETE` | The full requested range was scanned |
| `HAS_MORE` | More results — follow the `cursor` |
| `WAITING_FOR_LEDGERS` | Part of the range isn't indexed yet |
| `OLDEST_REACHED` | You hit the oldest ledger Sierpe has |

The cursor encodes the whole query, so pagination never drifts: passing a
cursor *and* different filters is a 400, by design.

## 4. Query contract state

Current snapshot of storage entries, or the full change history of any
entry with provenance:

```bash
curl "localhost:8080/v1/contracts/CBMLLYBH.../state?key=<base64-scval>"
curl "localhost:8080/v1/contracts/CBMLLYBH.../state/history?startLedger=..."
```

## 5. See what moved in and out

With the `movements` kind, every token transfer that names your contract
as sender or recipient lands here — whoever emitted it. A payment of any
asset to your contract is emitted by the **asset's own SAC**, and you do
not need to register that SAC:

```bash
curl "localhost:8080/v1/contracts/CBMLLYBH.../movements?role=recipient"
```

Each row carries `role`, `transferType`, the exact amount in raw token
units, `tokenContractId` (the asset's real identity) and the
counterparty. Two caveats the response itself repeats: movements are
evidence of token events, **not a balance**, and amounts from different
`tokenContractId` values must never be summed — different tokens,
different scales.

The backfill derives movement history from before the contract was
registered, so yesterday's deposits appear too.
