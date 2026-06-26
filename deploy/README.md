# Langfuse — Viewport Deploy Overlay

Self-host deployment of [Langfuse](https://github.com/langfuse/langfuse) v3 for the
**Monitoring and Observability** department, deployed via **Dokploy** on the Viewport
**new** Docker engine (`/var/run/docker-viewport.sock`).

Part of viewport-ops #404 · service sub-issue #412.

## What this is

`deploy/docker-compose.yml` is a deploy overlay derived from the upstream
`langfuse/langfuse` root `docker-compose.yml` (branch `main`). It is the **full v3
data plane** — Langfuse is not a single container:

| Service | Image (pinned) | Role |
|---|---|---|
| langfuse-web | `langfuse/langfuse:3.201.1` | Web UI + API |
| langfuse-worker | `langfuse/langfuse-worker:3.201.1` | Async ingestion worker |
| postgres | `postgres:17` | Transactional DB |
| clickhouse | `clickhouse/clickhouse-server:25.8` | OLAP / traces store (Langfuse requires >= 24.3) |
| redis | `redis:7` | Queue / cache (BullMQ) |
| minio | `cgr.dev/chainguard/minio` | S3-compatible blob store (events, media, exports) |

## Deliberate differences from upstream

1. **No host port publishing.** Every `ports:` block is removed. Services communicate
   over the internal Dokploy/compose network only. No DNS, no public subdomain, no
   `:80`/`:443` (gated / out of scope). Health verification is internal.
2. **Image tags pinned** for reproducibility. Upstream floats `:3` and
   `clickhouse-server:latest`. Pinned here to `3.201.1` (latest stable release at deploy
   time) and `clickhouse-server:25.8`.
3. **Secrets have no insecure inline defaults.** All `# CHANGEME` values (DATABASE_URL,
   SALT, ENCRYPTION_KEY, NEXTAUTH_SECRET, *_PASSWORD, REDIS_AUTH, S3 secret keys) are
   `${VAR}` with no fallback — a missing secret fails the deploy loudly instead of
   booting with upstream demo values (`mysecret`, `miniosecret`, `clickhouse`, ...).
4. **Internal S3 endpoints.** S3/MinIO endpoints that upstream pointed at
   `http://localhost:9090` are repointed at `http://minio:9000` (the internal service),
   since there is no published host port in this deployment.
5. **Named volumes kept** (Dokploy backs up named volumes only). No `container_name`
   set, so Dokploy can namespace the stack for isolated deployment.

## Required secrets (set in the Dokploy Environment UI — never commit values)

| Variable | How to generate / value |
|---|---|
| `DATABASE_URL` | `postgresql://postgres:<POSTGRES_PASSWORD>@postgres:5432/postgres` |
| `POSTGRES_PASSWORD` | random |
| `NEXTAUTH_SECRET` | `openssl rand -base64 32` |
| `SALT` | `openssl rand -base64 32` |
| `ENCRYPTION_KEY` | `openssl rand -hex 32` (must be 64 hex chars) |
| `CLICKHOUSE_PASSWORD` | random (must match across web/worker/clickhouse) |
| `REDIS_AUTH` | random (must match redis `--requirepass`) |
| `MINIO_ROOT_PASSWORD` | random |
| `LANGFUSE_S3_EVENT_UPLOAD_SECRET_ACCESS_KEY` | = `MINIO_ROOT_PASSWORD` |
| `LANGFUSE_S3_MEDIA_UPLOAD_SECRET_ACCESS_KEY` | = `MINIO_ROOT_PASSWORD` |
| `LANGFUSE_S3_BATCH_EXPORT_SECRET_ACCESS_KEY` | = `MINIO_ROOT_PASSWORD` |

Optional: `NEXTAUTH_URL` (defaults `http://localhost:3000`), `LANGFUSE_INIT_*`
(headless org/project/user/API-key seeding on first boot), `SMTP_CONNECTION_URL` +
`EMAIL_FROM_ADDRESS`.

## Resource requirements

Langfuse docs: **>= 4 cores, >= 16 GiB RAM, ~100 GiB storage** for the full stack
(ClickHouse is memory-hungry). VPS verified at deploy: 12 cores / 47 GiB RAM / 472 GiB free.

## Deploy (Dokploy, new engine)

1. Dokploy at `http://194.163.153.171:3001/` → **Monitoring and Observability** project.
2. **Create Compose service**, type **Docker Compose**.
3. Source = Git → this fork (`viewport-corp/fork-langfuse`), branch `viewport/deploy`,
   **Compose Path** = `deploy/docker-compose.yml`.
4. Set all required secrets above in the Dokploy **Environment** UI (variable names only).
5. Deploy on the new engine only (`DOCKER_HOST=unix:///var/run/docker-viewport.sock`).
6. Verify internally: web health at container `:3000/api/public/health`, and
   `service_healthy` on postgres / clickhouse / redis / minio. No public exposure.

## Upstream sync

```
git fetch upstream            # upstream = https://github.com/langfuse/langfuse.git
git rebase upstream/main      # or merge; re-apply overlay if upstream compose changed
```
