---
title: Sierpe 1.4.1 and 1.4.2 — fixes from the first real deployment
date: 2026-08-20T23:30:00Z
summary: Two bugs only a real user on a real deployment could find, fixed the same evening they were reported.
---

The first Basic-Auth deployment on a public domain found two bugs within
an hour of going live. Both are the kind no test suite catches, because
both live in the seams between the app and the world around it.

<!--more-->

**1.4.1**: with `HTTP_BASIC_AUTH` set, admin mutations were impossible.
The Basic credentials and the admin bearer token share the one
`Authorization` header, so no request could satisfy both layers at once.
The gate now also accepts the admin token as a valid credential — it is
already a stronger secret than the password protecting the same surface.

**1.4.2**: opening the UI through a URL with embedded credentials
(`https://user:pass@host/`) rendered a blank page. Relative fetch URLs
inherit the document's credentials and the Fetch spec rejects them, so
every API call from the UI failed before leaving the browser. API paths
now resolve against `location.origin`, which never carries credentials.

Both releases ship the same day their bugs were reported. If you run
1.4.0 with Basic Auth enabled, upgrade.
