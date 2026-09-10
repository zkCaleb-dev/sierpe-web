---
title: The archive leg
weight: 35
description: How Sierpe reaches history no RPC serves anymore — and why it verifies itself before writing a single healed ledger.
---

Stellar RPCs retain roughly **seven days** of events. The slim image stops
honestly at that wall: the range it cannot serve is recorded as a
**declared gap**, and the API says so.

The `-full` image variant closes the wall. It bundles `stellar-core` and
**heals** those gaps by replaying the missing ledgers from the public
history archives — register a contract with `from: "genesis"` and its
complete history converges even where no RPC reaches.

## The equivalence gate

Replaying archives is only useful if the result is *identical* to what the
RPC would have served. Before the first heal, Sierpe proves it: the
captive replay must come out **byte-equivalent to your RPC** on a
checkpoint-aligned range both can serve.

Two parts of the ledger meta are unstable run to run even on identical
core builds, so they are normalized before comparing: diagnostic events
are stripped, and ledger-entry-change units are canonically ordered within
each operation.

If the replay diverges, healing is **disabled** —
`sierpe_archive_equivalence_failures_total` increments (alert on it) and
the gaps stay recorded. Sierpe would rather show you an honest hole than
fill it with unverified data.

`/status` reports the verdict:

```text
archive: off | unverified | verified | equivalence_failed
```

## How healing progresses

Gaps are walked downward in atomic chunks — 100,000 ledgers by default,
`HEAL_CHUNK_LEDGERS` to tune. Each chunk lowers the heal watermark on
the gap row *and* the clamped backfill frontier in the same transaction
— so **declared coverage grows exactly as fast as healed data lands**,
never ahead of it.

The chunk size matters more than it looks: every chunk is a fresh
captive core run that re-downloads the bucket set of its anchor
checkpoint, a fixed multi-minute cost regardless of chunk length. Larger
chunks amortize that download; the trade is memory (a chunk's records
are held until its single commit) and replay work lost if the process
dies mid-chunk. On a slow link healing a deep gap, raise it.

Watch `sierpe_gaps_healed_total`, `sierpe_healed_ledgers_total` and
`gaps_pending_heal` draining in `/status`.

## Sparse healing: replaying only where your contracts lived

Replaying a deep gap in full is measured in days. On a real mainnet
deployment the registered contracts had touched **0.15% of the ledgers**
in the gap being healed — the rest was empty history being replayed at
full price.

`POST /v1/admin/gaps/plan` takes the ledger ranges you want replayed and
splits every open gap into those ranges and the rest. The ranges you
asked for stay with the healer; **the rest stay open and declared** — a
deferred gap records a decision not to replay, never a claim that a
range is empty. That is what makes the plan pure scheduling: if your
plan is wrong you lose coverage, and Sierpe still never states that it
indexed history it did not read.

```bash
curl -X POST https://your-instance/v1/admin/gaps/plan \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  -d '{"replay": [{"from": 59146455, "to": 59152703}], "padding": 300}'
```

Where the plan comes from is your business: Sierpe never fetches one and
has no oracle client, no credentials and no way to spend your money. A
query against a public chain-data warehouse, another indexer, or a
hand-written list all work. Send the ranges where you believe activity
is; the server pads them and snaps them to checkpoint boundaries.

Three things worth knowing before you plan:

- **Include each contract's deployment ledger.** An instance creation is
  a contract-data change that often carries no event, so an
  events-only plan misses it. On the pilot's 1,285 contracts, 8.4% were
  deployed in a ledger holding no event at all.
- **A plan is only valid for the contracts it was computed from.**
  Registering a contract therefore reopens every deferred gap covering
  its history — the plan that deferred those ranges never looked for it.
  Register your whole batch first, then plan.
- **`open_gaps` stops reaching zero**, because deferred gaps stay open on
  purpose. Use `gaps_pending_heal` (or
  `sierpe_open_gaps - sierpe_deferred_gaps`) as the completion signal.

`HEAL_WORKERS` replays several gaps at once, which is what a plan
produces. Each worker is its own captive core, so budget roughly 10 GB
per worker against the machine's **total** memory minus everything else
it runs.

## Enabling it

Deploy the `-full` tag; `STELLAR_CORE_BINARY` is pre-set:

```bash
docker pull ghcr.io/zkcaleb-dev/sierpe:v1.10.0-full
```

| Variable | Meaning |
|---|---|
| `STELLAR_CORE_BINARY` | Path to a stellar-core binary; enables the leg. Pre-set in `-full` |
| `HISTORY_ARCHIVE_URLS` | Archives to replay from. Defaults to the SDF public archives |
| `CAPTIVE_STORAGE_PATH` | Disposable scratch space for buckets. Defaults to the OS temp dir |
| `HEAL_WORKERS` | Gaps replayed at once (default 1, max 16). Budget ~10 GB per worker against total machine memory |
| `HEAL_CHUNK_LEDGERS` | Ledgers per heal chunk (default 100000, min 64). Larger chunks amortize bucket downloads on deep heals |

Before enabling it, know that:

- The `-full` image is **linux/amd64 only** (SDF publishes stellar-core
  for amd64); it runs under emulation on ARM hosts.
- Budget more CPU and a few GB of scratch disk for bucket downloads.
- The slim image stays multi-arch and distroless for archive-less
  deployments — if you only need the last seven days plus everything
  since, you do not need this.
