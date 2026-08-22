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

This is the complete file — save it as `docker-compose.yml`, nothing else
is needed (it matches the one
[in the repository](https://github.com/zkCaleb-dev/sierpe/blob/main/docker-compose.yml)):

```yaml
services:
  sierpe:
    image: ghcr.io/zkcaleb-dev/sierpe:v1.5.1
    # image: ghcr.io/zkcaleb-dev/sierpe:v1.5.1-full   # archive leg: heals history below RPC retention (amd64)
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://sierpe:${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD}@db:5432/sierpe?sslmode=disable
      NETWORK: ${NETWORK:-testnet}
      ADMIN_TOKEN: ${ADMIN_TOKEN:?set ADMIN_TOKEN (min 16 chars)}
      # RPC_URLS: https://your-rpc-1,https://your-rpc-2   # required on mainnet
    ports:
      - "8080:8080"

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: sierpe
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:?set POSTGRES_PASSWORD}
      POSTGRES_DB: sierpe
    volumes:
      - sierpe-pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sierpe -d sierpe"]
      interval: 5s
      timeout: 3s
      retries: 12

volumes:
  sierpe-pgdata:
```

```bash
export POSTGRES_PASSWORD=$(openssl rand -hex 16)
export ADMIN_TOKEN=$(openssl rand -hex 24)
docker compose up -d
curl localhost:8080/health
```

Set `NETWORK=mainnet` and `RPC_URLS` for mainnet.

## Railway

The image route — no GitHub account or build step needed:

1. **New Project → Deploy PostgreSQL.** A fresh Railway Postgres is
   empty, which is what Sierpe needs; do not reuse a database another app
   owns. Note the service name on the card (default `Postgres`).
2. **+ New → Docker Image**, type `ghcr.io/zkcaleb-dev/sierpe:v1.5.1`.
   The first deploy fails until the variables exist — expected.
3. **Variables** tab of the new service:
   - `DATABASE_URL` = `${{Postgres.DATABASE_URL}}` — the reference works
     as-is; no `sslmode` parameter is needed on Railway's internal network.
   - `NETWORK` = `testnet` (or `mainnet`, plus `RPC_URLS`)
   - `ADMIN_TOKEN` = a random string, 16+ characters (`openssl rand -hex 24`)
   - `HTTP_BASIC_AUTH` = `user:password` — set it if you will give the
     service a public domain (next step). Skip it if only other services
     in the same project will reach it over private networking.
4. **Settings → Deploy → Healthcheck Path**: `/health`. Not `/ready`,
   which returns 503 while catching up and would fail the deploy.
5. **Settings → Networking → Generate Domain**, and when asked for the
   port, answer **`8080`**. Sierpe listens on its own `HTTP_PORT`
   (default 8080) and ignores Railway's injected `PORT`. Private
   networking stays on by default: other services in the project reach
   it at `http://sierpe.railway.internal:8080`.
6. Deploy, then open `https://YOUR-APP.up.railway.app/status`. A fresh
   database starts at the current tip, so `ready` flips to `true` within
   seconds — history arrives per contract, through the backfill.

No volume is needed on the Sierpe service: all state lives in Postgres.
The `-full` image is the one exception — it wants a few GB of scratch
disk for captive core; see [the archive leg](/docs/archive-leg/).

## AWS (ECS Fargate + RDS)

The shape a small team would run: one Fargate task, a managed Postgres,
nothing public.

- **RDS for PostgreSQL 16**, private subnets, not publicly accessible.
  Create an empty database and a role that **owns** it
  (`CREATE ROLE sierpe LOGIN PASSWORD '…'; CREATE DATABASE sierpe OWNER sierpe;`).
  Owner is enough — Sierpe runs its own migrations; it never needs
  superuser.
- **`DATABASE_URL` must end in `?sslmode=require`.** RDS enforces TLS by
  default on PostgreSQL 15+, and the compose file's `sslmode=disable` is
  for the bundled local Postgres only. Use `require`, not `verify-full`:
  the image carries the public root CAs (it talks HTTPS to the RPC) but
  not Amazon's RDS CA, so full verification would fail. This applies to
  every managed Postgres with a private CA (RDS, Supabase, Neon…).
- **Task definition**: image `ghcr.io/zkcaleb-dev/sierpe:v1.5.1`,
  container port `8080`, `NETWORK` as plain environment, `DATABASE_URL`
  and `ADMIN_TOKEN` as ECS `secrets` from Secrets Manager. 0.5 vCPU /
  1 GiB is a sound start (see sizing below). No volume, no EFS. Use the
  `awslogs` driver; logs are structured JSON with secrets redacted.
- **Health**: the image is distroless (no shell, no curl), so use the
  load balancer's target-group check, not a container `CMD` check.
  Path `/health`, success code 200. **Never `/ready`** here — it returns
  503 while catching up, and an ECS health check on it would kill a
  healthy task mid-backfill.
- **Networking**: private subnets with a NAT gateway — the task needs
  outbound HTTPS for the Stellar RPC and for the `ghcr.io` image pull.
  An **internal** ALB gives your backend a stable name inside the VPC.
  Only if you truly need a public endpoint: internet-facing ALB + ACM
  certificate and set `HTTP_BASIC_AUTH`.
- **Exactly one task**: `desiredCount: 1`, `minimumHealthyPercent: 0`,
  `maximumPercent: 100`, so a deploy never runs two copies at once (why:
  next section).

## Any container platform

```bash
docker run -d -p 8080:8080 \
  -e DATABASE_URL=postgres://user:pass@host:5432/sierpe \
  -e NETWORK=testnet \
  -e ADMIN_TOKEN=$(openssl rand -hex 32) \
  ghcr.io/zkcaleb-dev/sierpe:v1.5.1
```

## Operating it on any cloud — the facts that matter

Answers to what an operator (or their assistant) has to decide, stated
from the code rather than guessed.

- **Run exactly one instance per database.** Ingestion is a single
  writer with no leader election. A second instance does not corrupt
  anything — its first commit trips the hash-continuity guard and the
  process exits loudly — but you get a crash loop, not high availability.
  Restarts are safe at any moment: the cursor and the data commit in one
  transaction, so a killed task resumes exactly where it stopped.
- **Shutdown**: `SIGTERM` is handled; the HTTP server drains for up to
  5 seconds and the loop stops between commits. First boot runs the
  embedded migrations in well under a minute.
- **Database connections**: pgx defaults — at most `max(4, CPUs)`
  pooled connections. A `db.t4g.micro` or Railway's Postgres is fine.
- **Memory**: the slim image idles at tens of MB. The ceiling is the
  backfill, which buffers RPC responses of up to 64 MB each and shrinks
  its batch when the network is busier than that; plan 512 MB, and
  1 GiB if you register many contracts at once.
- **TLS to Postgres**: `sslmode=require` for any managed provider with a
  private CA; `disable` only for a Postgres on the same private network
  that you control; `verify-full` only if you can mount the provider's
  CA bundle (the image is distroless — building a derived image is the
  way).
- **Testnet resets**: when the network is reset (the tip jumps back by
  millions of ledgers), the loop detects it and **stops with zero
  writes** rather than mixing two chains. Recovery is deliberate and
  manual: drop and recreate the empty database, redeploy, re-register
  your contracts. Everything Sierpe holds is re-derivable from the chain.
- **Defaults you do not need to set**: on testnet the RPC pool is
  `https://soroban-testnet.stellar.org`; history archives default to the
  SDF public archives on **both** networks. Mainnet has no free public
  RPC, so `RPC_URLS` is required there.
- **Behind a proxy**: TLS termination in front is fine. Path prefixes
  are not — the embedded UI and the API assume they live at `/`.

## Configuration

Boot configuration comes from environment variables; everything else
(contracts, their kinds) is data managed at runtime through the admin API.

| Variable | Required | Meaning |
|---|---|---|
| `DATABASE_URL` | yes | Postgres connection string; Sierpe owns this database |
| `NETWORK` | yes | `testnet` or `mainnet` |
| `ADMIN_TOKEN` | yes | Bearer token for the admin surface. At least 16 characters with 6 distinct ones, enforced at boot (`openssl rand -hex 24` is fine) |
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

## First-run troubleshooting

Every one of these was hit by a real first deployment; the fixes are
exact.

| Symptom | Cause | Fix |
|---|---|---|
| Boot error naming `DATABASE_URL`, `NETWORK` or `ADMIN_TOKEN` | Variables are unprefixed — `SIERPE_DATABASE_URL` is not read | Use the exact names from the table above |
| `/ready` returns 503, `/health` returns 200 | Normal while catching up to the tip | Wait; watch `/status` — `ready` flips when the cursor reaches the tip |
| `401` on `POST /v1/contracts` | Missing bearer, or `HTTP_BASIC_AUTH` is set and the client sent only one credential | Send `Authorization: Bearer $ADMIN_TOKEN`; with Basic Auth enabled the admin token is also accepted as the Basic password |
| `404 contract does not exist` on registration | Contract not found on the configured network — wrong `NETWORK`, a typo, or an asset whose SAC was never deployed | Check the id on that network; deploy the SAC first for classic assets |
| A fresh database starts at the tip, not in the past | By design — history arrives via each contract's backfill, not by replaying the whole chain | Register contracts with `"from"`; use `START_LEDGER` only when you need the live cursor itself to begin earlier |
| Right after registering, coverage shows `indexedFromLedger` above `indexedToLedger` | An intentionally empty window: the backfill anchors slightly past the live cursor | It closes on its own within a minute; not an error |

## Exposing it safely

The default shape is **private networking**: do not give the instance a
public domain, and let your backend reach it over your platform's
internal network (on Railway, `http://sierpe.railway.internal:8080`).
Management surfaces do not face the internet — the same rule of thumb you
apply to RabbitMQ or Postgres.

If you do need a public domain, set `HTTP_BASIC_AUTH=user:password`, which
gates everything except the orchestrator probes.
