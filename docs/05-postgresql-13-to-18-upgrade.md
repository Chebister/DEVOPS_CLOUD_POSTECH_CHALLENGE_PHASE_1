# PostgreSQL 13 → 18 Upgrade (Docker Compose)

**Date:** 2026-09-22 · **Environment:** local validation VPS (Docker Compose, `app/` stack) · **Status:** completed and validated

## Summary

The database container of the ToggleMaster stack was upgraded from the official image `postgres:13` (13.23) to `postgres:18` (18.6). Because the upgrade involved a **major version change**, the data directory was **recreated from scratch** (the database held only test data — this was a deliberate, accepted trade-off). The application re-created its schema automatically on startup via `flask init-db` (see `app/entrypoint.sh`).

Two changes were required, not one:

1. Bump the image tag in `app/docker-compose.yaml`: `postgres:13` → `postgres:18`.
2. **Change the volume mount point**: `/var/lib/postgresql/data/` → `/var/lib/postgresql/`.

## Why the mount point had to change

Starting with the 18.x images, the official `postgres` Docker image changed its data-directory layout to be compatible with `pg_ctlcluster`: `PGDATA` now defaults to a **major-version-specific path** (`/var/lib/postgresql/18/docker`), and the recommended single mount is `/var/lib/postgresql`, so that future major upgrades can use `pg_upgrade --link` without mount-point boundary issues.

- Reference: [docker-library/postgres PR #1259](https://github.com/docker-library/postgres/pull/1259) and [issue #37](https://github.com/docker-library/postgres/issues/37).
- The image's entrypoint (`docker_error_old_databases`) **intentionally refuses to start** if it detects a mount at the legacy `/var/lib/postgresql/data` path — even if the volume is **empty**:

  ```text
  Error: in 18+, these Docker images are configured to store database data in a
  format which is compatible with "pg_ctlcluster" ... Counter to that, there
  appears to be PostgreSQL data in: /var/lib/postgresql/data (unused mount/volume).
  ```

  This was confirmed empirically: starting `postgres:18` with the old mount failed with exit code 1 and the error above.

Additionally, PostgreSQL itself cannot open a data directory initialized by an older major version ("database files are incompatible with server"), so simply swapping the image tag while keeping the old volume would also fail even without the entrypoint check.

## Upgrade path for major versions

- **Data preserved (small DBs):** `pg_dumpall` (13) → restore into a fresh `postgres:18` container. Downtime proportional to dump/restore size.
- **Data preserved (large DBs):** `pg_upgrade` (or `pg_upgrade --link`) — requires both version binaries; facilitated by mounting at `/var/lib/postgresql` so old and new data dirs sit side by side (e.g. `13/docker` and `18/docker`).
- **Data disposable (this project):** remove the volume, let `postgres:18` run `initdb`, and let the app re-create its schema (`flask init-db`). Chosen here.

## Performed procedure

```bash
cd /opt/togglemaster/app

# 0. Backup of the compose file (rollback reference)
cp docker-compose.yaml docker-compose.yaml.bak-pg13

# 1. New image + new mount point
sed -i 's/postgres:13/postgres:18/' docker-compose.yaml
sed -i 's|postgres_data:/var/lib/postgresql/data/|postgres_data:/var/lib/postgresql/|' docker-compose.yaml

# 2. Tear down the stack and remove the old data volume
docker compose down
docker volume rm app_postgres_data

# 3. Start the stack; the 18 image runs initdb and the app runs `flask init-db`
docker compose up -d
```

## Validation

| Check | Result |
| :--- | :--- |
| `docker ps` | `app-db-1` (postgres:18) **Up · healthy**; `app-app-1` Up |
| `SELECT version();` | `PostgreSQL 18.6 (Debian 18.6-1.pgdg13+2)` |
| App logs | `Tabela 'flags' inicializada com sucesso.` |
| `GET /health` | `200 {"status": "ok"}` |
| `GET /flags` | `200 []` (schema re-created, data intentionally discarded) |
| `POST /flags` + `GET /flags/<name>` | `201` / `200` — full CRUD working |

## Rollback plan

Restore the backup and recreate the previous version's volume (old data was discarded; rollback implies a fresh PG 13 database):

```bash
cd /opt/togglemaster/app
docker compose down
docker volume rm app_postgres_data
mv docker-compose.yaml.bak-pg13 docker-compose.yaml
docker compose up -d   # postgres:13 runs initdb; app re-creates schema
```

## Future major upgrades (13→18 lesson learned)

- Keep the single mount at `/var/lib/postgresql` (already done) so data lives in `<major>/docker` subdirectories.
- To upgrade **with data**, run a side-by-side `pg_upgrade` between the old and new version subdirectories, or dump/restore — never reuse the old data directory with the new image.
