# Architecture Patterns

**Domain:** Single-user book CRUD and search web application  
**Project:** Book Tracker  
**Researched:** 2026-08-13  
**Confidence:** HIGH for the proposed MVP boundaries; MEDIUM for implementation details that depend on the final package/version choices.

## Recommended Architecture

Use a small, layered two-tier application: a React browser client calls a versioned FastAPI HTTP API; the API owns validation, business rules, transactions, and SQLite persistence. Keep the database inaccessible to the browser and do not put SQL in route handlers.

```text
Browser
  React UI state + forms + API client
        │ HTTP/JSON (localhost:8000 in dev)
        ▼
FastAPI API container
  routes → Pydantic schemas → book service → repository/session
                                                │ committed SQL transactions
                                                ▼
                                        SQLite file (/data/books.db)

Docker Compose: frontend container (static files/nginx) + api container + named db volume
```

For development, the frontend may run Vite on port 5173 while the API runs on 8000. The production-like Compose path should build the React app and serve its static output from an Nginx frontend container on port 80. The API remains a separate container on port 8000 so the learning boundary is visible and independently testable.

### Component Boundaries

| Component | Responsibility | Communicates With |
|-----------|----------------|-------------------|
| `App` / page shell | Owns collection-level state: books, query, loading/error/status, and edit mode | Search bar, book form, collection/list items, API client |
| Book form | Controlled inputs, client-side validation, accessible labels/errors, submit-disabled state | App callbacks; API client through App |
| Search bar | Debounced or submit-triggered case-insensitive query state; clear action | App state |
| Book collection/list | Deterministic rendering, empty/no-match states, edit/delete actions | App callbacks |
| API client module | One fetch wrapper, JSON parsing, timeout/error normalization, base URL configuration | React components/hooks; FastAPI API |
| FastAPI routes | HTTP verbs/status codes, dependency injection, request/response schemas | Book service; health endpoint |
| Pydantic schemas | Canonical API input/output validation and serialization | Routes; shared contract documentation |
| Book service | Use-case orchestration: normalize input, select repository operation, map not-found cases | Routes; repository |
| Repository/data-access layer | Queries and persistence only; no HTTP or UI concerns | SQLAlchemy/SQLite session |
| Database/session module | Engine, session lifecycle, schema initialization/migrations, transaction boundaries | Repository; test overrides |
| SQLite database | Durable single-user record store | API only |
| Compose/Nginx | Process isolation, browser-facing static serving, ports, volume, health checks | Frontend and API containers |

Prefer feature-oriented backend files (`books/routes.py`, `books/schemas.py`, `books/service.py`, `books/repository.py`) over a single large `main.py`. The frontend can remain small, but keep network code out of presentational components.

## API Contract

Use `/api/v1` as the stable prefix even for the first release. FastAPI/Pydantic models should generate the OpenAPI contract; the frontend API client should be written against that explicit shape rather than database rows.

### Resource shape

```json
{
  "id": 1,
  "title": "The Left Hand of Darkness",
  "author": "Ursula K. Le Guin",
  "publication_year": 1969,
  "genre": "Science Fiction",
  "isbn": "9780441478125"
}
```

`id` is server-generated. Required text values are trimmed before domain validation; blank values are rejected. Keep `publication_year` an integer and `isbn` a normalized string so leading zeroes and ISBN-10/ISBN-13 formatting are not lost. The exact accepted ISBN normalization rule must be implemented once in the backend and mirrored by the frontend for immediate feedback; the backend remains authoritative.

### Endpoints

| Method | Path | Request | Success | Failure cases |
|--------|------|---------|---------|---------------|
| `GET` | `/api/v1/health` | None | `200 {"status":"ok"}` and database readiness check | `503` if API cannot use its database |
| `GET` | `/api/v1/books?q=<substring>` | Optional `q`; empty/missing means all | `200` array of `Book` objects | `422` only for malformed query constraints; otherwise return `[]` for no match |
| `POST` | `/api/v1/books` | `BookCreate` JSON | `201` created `Book` | `422` validation error; `500` unexpected persistence failure |
| `GET` | `/api/v1/books/{id}` | None | `200` `Book` | `404` if absent |
| `PUT` | `/api/v1/books/{id}` | Complete `BookUpdate` JSON | `200` updated `Book` | `404`, `422` |
| `DELETE` | `/api/v1/books/{id}` | None | `204` no body | `404` |

