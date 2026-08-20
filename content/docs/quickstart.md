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
  -d '{"contract_id": "CBMLLYBH...", "from": "genesis"}'
```

Registration is idempotent. `DELETE` stops indexing but keeps the data;
re-registering resumes where it left off.

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
