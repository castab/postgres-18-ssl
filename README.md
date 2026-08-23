# SSL-enabled Postgres 18 Image
Postgres 18 template for Railway deployment and hosting.

Largely taken from the [Railway's repository](https://github.com/railwayapp-templates/postgres-ssl) with changes to support Postgres 18.

[![Deploy on Railway](https://railway.app/button.svg)](https://railway.com/deploy/postgresql-18-1?referralCode=TQuzNl&utm_medium=integration&utm_source=template&utm_campaign=generic)

## ⚠️ Important: Upgrading from PostgreSQL 17 or Earlier

**DO NOT simply change the image version and restart your service.** PostgreSQL 18 introduced breaking changes to the data directory structure that will result in data loss if not handled properly.

### The Problem
- PostgreSQL 17 and earlier use `/var/lib/postgresql/data` as the data directory
- PostgreSQL 18+ uses `/var/lib/postgresql/18/docker` (version-specific path)
- PostgreSQL 18 cannot read PostgreSQL 17 database files directly

### Migration Steps

If you're upgrading from PostgreSQL 17:

1. **Create a backup** of your existing database using `pg_dumpall`
2. **Deploy a NEW PostgreSQL 18 service** (don't reuse the old volume)
3. **Use Railway's PostgreSQL Migration template** or manually restore your backup
4. **Update your application** to use the new database URL
5. **Verify everything works** before deleting the old service

### For New Deployments

If you're deploying a fresh PostgreSQL 18 database, you can use this template directly. Just ensure your Railway volume is mounted to `/var/lib/postgresql` (not `/var/lib/postgresql/data`).

### Volume Mount Configuration

- **Volume Mount Path**: `/var/lib/postgresql`
- **PGDATA** (auto-set): `/var/lib/postgresql/18/docker`

See [Railway's PostgreSQL upgrade guide](https://docs.railway.app) for detailed migration instructions.

## Postgres Version / CVE-2026-15741 Remediation

This image defaults to **Postgres 18.6**, which fixes [CVE-2026-15741](https://www.postgresql.org/support/security/CVE-2026-15741/) (SQL injection via `EXTRACT()` expression deparse, CVSS 8.8). The CVE was patched upstream as of PostgreSQL 18.5, but the `postgres:18.5` Docker image tag was never published (Docker Hub jumps from `18.4` straight to `18.6`), so `18.6` is the earliest available image containing the fix. If you're running an image built before this change, rebuild and redeploy to pick up the fix.

### Overriding the Postgres version

The base image tag is controlled by the `POSTGRES_VERSION` Docker build argument, declared in `Dockerfile.18`:

```dockerfile
ARG POSTGRES_VERSION=18.6
FROM postgres:${POSTGRES_VERSION}
```

**On Railway:** set a service Variable named `POSTGRES_VERSION` (Service → Variables) to any valid `postgres` image tag, e.g. `18.6`, `18.6-bookworm`, or `18.6-alpine`. Railway automatically passes matching service Variables as Docker build args when the Dockerfile declares a matching `ARG` — no changes to `railway.toml` are required. Redeploy the service to trigger a rebuild with the new version.

**Local build:**

```bash
docker build -f Dockerfile.18 --build-arg POSTGRES_VERSION=18.6-bookworm -t postgres-18-ssl .
```

> **Note:** `POSTGRES_VERSION` must stay within the Postgres **18.x** line — this image's data directory handling (see PGDATA migration above) and SSL setup scripts assume the Postgres 18 major version. The build enforces this: setting `POSTGRES_VERSION` to a different major version (e.g. `17` or `19`) fails the Docker build with an explicit error instead of producing a broken image.

## Environment Variables

**No configuration is required to deploy this template on Railway.** Every variable below ships with a secure, working default — a random password is generated automatically, networking is wired to Railway's private domain, and SSL is enabled out of the box. Click "Deploy on Railway" and it works as-is.

The variables are documented below for reference, and in case you want to customize the deployment.

### Wired / computed — leave as-is

These reference other variables or Railway platform values. Change the variable they point to instead of editing these directly (e.g. edit `POSTGRES_DB`, not `PGDATABASE`).

| Variable | Default | Description |
| --- | --- | --- |
| `PGDATA` | `/var/lib/postgresql/18/docker` | Postgres 18's data directory path (version-specific, relative to the mounted volume). |
| `PGHOST` | `${{RAILWAY_PRIVATE_DOMAIN}}` | Internal hostname for connecting to Postgres over Railway's private network. |
| `PGUSER` | `${{POSTGRES_USER}}` | Connecting user, mirrors `POSTGRES_USER`. |
| `PGDATABASE` | `${{POSTGRES_DB}}` | Database to connect to, mirrors `POSTGRES_DB`. |
| `PGPASSWORD` | `${{POSTGRES_PASSWORD}}` | Connecting user's password, mirrors `POSTGRES_PASSWORD`. |
| `DATABASE_URL` | computed | Full connection string for internal access from other services in the same Railway project. |
| `DATABASE_PUBLIC_URL` | computed | Full connection string for external access via Railway's public TCP proxy. |

### Optional overrides

Safe to change in Railway's Service → Variables if you want different defaults.

| Variable | Default | Description |
| --- | --- | --- |
| `POSTGRES_DB` | `railway` | Name of the default database created on first init. |
| `POSTGRES_USER` | `postgres` | Superuser account created on first init. |
| `POSTGRES_PASSWORD` | auto-generated (32 chars) | Password for `POSTGRES_USER`, generated automatically by Railway. Override only if you need a specific password. |
| `PGPORT` | `5432` | Port Postgres listens on. |
| `SSL_CERT_DAYS` | `820` | Validity period, in days, for the self-signed SSL certs generated at init. |
| `POSTGRES_VERSION` | `18.6` | Postgres image tag to build (see [Overriding the Postgres version](#overriding-the-postgres-version) above). Build-time only, unlike the other variables here which are read at container runtime — must stay within the 18.x line. |
| `RAILWAY_DEPLOYMENT_DRAINING_SECONDS` | `60` | Grace period between SIGTERM and SIGKILL for the old deployment during a rollout, letting in-flight connections finish. |