Use `PUT` for the full edit form: it makes the update contract unambiguous and matches the form's complete record. Do not accept a request body on `GET`. A consistent validation error should expose field locations and messages in a predictable envelope, for example:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Book data is invalid",
    "fields": {
      "isbn": "Enter a valid ISBN-10 or ISBN-13"
    }
  }
}
```

If relying on FastAPI's native validation format, normalize it in the exception handler to this application envelope so the React client does not depend on framework internals. Return `404` for missing IDs and a generic non-sensitive message for unexpected failures.

## Database Contract

The initial schema is one table, `books`:

| Column | SQLite type | Constraints |
|--------|-------------|-------------|
| `id` | `INTEGER` | Primary key, server generated |
| `title` | `TEXT` | `NOT NULL`, trimmed non-empty |
| `author` | `TEXT` | `NOT NULL`, trimmed non-empty |
| `publication_year` | `INTEGER` | `NOT NULL`, validated supported year range |
| `genre` | `TEXT` | `NOT NULL`, trimmed non-empty |
| `isbn` | `TEXT` | `NOT NULL`, normalized representation |

Add indexes on `title` and `author` only if query inspection shows they are useful; the MVP collection is explicitly small and substring search with SQLite `LIKE` is sufficient. The list query must include an explicit deterministic order, preferably `ORDER BY title COLLATE NOCASE, author COLLATE NOCASE, id`, rather than relying on insertion order.

Use SQLAlchemy 2.x (or an equally explicit SQLAlchemy-backed repository) with one request-scoped session and a transaction around every mutation. Commit only after successful validation and persistence; rollback on exceptions. Configure the SQLite connection deliberately, including a writable database directory and, where supported by the chosen Python version, modern transaction behavior. Do not treat `create_all()` as a substitute for versioned migrations once schema evolution begins; for the greenfield MVP it can bootstrap the first schema, but introduce Alembic before v2 changes.

Testing must inject a separate database URL, preferably a temporary file or properly configured shared in-memory SQLite database. Never let tests point at the Docker volume. Verify that create/update/delete changes are committed and that a recreated API container sees data when the named volume remains.

## UI Contract and Data Flow

The React app should have one collection controller (App or a feature hook) as the source of truth:

1. On mount, call `GET /api/v1/books`; show loading, then list, empty, or API-failure state.
2. On query change, call the same endpoint with `q`; preserve the query in state and show no-match separately from an empty collection. Clearing the query requests or restores the full list.
3. On create, validate locally, disable the submit control, send `POST`, replace or append the returned server record, then show success feedback.
4. On edit, load the selected row into the form, send complete `PUT`, replace that ID in state from the response, then exit edit mode and show success feedback.
5. On delete, open a confirmation that names the book; only after confirmation send `DELETE`, remove the ID from state, and show success feedback.
6. On any network or server failure, retain usable form/list state and show a general message; map field validation errors to the associated control.

Use stable database IDs as React list keys. Keep server state separate from transient form state so a failed mutation does not silently overwrite user input. Avoid a global state library and caching framework for this MVP; a single feature hook and a small fetch wrapper make loading and mutation transitions easy to test.

Accessibility is an architectural contract, not a styling afterthought: every control has a visible label, required fields are programmatically marked, errors use `aria-describedby`/stable IDs, status messages use an appropriate live region, and pending mutation controls cannot be double-submitted.

## Docker Topology

Recommended `compose.yaml` services:

```text
frontend: build ./frontend, serve dist with nginx, publish 8080:80
api:      build ./backend, run uvicorn, publish 8000:8000
volume:   book_tracker_data -> /data (api only)
network:  default Compose bridge network
```

The frontend's browser JavaScript cannot use `http://api:8000`; Docker service DNS is for container-to-container traffic, while the browser runs on the host. Configure the browser API base as `http://localhost:8000` (or expose an Nginx `/api` reverse proxy later and use same-origin requests). Explicitly allow the development frontend origin in FastAPI CORS; do not use a permissive wildcard as the default.

The API should bind to `0.0.0.0:8000`, include a health check, and mount the database directory from a named volume. Do not publish SQLite as a separate service and do not mount the database into the frontend. Compose's default bridge network and service names are sufficient; add a custom network only if isolation is needed. `depends_on` controls startup ordering but is not readiness, so the smoke test should poll `/api/v1/health` before exercising CRUD.

For a development profile, bind mounts and Vite hot reload are useful; for the reproducible delivery path, use immutable image builds and serve the built frontend. Document the distinction so local convenience does not become the production-like acceptance path.

## Patterns to Follow

