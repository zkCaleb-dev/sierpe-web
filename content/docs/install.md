---
title: Installation
weight: 10
description: Deploy the container on Railway, Docker Compose, or any platform that runs OCI images.
---

Sierpe is distributed as a container image and as release binaries. All it
needs is an **empty Postgres database** it can own — Sierpe manages its own
schema and migrations.

## Choosing an image

| Tag | Contents |
|---|---|
| `ghcr.io/zkcaleb-dev/sierpe:v1.5.1` | Slim: static, distroless, multi-arch. Indexes from the RPC and clamps honestly at the retention wall |
| `ghcr.io/zkcaleb-dev/sierpe:v1.5.1-full` | Slim plus `stellar-core`, to heal history below RPC retention. **linux/amd64 only** — see [the archive leg](/docs/archive-leg/) |

Start with the slim image. Move to `-full` when you need history older
than the roughly seven days an RPC serves.

## Requirements

- An **empty Postgres database** reachable via `DATABASE_URL` — Sierpe
  owns the schema and runs its own migrations; do not point it at a
  database shared with another application. The bundled compose runs
  Postgres 16.
- Outbound HTTPS to public Stellar RPC endpoints.
- Storage grows with the contracts you register, not with the chain: a
  typical project (a handful of contracts) fits Railway's smallest paid
  tier.

## Docker Compose

The repository ships a ready
[docker-compose.yml](https://github.com/zkCaleb-dev/sierpe/blob/main/docker-compose.yml)
that wires Sierpe to its own Postgres 16 with a named volume:

```bash
git clone https://github.com/zkCaleb-dev/sierpe
cd sierpe
export POSTGRES_PASSWORD=$(openssl rand -hex 16)
export ADMIN_TOKEN=$(openssl rand -hex 24)
docker compose up -d
curl localhost:8080/health
```

Set `NETWORK=mainnet` and `RPC_URLS` for mainnet.

## Railway

1. Create a project with a **Postgres** service.
2. Add a service from the [GitHub repo](https://github.com/zkCaleb-dev/sierpe)
   — Railway builds the Dockerfile.
3. Set the variables: `DATABASE_URL` = `${{Postgres.DATABASE_URL}}`,
   `NETWORK`, `ADMIN_TOKEN` (and `RPC_URLS` on mainnet).
4. Expose the service; `/health` is the health check path.

## Any container platform

```bash
docker run -d -p 8080:8080 \
  -e DATABASE_URL=postgres://user:pass@host:5432/sierpe \
  -e NETWORK=testnet \
  -e ADMIN_TOKEN=$(openssl rand -hex 32) \
  ghcr.io/zkcaleb-dev/sierpe:v1.5.1
```

## Configuration

Boot configuration comes from environment variables; everything else
(contracts, their kinds) is data managed at runtime through the admin API.

| Variable | Required | Meaning |
|---|---|---|
| `DATABASE_URL` | yes | Postgres connection string; Sierpe owns this database |
| `NETWORK` | yes | `testnet` or `mainnet` |
| `ADMIN_TOKEN` | yes | Bearer token for the admin surface; minimum entropy enforced at boot |
| `RPC_URLS` | mainnet | Comma-separated failover pool, in preference order; testnet defaults to the public SDF endpoint |
| `HTTP_PORT` | no | API port, default 8080 |
| `START_LEDGER` | no | First ledger for a fresh database (default: current tip) |
| `HTTP_BASIC_AUTH` | no | `user:password`; when set, every request needs these credentials except `/health` and `/ready`. For public-domain deployments |
| `STELLAR_CORE_BINARY` | no | Path to a stellar-core binary; enables the [archive leg](/docs/archive-leg/). Pre-set in the `-full` image |
| `HISTORY_ARCHIVE_URLS` | no | History archives for the archive leg. Defaults to the SDF public archives |
| `CAPTIVE_STORAGE_PATH` | no | Disposable scratch space for captive core buckets. Defaults to the OS temp dir |

Secrets are redacted from all logs. Verify the deployment with
`GET /health` and `GET /status`; `/ready` returns 503 while catching up —
wire it to your platform's readiness probe. Then open `/` in a browser:
the [embedded UI](/docs/management-ui/) covers the whole surface.

## Exposing it safely

The default shape is **private networking**: do not give the instance a
public domain, and let your backend reach it over your platform's
internal network (on Railway, `http://sierpe.railway.internal:8080`).
Management surfaces do not face the internet — the same rule of thumb you
apply to RabbitMQ or Postgres.

If you do need a public domain, set `HTTP_BASIC_AUTH=user:password`, which
gates everything except the orchestrator probes.
