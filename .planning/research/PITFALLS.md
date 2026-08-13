# Domain Pitfalls

**Domain:** Full-stack book database (FastAPI + SQLite + React + Docker)
**Researched:** 2026-08-13
**Overall confidence:** HIGH for framework/database/container mechanics; MEDIUM for project-specific failure frequency

This project is deliberately small, but the failure modes are mostly integration failures rather than algorithmic ones. Prevent them with a vertical slice: define the book contract, persist one record, read it back through the API, render it in React, then exercise the same path in tests and Docker before adding polish.

## Critical Pitfalls

### 1. Validation exists in the form but not at the API/database boundary

**What goes wrong:** The React form permits blank or malformed values, or the API accepts arbitrary strings because the endpoint receives a `dict`. Invalid years, whitespace-only titles, malformed ISBNs, and accidental `null` values enter SQLite. Later edits may bypass the original form entirely.

**Warning signs / early detection:**
- OpenAPI shows an unstructured request body or fields are optional when the product says they are required.
- `POST` and `PUT` accept `"   "`, years outside the intended range, or an ISBN with an undefined normalization rule.
- A direct `curl` request can create data the UI would reject.
- Tests cover only a happy-path form submission.

**Prevention:**
- Define separate Pydantic request and response models. Make title, author, publication year, genre, and ISBN required; add length constraints and an explicit year range (for example, `1..current year` if historical dates are allowed).
- Normalize at the boundary (trim text; choose and document whether ISBN hyphens/spaces are stored normalized), then validate the normalized value. Do not rely on HTML `required` or input types as security/data integrity controls.
- Add database `NOT NULL` constraints and any deliberately chosen uniqueness constraint (usually normalized ISBN), while translating constraint errors to stable `409`/`422` responses.
- Test invalid JSON, missing fields, blank strings, boundary years, duplicate ISBNs, and update validation directly against the API.

**Phase mapping:** **Phase 1 — Domain model and persistence/API contract.** Establish the schema and validation before CRUD endpoints or UI forms. Re-check in **Phase 3 — Frontend/API integration** with browser-visible error states.

FastAPI uses Pydantic models to parse, validate, and expose JSON Schema/OpenAPI for request bodies; bypassing that boundary discards the framework's principal safety mechanism. Source: [FastAPI request bodies](https://fastapi.tiangolo.com/tutorial/body/) (HIGH).

### 2. SQLite writes appear successful but disappear after restart

**What goes wrong:** The app writes to a relative path in a container's writable layer, forgets to commit, or mounts a volume at the wrong path. CRUD appears to work until the container is recreated, at which point the collection is empty—or the app is silently using a different database file than expected.

**Warning signs / early detection:**
- The database path is relative, undocumented, or differs between local and container commands.
- `docker compose down` followed by `up` loses a book; `docker compose down -v` is not distinguished from ordinary teardown.
- `docker inspect` does not show a mount covering the directory containing the SQLite file.
- A test only reads the response from `POST` and never opens a new connection or restarts the service.

**Prevention:**
- Configure one absolute, environment-controlled path such as `/data/books.db`; create its parent directory at startup and log the resolved path (without leaking secrets—there are none in this app, but keep the habit).
- Mount a named Docker volume specifically at `/data`. Treat deleting the volume as an explicit destructive operation and document backup/restore expectations.
- Commit every write transaction and close connections via a dependency/context manager. Add a restart persistence test: create → stop/recreate API container → list → assert the record remains.
- Add a startup schema check/migration step that fails loudly if the expected schema is absent; never silently create a second database in the working directory.

SQLite's Python documentation states that inserts open a transaction and require `commit()`, and Docker documents that container writable layers disappear while volumes persist beyond container lifecycle. Sources: [Python sqlite3 tutorial](https://docs.python.org/3/library/sqlite3.html) and [Docker volumes](https://docs.docker.com/engine/storage/volumes/) (HIGH).

**Phase mapping:** **Phase 1 — Domain model and persistence/API contract** for transaction/path conventions; **Phase 5 — Docker packaging and verification** for the named volume and restart test.

### 3. Search is implemented as unsafe string interpolation or has unspecified semantics

**What goes wrong:** A search term is concatenated into SQL, `%`/`_` behave as unexpected wildcards, matching is case-sensitive on one machine, or the endpoint returns an arbitrary order and an unbounded result set. Users report that searching for an ISBN, partial title, or author “doesn't work,” while the developer cannot reproduce the exact rule.