### Contract-first vertical slice
Define the book schema, endpoint behavior, and validation examples before splitting frontend and backend work. The first executable slice should be `POST → SQLite commit → GET → React render`, then expand to update/delete/search. FastAPI's typed request models provide validation and generated OpenAPI documentation; the UI still mirrors the rules for immediate feedback.

### Explicit state transitions
Represent `idle/loading/success/error` and mutation pending state explicitly. This prevents the common bug where an empty array is indistinguishable from “not loaded” or “no search matches.”

### Server-authoritative responses
After mutations, use the returned resource (or refetch the deterministic collection) rather than assuming the client representation is canonical. This keeps IDs, normalization, and persisted values aligned.

## Anti-Patterns to Avoid

### Shared database access from the browser
Never expose the SQLite file or let React encode SQL/data rules. It breaks the API boundary, makes validation inconsistent, and prevents meaningful integration tests.

### Route handlers as a god layer
Do not combine parsing, validation, SQL construction, commits, and response formatting in each endpoint. It creates inconsistent transaction behavior and makes the isolated test database difficult to inject.

### Client-only validation
Client validation is feedback, not enforcement. Every mutation must pass the same semantic checks at the API boundary because HTTP clients and tests can bypass React.

### Unstable list ordering or optimistic destructive updates
Do not rely on database insertion order, and do not remove a book from UI state before a confirmed delete response. Both produce confusing behavior after reload or failure.

## Dependency-Driven Build Order

1. **Repository and runtime skeleton** — pin Python/Node versions, establish backend/frontend directories, Compose, environment variables, and health endpoint. Everything else needs a reproducible process boundary.
2. **Database/session and book schema** — create the table, session dependency, bootstrap/migration approach, and isolated test database. The API cannot be verified without persistence.
3. **API contract and CRUD routes** — implement Pydantic schemas, normalization/validation, repository/service layers, deterministic list, status/error handling, CORS, and API tests. This is the stable dependency for the UI.
4. **React shell and API client** — establish typed client functions, collection state, loading/error/empty/no-match states, and list rendering against the real API.
5. **Create/edit form and mutation feedback** — add accessible controlled fields, client-mirrored validation, pending guards, create/update flows, and success/error mapping.
6. **Confirmed delete and search** — add named confirmation, delete flow, query behavior, and deterministic refresh/reconciliation. These depend on working list state and resource IDs.
7. **Container integration and delivery checks** — build production-like images, named volume persistence, health/readiness behavior, CORS from the actual frontend origin, and browser/API/container smoke tests from a clean checkout.

This order deliberately reaches a backend-only vertical slice before visual polish, then connects the real UI before adding search and destructive actions. It minimizes mock drift and makes each phase independently verifiable.

## Scalability Considerations

| Concern | MVP / 100 users | 10K users | 1M users |
|---------|------------------|-----------|----------|
| Database | SQLite file with named volume, one API process, committed transactions | Move to PostgreSQL; add migrations, backups, and connection management | Managed relational DB, replicas, observability, partitioning/search strategy as needed |
| Search | Case-insensitive `LIKE` over title/author | Indexed normalized columns or database full-text search | Dedicated search service only after measured need |
| API | Single FastAPI container | Multiple stateless API replicas behind a proxy; external DB | Horizontally scaled services with cache/queue only for proven workloads |
| Frontend | Static Nginx image | CDN/static hosting | CDN and asset pipeline |

Do not design the MVP around the 1M-user topology. Preserve the migration path by keeping SQLite behind a repository/session boundary and avoiding SQLite-specific behavior in route or UI code.

## Sources

- FastAPI, “Request Body” — typed Pydantic request models, validation, JSON Schema/OpenAPI: https://fastapi.tiangolo.com/tutorial/body/ (HIGH)
- FastAPI, “CORS (Cross-Origin Resource Sharing)” — explicit origins and preflight behavior: https://fastapi.tiangolo.com/tutorial/cors/ (HIGH)
- SQLAlchemy 2.0, “SQLite” — SQLite transaction behavior, connection configuration, and file-database considerations: https://docs.sqlalchemy.org/en/20/dialects/sqlite.html (HIGH; current documentation retrieved 2026-08-13)
- Docker Docs, “Networking in Compose” — service-name DNS, default bridge network, and host/container port distinction: https://docs.docker.com/compose/how-tos/networking/ (HIGH)
- React Docs, “Quick Start” — component boundaries, list keys, event/state flow, and lifting shared state: https://react.dev/learn (HIGH)
- MDN, “HTTP request methods” — GET/POST/PUT/DELETE semantics and idempotency: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods (HIGH)
