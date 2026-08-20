---
title: Observability
weight: 40
description: Prometheus metrics, the Grafana dashboard, and the alerts that matter.
---

Sierpe is built on the principle that **silent data loss is the worst
failure mode**. Everything it cannot index is counted, exposed and
alertable.

## Metrics

`/metrics` exposes Prometheus metrics from a private registry. The full
catalogue — and which ones deserve alerts — is in
[docs/METRICS.md](https://github.com/zkCaleb-dev/sierpe/blob/main/docs/METRICS.md).
The headline signals:

- **Ingestion lag** — distance from the network tip.
- **Open gaps** — ledger ranges Sierpe knows it is missing. A non-zero
  value is the alert.
- **Suppression counters** — anything skipped during extraction is
  counted, never silently dropped.

## Grafana

A ready-made dashboard ships in the repository at
[deploy/grafana/sierpe-dashboard.json](https://github.com/zkCaleb-dev/sierpe/blob/main/deploy/grafana/sierpe-dashboard.json)
— eight panels covering the signals above.

## Status page

A [Gatus](https://github.com/TwiN/gatus) configuration is provided at
`deploy/gatus/config.yaml`, including a check that turns the status page
red when open gaps exist — your users see data honesty, not just uptime.

## Integrity guarantees worth knowing

- The ingestion loop verifies `PreviousLedgerHash` continuity on **every**
  ledger, including the first one after a restart.
- Cursor and data commit in the same transaction — a crash can never
  leave them disagreeing.
- On testnet resets, divergence is detected and the process stops loudly
  with zero writes rather than mixing two histories.
