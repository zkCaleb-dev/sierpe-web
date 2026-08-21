---
title: Sierpe 1.5.1 — a stalled backfill and a wolf-crying warning
date: 2026-08-21T20:30:00Z
summary: An oversized RPC answer could stop a history walk permanently, with the client blaming the server's JSON for its own truncation.
---

Found by watching a real backfill on a real deployment, minutes after
1.5.0 shipped: the walk stopped 570 ledgers short of the data its
operator was waiting for, retrying the same request every 40 seconds,
logging `unexpected end of JSON input` forever.

<!--more-->

The JSON was fine. Two hundred ledgers of a busy testnet range weigh
about **151 MB**, and the client capped response bodies at 64 MB with a
reader that stops silently at its limit. The decoder received a perfectly
truncated document and blamed the server for the client's own cut. Same
request, same oversized answer, same failure — deterministic, so the
retry loop could never converge.

1.5.1 fixes the failure and the lie in one move: the overflow is now
detected and named, and the ledger batch halves itself until the answer
fits. Ledger meta size is data-dependent and unbounded — no fixed batch
size is safe over an arbitrary range. An oversized answer also stops
counting as an endpoint outage, because every endpoint in the pool would
send the same bytes.

Also in this release: registering a contract logged a WARN claiming a
backfill chunk had *failed*, when it was actually the deliberate anchor
margin waiting for ledgers that had not closed yet. A warning that fires
on a routine operation trains the operator to scroll past the one that
matters — it is now an INFO that says what it is waiting for. Fittingly,
that misleading WARN is what led straight to the real bug above.

Upgrade if you backfill anything: the stall needs no exotic conditions,
just a busy enough range.
