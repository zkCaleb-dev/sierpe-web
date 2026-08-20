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

Gaps are walked downward in atomic 2000-ledger chunks. Each chunk lowers
the heal watermark on the gap row *and* the clamped backfill frontier in
the same transaction — so **declared coverage grows exactly as fast as
healed data lands**, never ahead of it.

Watch `sierpe_gaps_healed_total`, `sierpe_healed_ledgers_total` and
`open_gaps` draining in `/status`.

## Enabling it

Deploy the `-full` tag; `STELLAR_CORE_BINARY` is pre-set:

```bash
docker pull ghcr.io/zkcaleb-dev/sierpe:v1.2.0-full
```

| Variable | Meaning |
|---|---|
| `STELLAR_CORE_BINARY` | Path to a stellar-core binary; enables the leg. Pre-set in `-full` |
| `HISTORY_ARCHIVE_URLS` | Archives to replay from. Defaults to the SDF public archives |
| `CAPTIVE_STORAGE_PATH` | Disposable scratch space for buckets. Defaults to the OS temp dir |

Before enabling it, know that:

- The `-full` image is **linux/amd64 only** (SDF publishes stellar-core
  for amd64); it runs under emulation on ARM hosts.
- Budget more CPU and a few GB of scratch disk for bucket downloads.
- The slim image stays multi-arch and distroless for archive-less
  deployments — if you only need the last seven days plus everything
  since, you do not need this.
