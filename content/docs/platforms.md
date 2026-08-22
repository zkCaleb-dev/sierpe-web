---
title: Where it runs
weight: 15
description: Every deployment target we evaluated — what works, what needs one setting, what fails and why — plus the constraints that let you judge a platform we did not list.
---

Sierpe is one long-running process next to a Postgres. That shape rules
some platforms in and others out, and the reasons are always the same
handful. This page states them once, then walks every target we
evaluated. If your platform is not here, the constraints are enough to
judge it.

## The constraints that decide everything

- **It polls.** The ingestion loop asks the Stellar RPC for the next
  ledger every few seconds, forever. A platform that scales to zero,
  sleeps on idle, or only allocates CPU while an HTTP request is in
  flight stops indexing — silently, because `/health` stays green.
- **One replica.** There is no leader election. A second copy is
  harmless to the data (the cursor only moves forward, every insert is
  idempotent) but doubles the RPC load and re-walks backfill chunks.
  Brief overlap during a rolling deploy is fine.
- **It listens on `HTTP_PORT`** (default 8080) and ignores `PORT`. This
  is deliberate: Railway injects `PORT` with a value of its own, and
  honouring it would change the listening port of every Railway
  deployment that did not pin `HTTP_PORT`. Platforms that *assign* a
  random port at runtime (Heroku web dynos) therefore fail; platforms
  that let you *declare* the port work.
- **The image is distroless**: static binary, non-root (UID 65532), no
  shell, no curl. Health checks must be HTTP from outside, or the
  built-in `sierpe healthcheck` (declared as the image `HEALTHCHECK`
  since 1.5.2). `linux/amd64` + `linux/arm64`; the `-full` variant is
  amd64 only, needs a few GB of writable scratch disk, and currently
  runs as root.
- **No disk.** The slim image writes nothing: read-only root filesystem
  is fine, no volume needed, ≥512 MB memory.
