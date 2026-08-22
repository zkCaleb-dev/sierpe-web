---
title: Sierpe 1.5.2 — where it runs, and two fixes that widen the answer
date: 2026-08-21T22:30:00Z
summary: A map of forty deployment targets and twenty-five Postgres providers, plus the simple-protocol fix and a built-in healthcheck that came out of drawing it.
---

We set out to answer one question: if you hand the site to someone who
needs an indexer, where can they actually put it? The result is a new
page, [Where it runs](/docs/platforms/), covering app platforms, the
three hyperscalers, serverless (no), self-hosted from Kubernetes to a
Raspberry Pi, and twenty-five Postgres providers — each with a verdict
and the one setting that matters.

<!--more-->

Drawing the map found two things to fix.

**Transaction-mode poolers.** Supabase's pooler on port 6543, PgBouncer
before 1.21 and several managed connection pools do not support the
prepared statements the driver sends by default. The documented escape
hatch — `default_query_exec_mode=simple_protocol` in the URL — turned
out not to work either: both jsonb writers leaned on type hints only
the extended protocol provides. They now send JSON text explicitly, the
store suite runs under the simple protocol as a regression test, and a
boot that dies on a prepared-statement error names the fix in its last
log line.

**Health checks inside the container.** The image is distroless: no
shell, no curl. Docker, Swarm, Coolify, Dokploy, CapRover and NAS
container managers run their health check *inside* the container, so
every one of them marked Sierpe unhealthy forever. `sierpe healthcheck`
probes the local `/health` and is declared as the image `HEALTHCHECK`.

One thing deliberately not changed: Sierpe still ignores the `PORT`
variable. Honouring it would open Heroku web dynos and silently move
the listener of every Railway deployment that did not pin `HTTP_PORT`.
The page says so, with the reasoning.
