# Python and Dependency Update (Base Image, Packages, PostgreSQL Client)

**Date:** 2026-09-22 · **Environment:** local validation VPS (Docker) · **Status:** completed and validated

## Summary

While applying the 12-Factor App review (factor II, Dependencies), the application base image and its declared dependencies were updated to current versions:

| Component | Before | After |
| :--- | :--- | :--- |
| Base image | `python:3.9-slim` | `python:slim` (Python 3.14.7) |
| Flask | 2.2.2 | 3.1.3 |
| Werkzeug | 2.3.8 | 3.1.8 |
| psycopg2-binary | 2.9.5 | 2.9.13 |
| Gunicorn | 20.1.0 | 26.2.0 |
| PostgreSQL client (apt) | `postgresql-client` (unpinned, Debian default 13) | `postgresql-client-18` (official PGDG apt repository) |

## Motivation

- **Python 3.9 reached end of life in October 2025** and no longer receives security fixes, so the `python:3.9-slim` base image was unsuitable for production.
- The `RUN apt-get update && apt-get install -y postgresql-client` line was unpinned: each rebuild could resolve a different client version, breaking build reproducibility (12-Factor factor II).
- The pinned Python dependencies had no wheels for modern Python versions: building `psycopg2-binary==2.9.5` on Python 3.12+ fails because it falls back to a source build that requires `pg_config` (confirmed empirically during the first upgrade attempt).

## Changes

### `app/Dockerfile`

- Base image changed to `python:slim`, which tracks the latest stable Python (currently 3.14.7). Note: this is a floating tag, which trades strict build reproducibility for automatic access to supported Python versions; the decision was deliberate for this validation environment. For strict reproducibility, pin a specific tag such as `python:3.14-slim` in CI builds.
- The PostgreSQL client is now installed from the **official PGDG apt repository** (apt.postgresql.org), pinned to major version 18, matching the database server (`postgres:18`, see [05-postgresql-13-to-18-upgrade.md](05-postgresql-13-to-18-upgrade.md)).
- The apt layer now follows image best practices: `--no-install-recommends`, removal of the fetch tools (`curl`, `gnupg`) after use, and cleanup of the apt lists, reducing image size and attack surface.

### `app/requirements.txt`

- All dependencies upgraded to the latest versions that support Python 3.14: `Flask==3.1.3`, `Werkzeug==3.1.8`, `psycopg2-binary==2.9.13`, `gunicorn==26.2.0`. Versions remain fully pinned.

## Validation (2026-09-22, all passed)

| Check | Result |
| :--- | :--- |
| Image build | Succeeded on `python:slim` (Python 3.14.7) |
| `python --version` in app container | Python 3.14.7 |
| `psql --version` / `pg_isready --version` in app container | 18.6 (PGDG build, matching the server) |
| Gunicorn startup | Gunicorn 26.2.0, worker booted |
| `GET /health` (local and public) | `200 {"status": "ok"}` |
| `GET /flags` (local and public) | `200` with the existing flag (data preserved) |
| Database | Unchanged (`postgres:18`, healthy) |

## Rollback

Revert `app/Dockerfile` and `app/requirements.txt` to the previous commit and rebuild (`docker compose up -d --build`).

## References

- [12-Factor App, factor II (Dependencies)](https://12factor.net/dependencies) and the project discussion in [02-12-factor-app.md](02-12-factor-app.md)
- [PostgreSQL 13 to 18 upgrade](05-postgresql-13-to-18-upgrade.md)
