---
title: Sierpe 1.2.0 — history below the retention wall
date: 2026-08-20T21:00:00Z
summary: The archive leg ships. Sierpe now heals declared gaps by replaying the public history archives — but only after proving the replay byte-equivalent to your RPC.
---

Stellar RPCs retain about seven days of events. Until now, Sierpe stopped
at that wall and recorded what it could not reach as a declared gap. In
1.2.0 it goes through the wall.

<!--more-->

## The archive leg

A new captive `stellar-core` replay source serves bounded history-archive
ranges with the same unified event semantics the RPC serves
(`EMIT_CLASSIC_EVENTS`, `BACKFILL_STELLAR_ASSET_EVENTS`). Enable it, and
the gaps recorded during backfill get **healed**: walked downward in
atomic 2000-ledger chunks replayed from the public archives.

Each chunk moves the heal watermark and lowers the clamped backfill
frontier in the same transaction, so declared coverage grows exactly as
fast as healed data lands — never ahead of it.

## Why it verifies itself first

Replayed history is only worth having if it is *identical* to what the RPC
would have served. Before the first heal, the captive replay must prove
itself **byte-equivalent to your RPC** on a checkpoint-aligned range both
can serve.

Two parts of the ledger meta turn out to be unstable run to run even on
identical core builds, so both are normalized before comparing:
diagnostic events are stripped, and ledger-entry-change units are
canonically ordered within each operation. Both behaviours were proven
live.

If the replay diverges, healing is disabled and
`sierpe_archive_equivalence_failures_total` increments instead of gaps
being filled with unverified data. `/status` reports the verdict as
`archive: off | unverified | verified | equivalence_failed`.

An honest hole beats a confident guess. That principle is the whole
reason this feature took an equivalence gate rather than a flag.

## The `-full` image

The archive leg ships as an image variant,
`ghcr.io/zkcaleb-dev/sierpe:v1.2.0-full`, bundling stellar-core with
`STELLAR_CORE_BINARY` pre-set — deploy it and registrations reach below
RPC retention out of the box. It is **linux/amd64 only**, since that is
what SDF publishes stellar-core for, and wants more CPU plus a few GB of
scratch disk for bucket downloads.

The slim image stays multi-arch and distroless for archive-less
deployments. Full details in [the archive leg
documentation](/docs/archive-leg/).
