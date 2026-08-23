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
