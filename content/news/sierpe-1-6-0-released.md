---
title: Sierpe 1.6.0 — the raw event on every movement, and registering the archived
date: 2026-09-07T22:00:00Z
summary: Movements now carry the ContractEvent XDR they were decoded from, and contracts whose instance expired can finally be registered. Both born from one production use case.
---

Two changes, both forced by the same real-world job: collecting eleven
months of mainnet history for a production escrow protocol (the case
described in the upcoming use-case post).

<!--more-->

**Movements carry the raw event.** A movement is usually emitted by a
token contract nobody registered — a payment into your contract comes
from the asset's own SAC — so there was no events row to join to and the
original `ContractEvent` bytes were unrecoverable from the database.
That is fine for reading, and not fine for building: a consumer that
re-emits movements into its own pipeline needs the event itself, not our
decode. Every movement row now stores and serves `rawXdr`, captured at
the only moment in the pipeline where those bytes exist. Rows ingested
before the migration have no stored event to backfill from and simply
omit the field — absence stays honest.

**Archived contracts register.** A Soroban contract whose instance TTL
expired vanishes from the RPC, and registration answered 404 — even
though the contract's entire history sits in the public History Archives
and the [archive leg](/docs/archive-leg/) reconstructs it regardless.
Now `POST /v1/contracts` with explicit `kinds` accepts it: classification
comes back `unknown`, a new `warnings` array on the response says exactly
what was accepted unverified, and the backfill anchors as usual. Without
explicit kinds the 404 stands — the kinds default is derived from the
on-chain classification, and with no instance there is nothing honest to
default to. Re-registering after a restore re-classifies.

One honest caveat, because the RPC cannot tell an archived contract from
one that never existed: a typo that survives the strkey checksum
registers too. The failure mode is deliberate — an empty history under
honest coverage, one `DELETE` to clean up.

Also in this release: the Go toolchain moved to 1.25.13 and the
reachable dependency findings govulncheck reported against 1.25.0
binaries are cleared.

Upgrade: pull `ghcr.io/zkcaleb-dev/sierpe:v1.6.0` (or `v1.6.0-full` for
the archive leg); migration 0011 applies itself at boot.
