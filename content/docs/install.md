---
title: Installation
weight: 10
description: Deploy the container on Railway, Docker Compose, or any platform that runs OCI images.
---

Sierpe is distributed as a static, distroless container image and as
release binaries. All it needs is an **empty Postgres database** it can
own — Sierpe manages its own schema and migrations.

## Requirements

- Postgres 14+ reachable via `DATABASE_URL` (Sierpe owns the schema; do
  not point it at a database shared with another application).
- Outbound HTTPS to public Stellar RPC endpoints.
- Roughly 256 MB of memory to start; storage grows with the contracts you
  register, not with the chain.

## Docker Compose

The repository ships a ready
[docker-compose.yml](https://github.com/zkCaleb-dev/sierpe/blob/main/docker-compose.yml):

```bash
git clone https://github.com/zkCaleb-dev/sierpe
cd sierpe
NETWORK=testnet ADMIN_TOKEN=$(openssl rand -hex 32) docker compose up -d
```

## Railway

Create a project with a Postgres service and a service from the public
image `ghcr.io/zkcaleb-dev/sierpe:v1.0.0`, then set:

```text
DATABASE_URL = ${{Postgres.DATABASE_URL}}
NETWORK      = mainnet | testnet
ADMIN_TOKEN  = <long random secret>
```

The container listens on port 8080.

## Any container platform

```bash
docker run -d -p 8080:8080 \
  -e DATABASE_URL=postgres://user:pass@host:5432/sierpe \
  -e NETWORK=testnet \
  -e ADMIN_TOKEN=$(openssl rand -hex 32) \
  ghcr.io/zkcaleb-dev/sierpe:v1.0.0
```

## Configuration

Boot configuration comes from environment variables; everything else
(contracts, their kinds) is data managed at runtime through the admin API.

| Variable | Required | Meaning |
|---|---|---|
| `DATABASE_URL` | yes | Postgres connection string; Sierpe owns this database |
| `NETWORK` | yes | `mainnet` or `testnet` |
| `ADMIN_TOKEN` | yes | Bearer token for the admin surface; entropy is enforced at boot |
| `RPC_URLS` | no | Comma-separated failover pool; sensible public defaults per network |

Secrets are redacted from all logs. Verify the deployment with
`GET /health` and `GET /status`; `/ready` returns 503 while catching up —
wire it to your platform's readiness probe.
