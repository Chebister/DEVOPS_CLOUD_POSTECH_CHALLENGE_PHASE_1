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

## How to run the application locally

```bash
cd app
docker compose up --build
curl http://localhost:5000/health
```

Full instructions (local + EC2 deployment) in the [application README](app/README.md) (in Portuguese, upstream repository).

## Useful links

- **Architecture diagram:** _TODO (Miro / diagrams.net)_
- **Demo video:** _TODO_
- **Original application repository:** https://github.com/dougls/toggle-master-monolith

## Security

- AWS and database credentials are managed **exclusively via environment variables** and are **never** committed to this repository.
- See [`.gitignore`](.gitignore) for blocked patterns.
