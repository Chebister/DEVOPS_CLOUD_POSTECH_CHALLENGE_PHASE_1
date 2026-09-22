# Tech Challenge — Phase 1 | ToggleMaster on AWS

Repository for **Phase 1 of the Tech Challenge of the DevOps & Cloud graduate program (POSTECH)**: deploying the **ToggleMaster** monolithic MVP (a Feature Flags platform) to AWS.

**Participant:** Yuri

## Repository structure

| Folder / File | Description |
| :--- | :--- |
| [`app/`](app/) | Source code of the monolithic application (Flask + PostgreSQL) |
| [`docs/`](docs/) | Challenge statement and technical documentation |
| [`evidence/`](evidence/) | Screenshots (Security Groups, cost estimate, running app) |
| [`delivery/DELIVERY.md`](delivery/DELIVERY.md) | Final delivery report |

## Technical documentation

- [Monolithic application analysis (pros and cons)](docs/01-monolith-analysis.md)
- [12-Factor App discussion](docs/02-12-factor-app.md)
- [Proposed AWS architecture](docs/03-aws-architecture.md)
- [AWS cost estimate](docs/04-cost-estimate.md)
- [PostgreSQL 13 → 18 upgrade (Docker Compose)](docs/05-postgresql-13-to-18-upgrade.md)
- [Python and dependency update (base image, packages, PostgreSQL client)](docs/06-python-and-dependency-update.md)
- [Config via environment: removing credentials from docker-compose.yaml](docs/07-config-via-env-files.md)

## How to run the application locally

```bash
cd app
docker compose up --build
curl http://localhost:5000/health
```

Full instructions (local + EC2 deployment) in the [application README](app/README.md) (in Portuguese, upstream repository).

## Current deployment (local validation environment)

The application is currently running with Docker on a Linux VPS (Rocky Linux 9), fronted by a [Caddy](https://caddyserver.com/) reverse proxy with a valid Let's Encrypt certificate (auto-renewal):

| URL | Description |
| :--- | :--- |
| **https://toggle-local.cheb.com.br/health** | Public health check |
| **https://toggle-local.cheb.com.br/flags** | Feature flags API |

```text
Internet --> Caddy (80/443, TLS + auto HTTPS redirect) --> app container (Flask/Gunicorn :5000) --> db container (PostgreSQL + persistent volume)
```

## Useful links

- **Application (local validation environment):** https://toggle-local.cheb.com.br
- **Architecture diagram:** _TODO (Miro / diagrams.net)_
- **Demo video:** _TODO_
- **Original application repository:** https://github.com/dougls/toggle-master-monolith

## Security

- AWS and database credentials are managed **exclusively via environment variables** and are **never** committed to this repository.
- See [`.gitignore`](.gitignore) for blocked patterns.