**Warning signs / early detection:**
- SQL contains f-strings/string concatenation around the query term.
- Search behavior for `ann`, `Ann`, `%`, `_`, apostrophes, and leading/trailing spaces is not specified by tests.
- The API scans every column in Python or returns all rows and filters in React.
- `EXPLAIN QUERY PLAN` shows a full scan once the seed data grows, or the endpoint has no limit/order contract.

**Prevention:**
- Use bound SQL parameters for every user value. Decide and document whether one `q` searches title/author/genre/ISBN and whether matching is substring, token, or exact.
- For a small MVP, use a single parameterized `LIKE` query with explicit escaping and `ESCAPE` handling if literal `%` and `_` should be searchable; normalize case consistently. Add a deterministic `ORDER BY` and a modest limit.
- Keep search in the API/database, not the browser. Test special characters, case, no-match, empty query, duplicate matches, and all searchable fields.
- Do not introduce FTS5 merely to solve a small-table problem. If the data set later requires full-text search, add it as a deliberate indexed feature and test index synchronization on insert/update/delete.

**Phase mapping:** **Phase 2 — CRUD and search behavior.** Define semantics before wiring the search box; include query-plan/performance checks only if realistic seed data makes them relevant.

SQLite's query planner documentation explains that unindexed predicates cause full table scans, while its FTS5 documentation makes clear that an external-content index must be kept synchronized or results become inconsistent. Sources: [SQLite query planning](https://www.sqlite.org/queryplanner.html) and [SQLite FTS5](https://www.sqlite.org/fts5.html) (HIGH).

### 4. CORS is “fixed” with a wildcard, but the browser still blocks CRUD requests

**What goes wrong:** The frontend runs at `http://localhost:5173` while the API allows only another port, or `GET` works but JSON `POST`/`PUT`/`DELETE` fails during the browser's `OPTIONS` preflight. Developers test with Swagger/curl and miss the browser-only policy.

**Warning signs / early detection:**
- Browser console reports a CORS error while the API logs no application request.
- DevTools shows a failed `OPTIONS` request or missing `Access-Control-Allow-Methods`/`Access-Control-Allow-Headers`.
- Allowed origins include `localhost` without the actual scheme/port, or use `*` without understanding credential implications.
- Frontend tests mock fetch exclusively and no real browser or preflight check exists.

**Prevention:**
- List the exact development and container-served origins, including scheme and port, through configuration; do not use `*` as a blanket fix.
- Configure `CORSMiddleware` for the actual methods and headers used by JSON CRUD, and add a test that sends an `Origin` header plus an `OPTIONS` preflight.
- Prefer same-origin production serving (reverse proxy/static frontend and API under one origin) when practical; retain explicit CORS for separated development servers.
- Treat CORS as browser access control, not API authorization. The project has no authentication, so do not imply that CORS protects the database.

**Phase mapping:** **Phase 3 — Frontend/API integration.** Validate from the real browser origin, not only API client tools; re-run in **Phase 5 — Docker packaging and verification** with the Compose URLs.

