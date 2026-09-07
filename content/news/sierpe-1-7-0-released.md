---
title: Sierpe 1.7.0 — one scan for many contracts, and a bandwidth bug the pilot caught
date: 2026-09-07T23:30:00Z
summary: Backfill walks now share their scans across contracts, and the RPC client stops silently re-paying aborted downloads. Both changes came out of the first hours of a real mainnet deployment.
---

Two releases in one day, because deploying against mainnet for real is
the fastest reviewer there is. 1.6.0 shipped this morning; by the
afternoon the first production pilot — three contracts walking eleven
months of history on a homelab machine — had exposed a cost model
problem and a silent bandwidth bug. 1.7.0 fixes both.

<!--more-->

**Backfill walks share their scans.** The walk used to be one job per
contract, which is correct and wasteful in equal measure: register N
contracts over the same range and you download N copies of every ledger.
At mainnet weight that arithmetic kills the use case Sierpe exists for —
registering a protocol's whole fleet backwards in time. Chunks now sit
on an absolute 2000-ledger grid, and contracts whose next chunk is the
same cell form a group whose range is fetched and extracted **once**,
with each contract still committing its own rows and watermark
atomically. A group is a scheduling fact, never a unit of failure: one
contract's failed commit isolates to that contract, and the trailer
catches back up through a one-entry scan cache instead of re-downloading.
A contract walking alone is a group of one and behaves exactly as before
— no mode, no flag, no config.

The first live smoke of the feature earned its keep: registration
anchors carry a few ledgers of jitter, and the initial exact-range
grouping left same-cell walks permanently one round apart, scanning
everything twice. Grouping by grid cell — scanning to the highest member
watermark so everybody lands on the cell floor together — fixed it, with
a regression test named after the incident.

**The batch shrink now remembers.** Since 1.5.1 the RPC client halves an
oversized `getLedgers` batch until it fits under the 64 MB body cap. What
it did not do was remember: every call restarted at the full batch size
and re-paid the aborted downloads, silently — no log, no metric —
multiplying a heavy range's bandwidth four to five times. The pilot
surfaced it as 3.5 GB downloaded in fifty minutes with zero chunks
committed. The client now keeps the batch size that fit and probes 25%
higher only after every eight successful batches: heavy ranges pay at
most one aborted body per eight good ones, and quiet ranges earn their
big batches back as the walk descends into lighter history.

Together the two changes take registering ~1300 contracts from
"unaffordable by a factor of thousands" to one shared walk at roughly
the true size of the data. The follow-up post with the measured mainnet
numbers is still coming — now with a better cost model to measure
against.

Upgrade: pull `ghcr.io/zkcaleb-dev/sierpe:v1.7.0` (or `v1.7.0-full` for
the archive leg). No migration, no API change; note that
`sierpe_backfill_chunks_total` and `sierpe_backfill_ledgers_scanned_total`
now count scans actually performed, once per shared chunk.
