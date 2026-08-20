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
  for, derived from backfill progress and the live cursor.
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
