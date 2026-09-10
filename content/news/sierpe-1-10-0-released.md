---
title: Sierpe 1.10.0 — healing only where your contracts actually lived
date: 2026-09-10T02:20:00Z
summary: A deep heal was replaying two weeks of empty history to recover data that lived in 0.15% of it. You can now tell the archive leg which ranges to replay, and the ranges it skips stay declared rather than quietly claimed.
---

The mainnet pilot finished walking the RPC window and started on what
lies below it: six million ledgers with no live source, healed by
replaying them through a captive stellar-core. The first measurements
made the problem plain. Replay runs at 4.72 ledgers per second on that
machine, which puts the remaining range at about two weeks — to recover
data that lives in **0.15% of those ledgers**. The rest was empty
history being replayed at full price.

<!--more-->

**You can now say which ranges to replay.** `POST /v1/admin/gaps/plan`
takes the ledger ranges where you believe your contracts were active and
splits every open gap into those ranges and the rest. The ranges you
asked for stay with the healer. The rest become *deferred* gaps: still
open, still declared, still listed as missing.

That distinction is the whole design, and it is why the endpoint takes a
plan instead of a certificate. A deferred gap records **a decision not
to replay** — never a claim that a range is empty. So the plan is pure
scheduling: if it is wrong, the instance covers less than it could and
says so, and it still never states that it indexed history it did not
read. The tempting version of this feature would have let the hint
certify the skipped ranges and shaved days off the walk. That trade
buys speed with an asterisk on the one promise this project exists to
keep, and it was rejected on purpose.

The mechanism is the gap itself. Each piece of a split gap carries its
own heal watermark, so **no watermark ever descends through a range
nobody read** — which was the obstacle all along, and the reason
skipping could not simply be a flag on the healer.

**Sierpe never fetches the plan.** No oracle client, no credentials, no
way to spend your money. Where the ranges come from is your business: a
query against a public chain-data warehouse, another indexer, a
hand-written list. Send the ranges; the server pads them and snaps them
to checkpoint boundaries.

Two things the pilot learned the hard way, both now in
[the docs](/docs/archive-leg/). **Include each contract's deployment
ledger**: creating a contract instance is a storage change that often
carries no event at all — 8.4% of the pilot's 1,285 contracts were
deployed in a ledger holding no event — so an events-only plan misses
them, and padding will not save a deployment that sits months from the
nearest cluster. And **a plan is only valid for the contracts it was
computed from**, so registering a contract reopens every deferred gap
covering its history. Register the batch first, then plan.

**One thing to change if you watch this instance.** Deferred gaps stay
open by design, so `open_gaps` no longer reaches zero once you apply a
plan. Anything checking `open_gaps == 0` for "history is complete" is
now waiting for something that cannot happen, and it waits silently.
`/status` serves `gaps_pending_heal` for exactly this, and the
Prometheus equivalent is `sierpe_open_gaps - sierpe_deferred_gaps`. If
you never apply a plan, nothing changes: your deferred count is zero and
the two numbers agree.

Also in this release: `HEAL_WORKERS` replays several gaps at once, which
is what a plan produces — budget around 10 GB per worker against the
machine's total memory, since each one is its own captive core. And two
fixes from auditing what we actually publish. The images now carry
`org.opencontainers.image.source`, without which the registry never
linked the package to its repository. And the `-full` image pins an
exact stellar-core build instead of the floating `28` tag — that tag
moved from 28.0.0 to 28.0.1 mid-pilot, which meant the same Dockerfile
produced a different replay engine depending on the day, in the one
image whose entire job is to reproduce history byte for byte. The
equivalence gate would have caught a divergence, but a gate that catches
drift is not the same thing as an engine that cannot drift.

Upgrade: pull `ghcr.io/zkcaleb-dev/sierpe:v1.10.0` (or `v1.10.0-full`
for the archive leg). No migration is needed beyond the automatic one;
existing gaps become replay gaps, which is exactly the behaviour they
had. Note that `latest` had drifted to v1.5.2 for four releases and now
points at v1.10.0 — if you pull untagged, pull again.
