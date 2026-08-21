---
title: API reference
weight: 30
description: The v1 REST surface — contracts, events, state, and the honesty contract every response follows.
---

The authoritative specification is
[docs/openapi.yaml](https://github.com/zkCaleb-dev/sierpe/blob/main/docs/openapi.yaml)
in the repository. This page is the map.

## Admin surface (bearer-authenticated)

| Method & path | Purpose |
|---|---|
| `POST /v1/contracts` | Register a contract (or reconcile an existing registration); classification and backfill start automatically |
| `GET /v1/contracts/:id` | Detail: classification, discovered events, backfill progress, coverage |
| `DELETE /v1/contracts/:id` | Stop indexing; data is kept, re-registration resumes |

## Read surface

| Method & path | Purpose |
|---|---|
| `GET /v1/contracts` | List every registration with its classification and kinds |
| `GET /v1/contracts/:id/events` | Events with getEvents-v2-style filters and cursors |
| `GET /v1/contracts/:id/state` | Current storage snapshot, paginated by key + durability |
| `GET /v1/contracts/:id/state/history` | Change history of storage entries, with provenance |
| `GET /v1/contracts/:id/transfers` | Decoded token movements in chain order |
| `GET /v1/contracts/:id/trustlines` | Current trustline holders of the SAC asset |
| `GET /v1/contracts/:id/trustlines/history` | Trustline changes with before/after balances |
| `GET /v1/contracts/:id/movements` | Token transfers this contract took part in, whoever emitted them |

### Token transfers

SEP-41 movements (transfer, mint, burn, clawback) decoded into structured
rows: `from`/`to` addresses, the exact i128 amount, the SEP-0011 asset and
the CAP-67 muxed destination id. Filter by `account` (either side of the
movement, exclusive with `from`/`to`), `from`, `to`, `type`, and a ledger
range. SAC registrations derive transfers by default; custom SEP-41 tokens
opt in through `kinds`.

### Trustlines

For a registered SAC, Sierpe attributes the classic trustline changes of
the asset it wraps: live holders at `/trustlines`, and chain-order changes
with before/after balances at `/trustlines/history`. Opt in through
`kinds`. Native XLM has no trustlines, so the kind observes issued assets
only.

### Movements

Transfers answer "what did this token emit". Movements answer the other
question — **"what came into and went out of my contract"** — and they are
different questions: a payment to your contract is emitted by the asset's
own SAC, not by your contract. Register the `movements` kind and every
token transfer naming your contract as sender or recipient lands here,
without registering the asset's SAC at all.

Query parameters: `role` (`recipient` | `sender`; omit for both),
`token` (the emitting contract id — the asset's real identity, never its
SEP-0011 string), `type` (`transfer` | `mint` | `burn` | `clawback`),
`startLedger`, `endLedger`, `limit` (1–1000, default 100) and `cursor`.
"Deposits" in the everyday sense are `role=recipient` — that includes
mints to the contract, since a mint is value arriving too.

Each row: `id` (shared by the two rows of a self-transfer — key on `id` + `role`), `role`, `transferType`, `tokenContractId`,
`counterparty` (absent on mints and burns), `amount` (exact i128 in raw
token units, as a string), `ledger`, `ledgerClosedAt`. The page carries
the usual `cursor`, `scanStatus`, `coverage` and a `note`.

Because ingestion downloads whole ledgers, the descending backfill derives
movement history from **before** the contract was registered — the thing
dynamic-source indexers cannot do.

One honest caveat, stated by the API itself in a `note` field: movements
are **not a balance**. Value can also arrive without any SEP-41 transfer
event, and amounts are raw base units of different tokens — never sum
across `tokenContractId`.

## Operational surface

| Path | Purpose |
|---|---|
| `/health` | Liveness |
| `/ready` | Readiness — 503 while catching up |
| `/status` | Cursor position, tip distance, per-contract summary |
| `/metrics` | Prometheus metrics ([documented](https://github.com/zkCaleb-dev/sierpe/blob/main/docs/METRICS.md)) |

## The honesty contract

Every paginated response carries:

- **`coverage`** — the ledger ranges this instance can actually answer
  for, derived from backfill progress and the live cursor. Since 1.5.0
  coverage is declared **per (contract, kind)**: a walk only vouches for
  the kinds it actually derived, and a kind added later reopens the walk
  instead of silently claiming history it never looked at.
- **`scanStatus`** — `COMPLETE`, `HAS_MORE`, `WAITING_FOR_LEDGERS` or
  `OLDEST_REACHED`.
- **`cursor`** — opaque, encodes the full query. Cursors are bound to
  their endpoint and cannot drift across filters.

Event ids follow the `getEvents` format: `{toid}-{event_index}`,
zero-padded, stable across replays.

## Access model

Reads are open; admin mutations require the `ADMIN_TOKEN` bearer. That
suits the default deployment shape — private networking, where nothing
outside your platform's internal network reaches the instance.

If you do expose a public domain, set `HTTP_BASIC_AUTH=user:password` and
**every** request needs those credentials — the UI, the API and
`/metrics` — leaving only `/health` and `/ready` open for orchestrator
probes. Browsers prompt natively; clients send the standard header
(`curl -u user:password …`). Admin mutations still need the bearer on top.
