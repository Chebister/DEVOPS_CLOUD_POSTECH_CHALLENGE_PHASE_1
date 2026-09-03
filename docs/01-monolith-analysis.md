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

_TODO: describe (to be confirmed after running locally)_

- All code (presentation, business logic, and data access) lives in a single process/deployment (`app.py`).
- There is no separation into independent services: the API and the database form a single deployment unit.
- Scaling and evolution happen on the application as a whole, not per component.

## Advantages for an MVP

_TODO_

- Simplicity of development and understanding (a single codebase).
- Single, fast deployment — ideal for validating the idea quickly.
- Lower operational overhead (no orchestration, messaging, or inter-service networking).
- More straightforward debugging and testing.

## Disadvantages / limitations

_TODO_

- Coupling: any change requires redeploying the entire application.
- Scales as a single block (cannot scale reads, writes, etc. independently).
- Single point of failure: if the process goes down, the whole platform goes down.
- Growth tends to produce code that is hard to maintain (evolution is planned for later phases of the course).

## Notes from running locally

_TODO: record results of `docker compose up` and tests with curl/Postman_

## References

- [Application README](../app/README.md)
- Challenge statement: `POSTECH - DCLT - Tech Challenge - Fase 1.pdf`
