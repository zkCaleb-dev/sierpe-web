---
title: Sierpe 1.3.0 — the appliance grows a face
date: 2026-08-20T22:00:00Z
summary: A management UI baked into the binary — status, contracts and a full data explorer — plus the contract listing endpoint it needed.
---

An appliance you can only talk to with curl is only half an appliance.
1.3.0 adds a management interface — and keeps it as boring to deploy as
the rest of the server.

<!--more-->

## One page, inside the binary

Open `/` in a browser. There is no build system, no external assets, no
second service: the UI is a single self-contained HTML page baked into
the binary, so it ships and versions with the API it talks to and cannot
drift out of sync with it.

It covers the whole surface: live instance status, the contract list with
classification, coverage and counts, and a data explorer with a tab per
kind — events, transfers, state and its history, trustlines and theirs —
with filters and cursor pagination. Registration and unregistration sit
behind an admin-token field the page holds **only in memory**.

Reads work without credentials, matching the open-reads access model of
the API.

## `GET /v1/contracts`

The UI needed a way to enumerate what an instance watches without knowing
contract ids upfront, and so does anyone integrating with Sierpe. The new
listing endpoint returns every registration with its classification and
kinds.

Full details in [the embedded UI documentation](/docs/management-ui/).
