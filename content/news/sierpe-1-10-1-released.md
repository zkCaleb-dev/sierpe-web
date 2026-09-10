---
title: Sierpe 1.10.1 — two ways a gap could be recorded wrong
date: 2026-09-10T02:42:00Z
summary: A patch for the gap bookkeeping underneath sparse healing, including one coverage-honesty bug that is older than the feature that exposed it.
---

Sparse healing shipped yesterday. Answering an operator's question about
it — *can I apply a plan before my next batch of registrations clamps?* —
turned up two bugs in how a gap gets recorded. The answer was no, and
now it is yes.

<!--more-->

**A new gap could overlap the ones already open.** Recording an unserved
range subtracts whatever open gaps already promise parts of it, so a
second batch of registrations does not make the healer replay millions
of ledgers a second time. That subtraction walked a floor upward and
stopped at the first ledger no open gap covered — correct as long as the
gaps below were one contiguous run.

A heal plan is precisely what stops guaranteeing that. It splits one gap
into clusters and deserts, and every cluster the healer finishes leaves
a hole in the open coverage. The next batch to clamp then recorded a
single gap from that hole up to its own wall, overlapping every gap
still open above it — and sent the healer to walk linearly through
exactly the deserts the plan had just excluded. Recording now subtracts
the whole set and records one gap per real hole, which collapses to the
old single gap when the coverage below is contiguous.

**And a healed range was never promised again.** This one is older than
sparse healing and worse. Gap ids are deterministic, so re-recording a
range whose gap had already been healed collided with the resolved row
and was silently dropped.

That matters because a gap is healed against the registry *as it stood
at the time*. A contract registered afterwards has no rows from that
range — and was never promised it again, so its declared coverage
claimed history nobody had derived for it. Sierpe's whole proposition is
that it does not do that. The insert now reopens the resolved gap, and
the range gets replayed with the new contract in the registry.

Reopening is conservative in the other direction: ranges that were
healed show as gaps again until they are re-healed. Under-claiming is
the side of this trade we are willing to be on.

Upgrade: pull `ghcr.io/zkcaleb-dev/sierpe:v1.10.1` (or `v1.10.1-full`).
No migration, no API change. If you have registered contracts in batches
over time and any of your gaps have healed, this release is the one that
stops your coverage from over-claiming.
