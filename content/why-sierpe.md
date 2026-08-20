---
title: Why Sierpe
---

Every team building on Stellar eventually hits the same wall: the RPC
retains about **seven days** of events. If your frontend depends on
contract events, you either build and babysit a custom indexer, pay for a
hosted service, or lose your own history.

The existing landscape offers three shapes, and all of them ask something
from you:

| Shape | Examples | What it asks of you |
|---|---|---|
| Hosted service | Mercury, stellarindexer.com | Your data lives in someone else's infra, usually behind a subscription |
| Composable toolkit | CDP building blocks, flowctl/nebu | You assemble and operate a pipeline |
| Framework | SubQuery | You fork a template and write indexing code |

Sierpe takes the fourth slot, the one nobody occupied: the **self-hosted
appliance**. Like Postgres or Prometheus, it is a server you deploy, not a
codebase you adopt. If you have to write code to use it, that's a bug.

## What makes it different

**History past the retention wall.** Registering a contract *today* gets
you its complete past, not just its future. Sierpe backfills as far as the
RPC serves, records what it cannot reach as a declared gap, and — with the
[archive leg](/docs/archive-leg/) enabled — heals those gaps by replaying
the public history archives, but only after the replay proves itself
byte-equivalent to your RPC. History below retention, or an honest gap.
Never an unverified guess.

**Honesty as an API contract.** Distributed ingestion loses data in
silence; Sierpe refuses to. Every paginated response declares its
`coverage` and a `scanStatus` (`COMPLETE`, `HAS_MORE`,
`WAITING_FOR_LEDGERS`, `OLDEST_REACHED`). Gaps are persisted, queryable
and alertable — never papered over.

**More than events.** Storage entry change history with provenance plus a
current snapshot, decoded SEP-41 token movements, and the classic
trustlines of the asset a SAC wraps — data most event indexers ignore.

**Forward-compatible by design.** The events API follows the semantics of
the proposed `getEvents` v2 RPC endpoint (positional topic filters, opaque
cursors, scan status), so an integration built against Sierpe speaks
tomorrow's standard.

**Integrity paranoia.** A single-writer loop verifies ledger hash-chain
continuity permanently — including across restarts — commits cursor and
data in the same transaction (exactly-once by construction), detects
testnet resets, and would rather stop loudly than write a lie.

## What Sierpe is not

- **Not a hosted service** — you run it. That's the point.
- **Not an analytics platform** — no aggregations or dashboards over your data.
- **Not a chain-wide indexer** — it indexes the contracts you register.
- **Not a framework** — there is nothing to fork and no SDK to learn.

## The cost story

The volume Sierpe stores is proportional to the contracts you register,
not to the chain. A typical project runs the container plus a small
Postgres for **under $10/month** on Railway or any VPS.
