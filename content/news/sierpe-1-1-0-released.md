---
title: Sierpe 1.1.0 — token transfers and trustlines
date: 2026-08-20T19:00:00Z
summary: "Two new data kinds land: decoded SEP-41 token movements, and the classic trustlines of the asset a SAC wraps."
---

Sierpe 1.1.0 adds two data kinds beyond events and contract state, both
under the same honesty contract as the rest of the API.

<!--more-->

## Token transfers

SEP-41 movement events — transfer, mint, burn, clawback — now decode into
structured rows: `from`/`to` addresses, the exact i128 amount, the
SEP-0011 asset, and the CAP-67 muxed destination id. They are written in
the same atomic commit as events and state, and served at
`GET /v1/contracts/:id/transfers` with `account`/`from`/`to`/`type`
filters.

SAC registrations derive transfers by default; custom SEP-41 tokens opt in
through `kinds`. An event that names a movement but fails to decode is
counted in `sierpe_suppressed_transfers_total` — alertable — while its raw
event row still lands. The decoder never quietly swallows what it does not
understand.

## Classic trustlines

For a registered SAC, the trustline changes of the asset it wraps are now
attributed to that contract — with the contract id derived locally, so
this costs **zero extra RPC calls**. Stored as full history plus a
convergence-safe holder snapshot with tombstones, and served at
`GET /v1/contracts/:id/trustlines` (live holders) and
`/trustlines/history` (chain-order changes with before and after
balances).

The kind is opt-in through `kinds`, and observes issued assets only —
native XLM has no trustlines.
