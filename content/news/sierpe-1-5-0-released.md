---
title: Sierpe 1.5.0 — movements, and coverage that names its kind
date: 2026-08-21T18:00:00Z
summary: A new resource answers what came into and went out of your contract — without registering anyone else's token — and coverage becomes a per-kind declaration.
---

This release started as a user question: *"I sent 100 USDC and 69 XLM to
my escrow and they do not show up — and I do not want to register
Circle's USDC contract just to see my own deposits."*

He was right to expect them and right to refuse the workaround. A payment
to a contract is emitted by the **asset's** SAC, so `transfers` — which
attributes rows to the emitter — can never answer "what came into my
contract". That is a different question, and 1.5.0 adds the resource that
answers it.

<!--more-->

## Movements

Register the `movements` kind and every token transfer naming your
contract as sender or recipient lands at
`GET /v1/contracts/:id/movements`, whoever emitted it. No extra RPC per
registration: attribution happens inside the ledgers the pipeline already
downloads. And because the descending backfill re-walks history with your
contract in scope, you get movement history from **before** you
registered — the thing dynamic-source indexers refuse to do.

The kind is deliberately bidirectional and deliberately not called
"deposits": a one-directional total reads exactly like a balance and is
indistinguishable from a correct one until the first outflow that never
shows up. The response says so itself, in a `note` field. Movements are
evidence of token events, not an account balance.

## Coverage grows a dimension

A backfill walk only derives the kinds the registration carried while it
ran. Until now, adding a kind to an existing registration left its
history underived while the API kept declaring `COMPLETE` — the one thing
that status must never mean. Coverage is now declared per
**(contract, kind)**: every declaration names its `kind`, says whether
the registration derives it at all, and adding a kind reopens the walk
instead of inheriting a claim it never earned.

One breaking change follows: `GET /v1/contracts/:id` returns `coverage`
as an array, one entry per registered kind.

## Reviewed against itself

The diff was adversarially reviewed before merging, and the review earned
its keep: six confirmed defects fixed before the tag, each with a
regression test verified by mutation. The worst: a page boundary landing
inside a self-transfer's two attributions silently dropped one of them —
no gap, no counter. The page cursor now carries the whole row key.
