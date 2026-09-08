---
title: Sierpe 1.8.0 and 1.9.0 — the archive leg learns what deep healing actually costs
date: 2026-09-08T15:45:00Z
summary: Staged registrations no longer multiply the archive replay, cursors enforce their kind, and heal chunks grow 50x after the pilot showed each one was re-downloading the full bucket set.
---

The mainnet pilot moved from backfilling the RPC window into the archive
leg's territory: healing the six million ledgers below it. That
transition surfaced three problems in three days — two shipped in 1.8.0
while the window walk was still running, and the third, found the hour
the deep heal started, is 1.9.0.

<!--more-->

**Gaps are trimmed, and resolutions hand their clamps down (1.8.0).**
Registrations arrive in batches over days, and every batch clamps at its
own ever-rising retention wall, recording a gap that promises the
history below. Untrimmed, those gaps overlapped on everything under the
previous wall — so a staged rollout replayed the same deep history once
per batch, tripling the captive-core work for a three-batch plan. A new
gap is now trimmed against the open ones below it, and a gap that
resolves hands its clamped registrations to the deepest gap still open,
so a late batch's declared coverage keeps descending with the deeper
heal instead of freezing at the shared floor. An open gap is a standing
promise; the trim leaves the un-vouched set exactly as it was.

**Cursors enforce their kind (1.8.0).** Every paginated endpoint stamps
its cursors — except `/events`, the oldest codec, which never did. A
cursor minted by a different endpoint whose fields happened to unmarshal
was accepted; the pilot caught a movements cursor paging `/events` with
a straight 200. Events cursors now carry their kind, a foreign kind is
rejected with 400, and cursors minted before the stamp stay valid, so
nothing in the wild breaks.

**Heal chunks grow from 2,000 to 100,000 ledgers (1.9.0).** The healer
walks a gap in atomic chunks, and each chunk is a fresh captive core
run. What the pilot's first deep heal made unmissable is that the SDK
gives every bounded catchup an ephemeral working directory it deletes on
close — so each chunk re-downloads the full bucket set of its anchor
checkpoint. That is a fixed multi-minute cost no matter how few ledgers
the chunk covers. At 2,000 ledgers per chunk, a six-million-ledger gap
pays it about 3,100 times; on the pilot's residential link the
arithmetic came out in weeks, nearly all of it re-downloading the same
buckets. The chunk size is the lever that amortizes the cost, so it
grew 50x and became `HEAL_CHUNK_LEDGERS` for tuning — larger chunks
mean fewer downloads, traded against the memory a chunk's records
occupy until their single atomic commit and the replay work lost if the
process dies mid-chunk. The descending-watermark semantics are
untouched: declared coverage still grows exactly as fast as healed data
lands.

Upgrade: pull `ghcr.io/zkcaleb-dev/sierpe:v1.9.0` (or `v1.9.0-full` for
the archive leg). No migration; the only API change is that `/events`
now rejects foreign-kind cursors with 400, which only ever returned
wrong pages. On a slow link healing a deep gap, consider raising
`HEAL_CHUNK_LEDGERS` above the default.
