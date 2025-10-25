# SSL-enabled Postgres 18 Image
Postgres 18 template for Railway

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