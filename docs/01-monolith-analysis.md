# Monolithic Application Analysis

> Challenge goal: run the application locally, understand how it works, and discuss why it is considered a "monolith", along with the pros and cons of this approach for an MVP.

## Application overview

The ToggleMaster MVP is a REST API for managing Feature Flags, composed of:

- **`app.py`** — a single Flask application containing all responsibilities: API routes, database access, business rules, and initialization.
- **PostgreSQL** — relational database with a single table (`flags`).
- **`Dockerfile` / `docker-compose.yaml`** — packaging of the application and database for local execution.

### Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/health` | Application health check |
| `POST` | `/flags` | Creates a feature flag |
| `GET` | `/flags` | Lists all flags |
| `GET` | `/flags/<name>` | Returns the status of a flag |
| `PUT` | `/flags/<name>` | Updates the status of a flag |

## Why is it a monolith?

- All code (presentation, business logic, and data access) lives in a single process/deployment (`app.py`).
- There is no separation into independent services: the API and the database form a single deployment unit.
- Scaling and evolution happen on the application as a whole, not per component.

## Advantages for an MVP

- Simplicity of development and understanding (a single codebase).
- Single, fast deployment — ideal for validating the idea quickly.
- Lower operational overhead (no orchestration, messaging, or inter-service networking).
- More straightforward debugging and testing.

## Disadvantages / limitations

- Coupling: any change requires redeploying the entire application.
- Scales as a single block (cannot scale reads, writes, etc. independently).
- Single point of failure: if the process goes down, the whole platform goes down.
- Growth tends to produce code that is hard to maintain (evolution is planned for later phases of the course).

## Notes from running locally

**Environment:** Linux VPS (Rocky Linux 9), Docker 29.8.0 + Docker Compose v5.5.1, cloned from this repository to `/opt/togglemaster` and started with `docker compose up -d --build`. A [Caddy](https://caddyserver.com/) reverse proxy (in `/opt/proxy`) terminates TLS on ports 80/443 with a Let's Encrypt certificate and forwards to the app on port 5000.

**Public URL:** https://toggle-local.cheb.com.br

**Endpoint tests (2026-09-03, all passed):**

| Test | Request | Result |
| :--- | :--- | :--- |
| Health check | `GET /health` | `{"status":"ok"}` (200) |
| Create flag | `POST /flags` `{"name":"new-feature","is_enabled":true}` | 201, flag created |
| List flags | `GET /flags` | `[{"is_enabled":true,"name":"new-feature"}]` |
| Get flag | `GET /flags/new-feature` | `{"is_enabled":true,"name":"new-feature"}` |
| Update flag | `PUT /flags/new-feature` `{"is_enabled":false}` | 200, flag updated |
| Duplicate flag | `POST /flags` (existing name) | 409 with error message |
| Persistence | `docker compose restart` | Data preserved (PostgreSQL volume) |

**Confirmation of monolithic behavior observed during execution:** the entire application (routes, business rules, database access) runs as a single process inside one container; the only separate component is the database. Restarting the app container restarts the whole platform, and there is no way to deploy or scale parts of it independently.

## References

- [Application README](../app/README.md)
- Challenge statement: `POSTECH - DCLT - Tech Challenge - Fase 1.pdf`