FastAPI documents that origin includes protocol, host, and port, and that preflight requests require matching allowed methods/headers. MDN notes that JavaScript receives only a generic failure and details are visible in the browser console. Sources: [FastAPI CORS](https://fastapi.tiangolo.com/tutorial/cors/) and [MDN CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS) (HIGH).

### 5. Tests use a different database lifecycle than production

**What goes wrong:** Tests share state, depend on test order, use `:memory:` with multiple connections and see different databases, or validate mocked repository calls rather than real SQL constraints and transactions. The suite is green while the Docker app fails on a fresh database.

**Warning signs / early detection:**
- Tests fail when run in isolation or in a different order.
- A fixture points at the development `.db` file, or test data remains after the suite.
- API tests assert status codes but never verify a subsequent `GET`, update, delete, or fresh connection.
- There is no test of schema initialization, constraint errors, CORS preflight, or frontend loading/error states.

**Prevention:**
- Inject the database dependency so tests use a temporary file (or one deliberately shared in-memory connection) and production uses the configured file; never use the development database in tests.
- Reset/rollback deterministically per test and seed through the same schema path used by the app. Include API contract tests for all CRUD status codes and a persistence-across-connection test.
- Use React Testing Library queries by labels/roles and test user-visible behavior: loading, empty, validation error, server error, successful create/edit/delete, and search results. Mock only the network boundary, not component internals.
- Run backend tests, frontend tests, lint/type checks, and a smoke test in CI; run the same commands inside the relevant Docker build where dependency drift is possible.

**Phase mapping:** **Phase 4 — Automated tests and verification**, started alongside Phase 1 so test seams exist before the implementation grows; **Phase 5** adds container smoke tests.

FastAPI's testing guidance uses `TestClient` with pytest and its database guidance emphasizes studying a dedicated SQL-database testing setup. React Testing Library explicitly recommends tests that resemble user behavior rather than implementation details. Sources: [FastAPI testing](https://fastapi.tiangolo.com/tutorial/testing/), [FastAPI database testing](https://fastapi.tiangolo.com/how-to/testing-database/), and [React Testing Library](https://testing-library.com/docs/react-testing-library/intro/) (HIGH).

### 6. Containers start in the wrong order or report “running” while unusable

**What goes wrong:** Compose starts the frontend/API process before its required initialization is complete, the server binds to `127.0.0.1` inside the container, or a healthcheck checks only that a process exists. A build succeeds but `docker compose up` produces a blank UI or connection failures.

**Warning signs / early detection:**
- `docker compose up` is flaky on a clean machine or needs manual restart.
- The container is “Up” but `curl` from another service cannot reach the API.
- The API command binds to localhost instead of `0.0.0.0`, or the frontend hard-codes a host-only API URL.
- No `/health` endpoint, healthcheck, service logs, or post-start smoke test exists.

**Prevention:**
- Bind the API to `0.0.0.0` and configure the frontend API base URL for the browser's reachable origin (remember: `localhost` in browser JavaScript means the user's host, not the Compose service name).
- Add a cheap `/health` endpoint that verifies app readiness and, if appropriate, can open/query SQLite. Add Docker `healthcheck` with a bounded interval/start period/retries.
- Use `depends_on: condition: service_healthy` only for actual service readiness; it does not replace application retry/error handling. For a single-container SQLite app, initialize schema idempotently before serving traffic.
- Make the acceptance command explicit: build, start clean, wait for health, create a book, list/search/edit/delete it, then restart and verify persistence.

**Phase mapping:** **Phase 5 — Docker packaging and verification.** Prove clean-start, health, inter-service networking, and restart behavior as one release gate.

