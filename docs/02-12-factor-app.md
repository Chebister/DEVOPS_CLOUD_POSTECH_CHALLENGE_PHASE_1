# Discussion: 12-Factor App applied to ToggleMaster

> Challenge goal: read and understand the 12 Factors (12-Factor App) and identify which ones the application already satisfies and which ones would need adjustments for a more robust production environment.

Official reference: https://12factor.net/

## Summary table

| # | Factor | Status in the MVP | Notes |
| :--- | :--- | :--- | :--- |
| I | Codebase | Yes | One codebase versioned in Git, many deploys (local, EC2) |
| II | Dependencies | Yes | `requirements.txt` declares dependencies explicitly |
| III | Config | Yes | Database credentials via environment variables (`DB_*`) |
| IV | Backing services | Yes | PostgreSQL treated as an attached resource (address/credentials via env) |
| V | Build, release, run | Partial | Docker image separates build/run; missing a versioned release pipeline |
| VI | Processes | Yes | App is stateless (state lives in the database); Gunicorn with workers |
| VII | Port binding | Yes | App exports HTTP on port 5000 |
| VIII | Concurrency | Partial | Gunicorn enables workers; no automated horizontal scaling |
| IX | Disposability | Partial | Process starts/stops quickly, but no explicit graceful shutdown |
| X | Dev/prod parity | Partial | Local Docker approximates environments; RDS (managed) vs local container |
| XI | Logs | No | Logs only go to process stdout; no handling/centralization |
| XII | Admin processes | Yes | CLI command `flask init-db` as a one-off process |

_TODO: revisit and detail each factor after running the application and reflecting on the AWS deployment._

## Factors satisfied (details)

_TODO_

## Factors that need adjustments for production (details)

_TODO_

### Config (III)

The application already reads `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` from the environment — a good starting point. Possible improvements: use managed secrets (AWS Secrets Manager / SSM Parameter Store) instead of manually running `export` in the SSH session.

### Logs (XI)

Flask/Gunicorn write to stdout; in production it would be recommended to ship logs to CloudWatch Logs.

### Dev/prod parity (X)

Locally PostgreSQL runs in a container; production will use managed RDS. Version and performance differences must be considered.
