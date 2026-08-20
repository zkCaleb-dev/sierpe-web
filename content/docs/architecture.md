---
title: Architecture
weight: 50
description: The pipeline, the data model, and the design decisions behind the appliance.
---

The full design document lives at
[docs/DESIGN.md](https://github.com/zkCaleb-dev/sierpe/blob/main/docs/DESIGN.md);
the study behind it — 29 principles distilled from production indexers,
each with its source — at
[docs/KNOWLEDGE.md](https://github.com/zkCaleb-dev/sierpe/blob/main/docs/KNOWLEDGE.md).

## The pipeline

```text
source  →  ingest  →  process  →  store  →  serve
 (RPC       (single-   (classify,   (Postgres,  (REST API,
  pool)      writer     extract      atomic      honest
             loop)      events &     commits)    paging)
                        state)
```

- **Source** is a seam: the RPC pool today; captive-core / History
  Archives (v1.2) plug in behind the same interface without touching the
  rest.
- **Ingest** is a single-writer loop. One writer means hash-chain
  continuity can be *verified*, not assumed.
- **Store** owns its Postgres: embedded migrations under an advisory
  lock, cursor and data in one transaction.
- **Serve** never invents data: what the store hasn't got, the API
  declares as a gap.

## Backfill

Registration triggers a **descending** backfill: from the tip backwards
in atomic 2000-ledger chunks, each with its own watermark. Newest data
arrives first — usually what you want — and progress survives restarts
exactly. At the RPC retention wall, the unserved range is persisted as a
gap *before* the clamp commits, so nothing is ever silently missing.

## Design rules the code enforces

- If the user has to touch code, it's a design bug.
- Sierpe owns its database; consumers use the API, never the tables.
- Exactly-once by construction (atomic cursor+data), not by deduplication.
- Systematic distrust: failed transactions skipped and counted, spec
  parse failures degrade to `opaque` classification instead of erroring.
- No CGO — a single static binary, trivially containerized.