Compose documentation explicitly says startup ordering waits for a container to be running, not ready, unless a healthcheck and `service_healthy` condition are used. Source: [Docker Compose startup order](https://docs.docker.com/compose/how-tos/startup-order/) (HIGH).

## Moderate Pitfalls

### 7. API and UI contracts drift

**What goes wrong:** The backend returns `publication_year` while the form submits `year`, IDs are strings in one layer and numbers in another, or a delete returns `204` while the frontend tries to parse JSON. The UI appears broken even though each layer works in isolation.

**Warning signs / early detection:** Network responses do not match TypeScript types; browser console shows `undefined` fields; successful mutations leave stale rows visible.

**Prevention:** Treat the OpenAPI schema and explicit TypeScript API types as a contract. Centralize fetch/error handling, check `response.ok`, handle `204` without parsing, and invalidate/refetch or update state after every mutation. Add one end-to-end CRUD contract test.

**Phase mapping:** **Phase 3 — Frontend/API integration**, before visual polish.

### 8. Database connection/transaction handling is unsafe for concurrent requests

**What goes wrong:** A global SQLite connection is shared across request threads, a request holds a write transaction while doing unrelated work, or exceptions leave transactions open. Under rapid edits, the app returns `database is locked` or commits partial work.

**Warning signs / early detection:** Intermittent lock errors, `check_same_thread` exceptions, tests that pass serially but fail under parallel requests, or connections that are never closed.

**Prevention:** Use a per-request connection dependency (or a carefully designed connection strategy), keep transactions short, commit only after successful writes, rollback on exceptions, and translate expected SQLite errors. Keep synchronous SQLite access synchronous unless the chosen async layer is understood; async endpoints do not make blocking database calls non-blocking.

**Phase mapping:** **Phase 1 — Persistence/API contract**; add a small concurrent-write smoke test in **Phase 4**.

Python's sqlite3 documentation notes both commit/rollback behavior and thread restrictions on connections. Source: [Python sqlite3 connection reference](https://docs.python.org/3/library/sqlite3.html) (HIGH).

### 9. Delete/edit UX allows stale or destructive actions

**What goes wrong:** A user double-submits a form, deletes a row without confirmation, edits an item that was already deleted, or sees no feedback while a request is pending. The database is correct but the visible list is misleading.

**Warning signs / early detection:** Duplicate books after double-click, buttons remain enabled during mutation, errors disappear, or the list only becomes correct after a full refresh.

**Prevention:** Disable or make mutation controls idempotent while pending, show success/error feedback, confirm delete, and handle `404`/`409` by refetching or clearly explaining the conflict. Test rapid clicks and failed network requests.

**Phase mapping:** **Phase 3 — Frontend/API integration**; verify in **Phase 4** behavioral tests.

## Minor Pitfalls

### 10. Seed data and schema initialization are not idempotent

**What goes wrong:** Restarting the API attempts to recreate an existing table or duplicates demo books. A developer “fixes” a schema error by deleting the database, masking an incompatible schema change.

**Warning signs / early detection:** Startup fails on the second run, duplicate seed records appear, or tests require manual database deletion.

**Prevention:** Use `CREATE TABLE IF NOT EXISTS` only for the initial simple schema, make seed inserts explicit and idempotent, and record schema changes as migrations rather than destructive startup code. Add a two-start test.

**Phase mapping:** **Phase 1** for initial schema; **Phase 5** for clean/repeated container startup.

### 11. Error handling exposes internals or collapses all failures into “Network error”

**What goes wrong:** SQLite tracebacks leak to the client, while the React app displays the same message for validation, duplicate ISBN, server failure, and actual network outage.

**Warning signs / early detection:** API responses contain SQL/table paths; frontend cannot distinguish `422`, `404`, `409`, and `500`; logs lack request context.

**Prevention:** Define stable error response shapes, map expected exceptions explicitly, keep detailed traces in server logs, and provide field-level validation plus a general retry/error state in React. Test each expected status.

**Phase mapping:** **Phase 2 — CRUD error contract**, then **Phase 3** for UI presentation.

### 12. Development-only configuration is accidentally treated as production behavior

**What goes wrong:** Hot-reload, permissive CORS, debug stack traces, hard-coded API URLs, or a bind-mounted source tree leaks into the “reproducible Docker” path.

**Warning signs / early detection:** The production-like Compose run depends on files outside the image, uses a dev server, or has no documented environment variables.

**Prevention:** Separate development and production Compose profiles/configuration, build a static React asset image for the release path, configure values through environment variables, and run a clean checkout smoke test without local caches.

**Phase mapping:** **Phase 5 — Docker packaging and verification**.

## Phase-Specific Warnings

| Phase topic | Likely pitfall | Mitigation |
|---|---|---|
| Phase 1 — Domain model, schema, API foundation | Validation/DB constraints drift; writes are not committed | Write request/response models, schema constraints, connection dependency, rollback/commit tests first |
| Phase 2 — CRUD and search | Unsafe SQL, ambiguous search, incorrect status codes | Parameterized queries, explicit search contract, deterministic ordering, direct API edge-case tests |
| Phase 3 — React/API integration | CORS, API URL, stale mutation state, contract mismatch | Browser-origin preflight check, centralized client, typed responses, loading/error/empty states |
| Phase 4 — Automated tests | Shared DB state and implementation-detail tests | Isolated temporary DB, CRUD round trips, user-visible RTL tests, CI commands |
| Phase 5 — Docker and release verification | Lost SQLite file, bad bind address, startup race | Named volume, `/health`, `0.0.0.0`, clean Compose smoke test, stop/recreate persistence test |

## Sources

- [FastAPI request bodies and Pydantic validation](https://fastapi.tiangolo.com/tutorial/body/) — HIGH
- [FastAPI CORS middleware](https://fastapi.tiangolo.com/tutorial/cors/) — HIGH
- [FastAPI testing](https://fastapi.tiangolo.com/tutorial/testing/) — HIGH
- [Python `sqlite3` documentation](https://docs.python.org/3/library/sqlite3.html) — HIGH
- [SQLite query planner](https://www.sqlite.org/queryplanner.html) — HIGH
- [SQLite FTS5](https://www.sqlite.org/fts5.html) — HIGH
- [Docker Compose startup order](https://docs.docker.com/compose/how-tos/startup-order/) — HIGH
- [Docker volumes](https://docs.docker.com/engine/storage/volumes/) — HIGH
- [MDN CORS guide](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS) — HIGH
- [React Testing Library introduction](https://testing-library.com/docs/react-testing-library/intro/) — HIGH (page last updated 2024-06-03; principles remain applicable)
