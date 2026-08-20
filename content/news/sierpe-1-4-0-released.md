---
title: Sierpe 1.4.0 — Basic Auth for public deployments
date: 2026-08-20T22:40:00Z
summary: One environment variable gates the whole surface, so an instance on a public domain stops being an open door.
---

Sierpe's default deployment shape is private networking: no public
domain, your backend reaching the instance over your platform's internal
network. That is the RabbitMQ rule of thumb — management surfaces do not
face the internet — and it stays the recommendation.

<!--more-->

But some deployments need a public domain anyway. For those, 1.4.0 adds
one variable:

```text
HTTP_BASIC_AUTH=user:password
```

Set it, and **every** request needs those credentials — the embedded UI,
the API and `/metrics` alike. Only `/health` and `/ready` stay open, so
orchestrator probes keep working.

Browsers prompt natively and the UI inherits the credentials with no
changes of its own. Programmatic clients send the standard header from
any network (`curl -u user:password …`, or Prometheus `basic_auth` in the
scrape config). Admin mutations still require the `ADMIN_TOKEN` bearer on
top, as always.

Leaving it unset keeps the open-reads model for private-networking
deployments — nothing changes for existing instances.

The comparison is constant-time, the value is validated at boot, and it
is redacted from the config printout like every other secret.
