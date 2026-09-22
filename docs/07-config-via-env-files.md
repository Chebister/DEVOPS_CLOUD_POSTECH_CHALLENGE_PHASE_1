# Config via Environment: Removing Credentials from docker-compose.yaml

**Date:** 2026-09-22 · **Environment:** local validation VPS (Docker) · **Status:** completed and validated

## Summary

As part of the 12-Factor App review (factor III, Config), the hardcoded credentials in `app/docker-compose.yaml` were replaced with variable interpolation. The real values now live in a **`.env` file on the deployment machine**, which is **gitignored and never committed**. The same credential values as before were kept, so the database and its data were not affected.

## Motivation (factor III: "store config in the environment")

The application code already satisfied factor III (`app.py` reads everything from `DB_*` environment variables). The gap was in the packaging: `docker-compose.yaml` had `POSTGRES_USER=user` and `POSTGRES_PASSWORD=password` hardcoded **in a public Git repository**, which:

- exposes the default credentials to anyone who reads the repository;
- invites the default credentials to leak into production by inertia (nobody changes them at deploy time);
- mixes config (what varies per deployment) with code.

## Changes

### `app/docker-compose.yaml`

All credential and database-name literals were replaced with `${...}` interpolation, resolved by Docker Compose from the `.env` file in the project directory:

```yaml
app:
  environment:
    - DB_HOST=db
    - DB_NAME=${POSTGRES_DB}
    - DB_USER=${POSTGRES_USER}
    - DB_PASSWORD=${POSTGRES_PASSWORD}
    - DB_PORT=5432
db:
  environment:
    - POSTGRES_USER=${POSTGRES_USER}
    - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
    - POSTGRES_DB=${POSTGRES_DB}
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
```

### `app/.env` (deployment machine only, gitignored)

Holds the real values. Permissions restricted (`chmod 600`). The `.gitignore` already blocked `.env`, and the file is confirmed untracked in Git.

### `app/.env.example` (committed)

Template with placeholder values documenting the required variables, the standard practice so new developers know what to configure.

## Validation (2026-09-22, all passed)

| Check | Result |
| :--- | :--- |
| `docker compose config` | Variables resolved correctly from `.env` |
| Container recreation | Not needed: interpolated values are identical to the previous literals (Compose config hash unchanged), which confirms equivalence |
| `GET /health` | `200 {"status": "ok"}` |
| `GET /flags` | `200` with the existing flag (database and data untouched) |
| Git | `.env` not tracked (`.gitignore`); only compose and `.env.example` committed |

## Production evolution (AWS, later phase)

On AWS the same pattern is strengthened: the EC2 instance receives an IAM Role (no static keys) and fetches the database credentials from **AWS Secrets Manager** or **SSM Parameter Store** (SecureString), exporting them as environment variables for the application process. No secret in Git, no permanent `.env` on disk, with rotation and audit support.

## References

- [12-Factor App, factor III (Config)](https://12factor.net/config) and the project discussion in [02-12-factor-app.md](02-12-factor-app.md)
