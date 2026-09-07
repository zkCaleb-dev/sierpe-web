---
title: Backfilling a year of mainnet history for a production protocol
date: 2026-09-07T00:00:00Z
summary: 1285 escrow contracts, eleven months of history nobody indexed, and no archival RPC budget. What Sierpe's archive leg was built for, about to run for real.
draft: true
---

Sierpe was designed around one promise: you can redefine what concerns
you *backwards in time*. Register a contract today and get its complete
history, even the part every RPC forgot months ago. Until now that
promise had only been exercised on testnet smokes. Here is the first
production case, before we run it.

<!--more-->

**The situation.** [Trustless Work](https://www.trustlesswork.com/), an
escrow platform on Stellar, rebuilt its read-model this year. The new
pipeline started from a recent ledger — correct for everything new, blind
to everything old. The old side is not small: **1285 escrow contracts**
deployed since **September 2025**, holding real USDC flows. The contracts
are still live and their new events index fine; the eleven months behind
the cursor do not exist anywhere queryable.

**The options that do not work.** Replaying from an RPC needs a node that
serves eleven months of mainnet — archival RPC providers charge real
money for that, and free endpoints keep days, not months. Rebuilding
from the platform's own application database was worse: deploy-time
metadata with an API clock, no event trail, no deposits. A list of
contract IDs is the only thing it could be trusted for.

**What Sierpe does instead.** The [archive leg](/docs/archive-leg/)
reconstructs below-retention history from the public History Archives —
the permanent record every validator publishes — through a captive
stellar-core replay. No RPC in that path, nothing to pay for. Each healed
range is gated by a byte-equivalence proof where archive and RPC overlap,
so the reconstructed history is held to the same standard as the live
feed. Coverage and gaps stay declared data the whole way: the API tells
you exactly which ranges it can vouch for while the walk is still
running.

The run itself is deliberately boring: a desktop-class machine in a
homelab, one `POST /v1/contracts` per escrow with `from: "genesis"`, and
roughly 5.2 million ledgers of replay measured in days, not dollars —
the marginal cost is the electricity. Progress is watched the same way
anything else in Sierpe is watched: `sierpe_backfill_pending` draining to
zero, `sierpe_open_gaps` at zero, and every `sierpe_suppressed_*` counter
staying at zero or raising an alarm.

**What the case fed back into Sierpe.** The consumer re-emits history
into its own event pipeline, and for that the decoded movement is not
enough — it needs the original `ContractEvent` bytes. But a movement is
usually emitted by a token contract nobody registered (USDC's SAC here),
so there is no events row to join to and the raw event was unrecoverable
from the database. [Sierpe 1.6.0](/news/sierpe-1-6-0-released/) fixes
that: movements store and serve the raw event XDR they were decoded from,
captured at the only moment it exists in the pipeline. A small column,
and exactly the difference between an indexer you can read and an indexer
you can build on. The same case surfaced the other 1.6.0 change —
registering contracts whose instance already expired.

A follow-up post will report the real numbers — replay throughput,
download volume, wall-clock time, and whatever the equivalence proof has
to say about eleven months of mainnet.
