---
title: Sierpe 1.0.0 released
date: 2026-08-20
summary: The first feature-complete cut of the appliance is out — events, contract state, honest paging, and a distroless container image, verified live against testnet.
---

The first public release of Sierpe is out. Milestones M0 through M3 are
complete and verified live against the Stellar testnet.

<!--more-->

## Highlights

- **Single-writer ingestion** over a failover pool of RPC endpoints, with
  permanent hash-chain continuity verification, testnet reset detection,
  and atomic cursor-plus-data commits — exactly-once by construction.
- **Contract registration API**: POST a contract id, and Sierpe classifies
  it from its on-chain spec (SAC detection, event discovery from the wasm
  `contractspecv0` section) and starts backfilling.
- **Descending backfill** in atomic chunks with per-chunk watermarks and
  honest clamping at the RPC retention wall — the unserved range persists
  as a queryable gap.
- **Contract state**: full change history with provenance plus a current
  snapshot guarded against out-of-order replays.
- **getEvents-v2-compatible API** with positional topic filters, opaque
  full-query cursors and declared coverage on every page.
- **Operational surface**: `/health`, `/ready`, `/status`, Prometheus
  `/metrics`, plus a Grafana dashboard and a Gatus status page config.
- **Distribution**: a static distroless container image
  (`ghcr.io/zkcaleb-dev/sierpe:v1.0.0`), docker-compose deployment, and a
  deployment guide for Railway and generic container platforms.

## Getting started

```bash
docker pull ghcr.io/zkcaleb-dev/sierpe:v1.0.0
```

Follow the [installation guide](/docs/install/), then register your first
contract in the [quickstart](/docs/quickstart/).

## What's next

v1.1 focuses on structured SEP-41 token transfers and classic trustlines
of SAC assets; v1.2 targets the archive leg — replaying History Archives
for ranges below RPC retention.
