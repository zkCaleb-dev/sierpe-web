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
| `POST /v1/contracts` | Register a contract; classification and backfill start automatically |
| `GET /v1/contracts` | List registered contracts |
| `GET /v1/contracts/:id` | Detail: classification, discovered events, backfill progress, coverage |
| `DELETE /v1/contracts/:id` | Stop indexing; data is kept, re-registration resumes |

## Read surface

| Method & path | Purpose |
|---|---|
| `GET /v1/contracts/:id/events` | Events with getEvents-v2-style filters and cursors |
| `GET /v1/contracts/:id/state` | Current storage snapshot, paginated by key + durability |
| `GET /v1/contracts/:id/state/history` | Change history of storage entries, with provenance |

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