- **Postgres ≥ 14** (the driver's floor) over plain TCP with a password
  in the URL. No Unix sockets, no cloud proxies, no IAM auth. It uses
  prepared statements by default and a transaction-scoped advisory lock
  at boot; `sslmode` is honoured, and `verify-full` works against a
  provider's private CA only if you mount the CA file and add
  `sslrootcert=/path/ca.pem` to the URL.
- **Outbound HTTPS** to the RPC and to `ghcr.io` for the image pull.

## App platforms (PaaS)

| Platform | Verdict | What to set |
|---|---|---|
| **Railway** | Works | Image route; domain target port `8080`; healthcheck `/health`. [Guide](/docs/install/#railway) |
| **Render** | Works with config (paid) | Starter or larger; `HTTP_PORT=10000` or rely on port detection; health `/health`, never `/ready`. **Free tier fails** — spins down after 15 min idle |
| **Fly.io** | Works with config | `internal_port = 8080`, `auto_stop_machines = "off"`, `min_machines_running = 1`, memory ≥512 MB, check `/health`. The default `fly launch` toml reintroduces autostop — watch it |
| **DigitalOcean App Platform** | Works with config | `http_port: 8080`, health `/health`, 512 MB+ instance, do not enable inactivity sleep. No persistent disk → no `-full` |
| **Koyeb** | Works with config (paid) | Expose 8080, health `/health`, min=max=1. **Free instance fails** — sleeps after 1 h idle, not disableable. Free Koyeb Postgres burns its 5 compute-hours in a day |
| **Northflank** | Works with config | Port 8080; liveness `/health`, readiness `/ready` (the one PaaS that maps readiness correctly); nf-compute-20 or larger |
| **Zeabur** | Works with config (Dev plan) | Port 8080 declared. **Free plan fails** — sleeps on idle. No health-check configuration exists |
| **Sevalla** | Works with config | Port 8080; liveness `/health`, readiness `/ready` |
| **Porter** | Works with config | BYO cloud; `port: 8080`, `healthCheck /health`, 1 replica |
| **Heroku** | **Fails** as a web dyno | Dynos must bind to a random `$PORT`; Sierpe listens on `HTTP_PORT` by design (see above). Worker dyno runs it but the API is unreachable |
| **Coolify** | Works with config | Docker Image resource, port 8080. Since 1.5.2 the image `HEALTHCHECK` makes Coolify's in-container check pass; on older tags disable health checks |
| **Dokploy** | Works with config | Docker provider, port 8080. Since 1.5.2 the Swarm health check can use the image's own; before, leave it unset |
| **CapRover** | Works with config | Container HTTP Port `8080` (default is 80 — forgetting it is a 502) |
| **Dokku** | Works with config | `ports:set http:80:8080`; startup check `/health` runs from the host, so distroless was never a problem here |

## Hyperscalers

| Platform | Verdict | What to set |
|---|---|---|
| **AWS ECS Fargate** | Works | [Guide](/docs/install/#aws-ecs-fargate--rds) |
| **AWS EKS / GKE / AKS** | Works | Plain Kubernetes — see below |
| **AWS Elastic Beanstalk (Docker)** | Works with config | `Dockerrun.aws.json` v1 with `ContainerPort 8080`; single instance or min=max=1; ALB health `/health` |
| **AWS Lightsail Containers** | Works with config | Power Micro (1 GB) or larger, scale 1, endpoint port 8080, health `/health` |
| **AWS App Runner** | **Fails** | Closed to new customers; ECR-only images; CPU throttled between requests so the poll loop starves while `/health` stays green |
| **AWS Lambda** | **Fails** | Invocation-driven, frozen between calls, 15 min cap |
| **Google Cloud Run** | Works with config | `--no-cpu-throttling --min-instances 1 --max-instances 1 --port 8080`; startup probe `/health`; Cloud SQL via private IP + `sslmode=require` (the built-in Cloud SQL socket is Unix — unusable). Filesystem is RAM → no `-full` |
| **Google GKE Autopilot** | Works | Kubernetes below; ephemeral storage up to 10 GiB covers `-full` on amd64 nodes |
| **Google Compute Engine (COS)** | Works with config | `docker run --restart=always --network host`; the container-VM UI path is deprecated, use cloud-init |
| **Azure Container Apps** | Works with config | `--target-port 8080 --min-replicas 1 --max-replicas 1`; probes on `/health` only. Azure Flexible Server chains to **public** roots: `sslmode=verify-full` works as-is |
| **Azure Container Instances** | Works with config | Declared port 8080, `restartPolicy: Always`, liveness `/health` |
| **Azure App Service for Containers** | Works with config | B1+, **Always On**, `WEBSITES_PORT=8080`, health `/health` |

## Serverless and edge — no

Vercel, Netlify, Cloudflare Workers/Containers, Deno Deploy, Replit and
Glitch all fail on the platform model: they run code in response to
requests and freeze or stop it in between. No change to Sierpe can make
a request-driven runtime keep polling. Cloudflare Containers can be
kept awake with a hand-rolled cron keepalive, without any guarantee.

## Self-hosted and bare metal

| Target | Verdict | Notes |
|---|---|---|
| **Kubernetes** (any distro) | Works | `replicas: 1`, `strategy: Recreate` (or RollingUpdate with readiness on `/health`, not `/ready`, or a long backfill wedges the rollout); liveness `/health`; `readOnlyRootFilesystem: true`, `runAsNonRoot: true` (UID 65532). Pin `-full` to amd64 nodes with an `emptyDir` for scratch |
| **Docker Swarm** | Works | `replicas: 1`, `order: stop-first`; the image `HEALTHCHECK` (1.5.2) gives Swarm its health signal |
| **Nomad** | Works | `count = 1`, `canary = 0`, service check `/health` — never `/ready` as the deployment check |
| **Podman + systemd (Quadlet)** | Works | `.container` unit with `ReadOnly=true`, `Restart=always`; `loginctl enable-linger` for rootless |
| **systemd + release binary** | Works | `DynamicUser=yes`, `ProtectSystem=strict` — it writes nothing |
| **VPS** (Hetzner, OVH, DigitalOcean, Linode, Vultr) | Works | ≥1 GB for Sierpe alone, ≥2 GB with Postgres on the same box; 512 MB tiers OOM during backfill. ARM plans (Hetzner CAX, Oracle A1) run the slim image only |
| **Oracle Cloud Free Tier** | Works with config | A1 (arm64, slim only); note Oracle reclaims idle Always-Free instances — an indexer at the tip is light enough to trip it |
| **Raspberry Pi 4/5** | Works with config | **64-bit OS required** (32-bit pulls fail with "no matching manifest"); ≥2 GB; Postgres on SSD, never SD. No `-full` |
| **NAS** | Works on x86 (Synology +/xs, Unraid, TrueNAS SCALE, QNAP x86) and arm64 QNAP; **fails on 32-bit ARM NAS** | Use the compose/YAML route; skip the exec-based health check UI |
| **Mac (Apple Silicon)** | Works for development | Slim runs natively; `-full` only via Rosetta (OrbStack/Colima `--vz-rosetta`; Docker Desktop needs the Apple Virtualization backend). Laptops sleep — not a server |
| **Windows** | Works via Docker Desktop/WSL2; **no native binary** | WSL2 shuts down on idle and on sleep — not a server |
| **Proxmox** | Works | Binary in an unprivileged LXC (cleanest), or a VM with compose |

## Postgres providers

The three things that decide a provider: whether it is real Postgres,
whether a transaction-mode pooler sits in the path, and what signs its
TLS certificate.

| Provider | Verdict | The URL detail |
|---|---|---|
| **Amazon RDS / Aurora** | Works | `sslmode=require` (private CA); Aurora: the **writer** endpoint |
| **RDS Proxy** | Works | Multiplexes prepared statements since 2023; `verify-full` works (public ACM cert) |
| **Google Cloud SQL** | Works with config | Public or private IP + `sslmode=require`; instance must **not** be set to "require trusted client certificates" |
| **AlloyDB** | Works with config | Public IP + `require`; keep managed pooling off or in session mode |
| **Azure Flexible Server** | Works | `sslmode=verify-full` — public roots. Avoid the built-in PgBouncer on 6432 unless `max_prepared_statements` > 0 |
| **Supabase** | Works with config | **Session pooler** (`…pooler.supabase.com:5432`) or direct (IPv6 only). The **transaction pooler on 6543 fails** unless you append `default_query_exec_mode=simple_protocol` (works since 1.5.2) |
| **Neon** (incl. former Vercel Postgres) | Works | Direct or `-pooler` host both fine; `verify-full` works. Free-tier autosuspend drops connections after 5 idle minutes — a stalled RPC can trigger it; the pool reconnects |
| **Railway Postgres** | Works | Private URL as-is; public proxy with `sslmode=require` |
| **Render Postgres** | Works | Internal URL as-is; external with `require` |
| **Fly Managed Postgres** | Works | Pooled URL defaults to session mode |
| **DigitalOcean Managed** | Works with config | Direct port + `require`; connection pools only in **session** mode |
| **Heroku Postgres** | Works with config | `DATABASE_URL` + `require`; skip the pooled URL |
| **Crunchy Bridge** | Works | `require`; its PgBouncer (5431) supports prepared statements |
| **Timescale Cloud** | Works | `require`; both pools fine (PgBouncer ≥ 1.21) |
| **PlanetScale Postgres** | Works | 5432 or 6432; `verify-full` works |
| **EDB Cloud Service** | Works | `verify-full` (Let's Encrypt); session-mode pooler |
| **Prisma Postgres** | Works with config | The **direct** string, not the pooled one |
| **Xata** | Unverified | SQL proxy behaviour with prepared statements unknown |
| **YugabyteDB** | Works with config | ≥ v2025.1 (advisory locks); port 5433 |
| **CockroachDB** | Unverified | Transaction-scoped advisory locks are supported; `hashtext()` and serialization retries untested |
| **Aurora DSQL** | **Fails** | IAM-only auth, no advisory locks, no `text[]` columns, `numeric` ≤ 38 digits (i128 needs 39) |
| **Spanner (PG interface)** | **Fails** | Requires the PGAdapter sidecar and IAM; no advisory locks |
| **PgBouncer** (self-hosted) | Works | Session mode as-is; transaction mode on ≥ 1.21 with `max_prepared_statements` > 0, else `simple_protocol` |
| **Pgpool-II / Odyssey / pgcat** | Works with config | Pgpool: no load balancing for the writer. Odyssey: `pool_reserve_prepared_statement yes`. pgcat: `prepared_statements = true` |

When a pooler is the problem, Sierpe says so: a boot that dies on a
prepared-statement error (SQLSTATE 26000 / 42P05) logs the
`default_query_exec_mode=simple_protocol` fix in its last line.

## How this page was made

Each cluster was researched against the platforms' current documentation
by an assistant that only knew the constraints above, then checked
against the code. Three deployment paths (Compose, Railway, AWS) were
additionally driven end-to-end by a fresh-context assistant given nothing
but this site. Two product changes came out of it — the simple-protocol
fix and the built-in healthcheck — and one decision not to change
anything (`PORT`). Verdicts marked *unverified* are exactly that; if you
run Sierpe somewhere not listed, [tell us](https://github.com/zkCaleb-dev/sierpe/discussions).
