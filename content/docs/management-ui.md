---
title: The embedded UI
weight: 25
description: A management interface baked into the binary — status, contracts, and a data explorer, with no build system and no external assets.
---

Open `/` in a browser. That is the whole setup.

Sierpe serves a management interface from the binary itself: **one
self-contained HTML page**, no build system, no external assets, no
separate service to deploy or keep in sync with the API. It covers the
same surface the REST API does.

## What it gives you

- **Live instance status** — cursor position, tip lag, the archive leg
  verdict, open gaps.
- **The contract list** — every registration with its classification,
  kinds, declared coverage and counts.
- **A data explorer** — a tab per kind: events, transfers, state and its
  history, trustlines and theirs. Filters and cursor pagination included,
  so you page through real data instead of composing curl calls.
- **Registration and unregistration** — behind an admin-token field that
  the page holds **only in memory**. Nothing is written to local storage;
  reload and the token is gone.

## Access

Reads work without credentials, matching the open-reads access model of
the API. Only mutations ask for the admin token.

If the instance faces a public domain, set `HTTP_BASIC_AUTH` — the browser
prompts natively and the UI inherits those credentials with no
configuration of its own. See [the access model](/docs/api/#access-model).

## Why it is built this way

The appliance rule applies to its own interface: if you had to build,
host or configure the UI separately, it would stop being an appliance.
Baking one static page into the binary keeps deployment a single
container, and keeps the UI incapable of drifting out of sync with the
API version it ships with.
