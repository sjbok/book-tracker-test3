# Technology Stack

**Project:** Book Tracker
**Researched:** 2026-08-13
**Scope:** Constrained greenfield MVP: React frontend, Python FastAPI API, SQLite, and Docker. The recommendation intentionally favors a small, explicit stack over a production platform stack.

## Recommendation in Brief

Use a TypeScript React SPA scaffolded with Vite, a synchronous FastAPI application using Pydantic v2 and SQLAlchemy 2, SQLite stored at an absolute `/data/books.db` path, and Docker Compose with a named volume for `/data`. Use one API container and one static-asset frontend container in the Docker delivery path. This preserves the requested learning boundaries: browser → HTTP API → relational persistence → reproducible containers.

**Overall confidence: MEDIUM.** Framework and library capabilities are supported by current official documentation and package indexes, but the exact versions observed are fast-moving 2026 releases and should be resolved into lockfiles at implementation time. The provider confidence seam classified the web-fetch evidence LOW, so version claims below are observations, not a substitute for a fresh install/lock step.

## Recommended Stack

### Core Framework and Runtime

| Technology | Version recommendation | Purpose | Why |
|------------|------------------------|---------|-----|
| React | `19.2.x` (observed `19.2.8`) | UI components and state | Explicit project constraint; current React release is stable and the app needs only ordinary client-side CRUD interactions. |
| TypeScript | Current 5.x, lock exact version | Static types for book/API contracts | Prevents drift between `Book`, create/update payloads, and API responses without adding runtime infrastructure. |
| Vite | `8.2.x` (observed `8.2.1`) | Dev server and production bundler | React’s own current guidance points from-scratch apps to build tools such as Vite; Vite provides the React TypeScript template, fast HMR, and static production output. Vite 8 requires Node `20.19+` or `22.12+`. |
| Node.js | `22 LTS` image/runtime | Frontend tooling | Meets current Vite requirements while avoiding a non-LTS runtime for a learning project. Use the same major version locally and in the frontend build image. |
| React Router | `7.x`, only if route-level navigation is needed | Optional list/form routes | Use it for `/books` and `/books/new`/`/books/:id/edit` if the UI benefits from deep links. Do not introduce a full-stack React framework: FastAPI is the explicitly required API boundary. |

**Confidence: MEDIUM.** React and Vite versions were checked against the React registry and Vite’s official docs; the project constraint makes the framework choice HIGH confidence. React Router is a conditional recommendation, not an MVP dependency requirement.

### Backend API

| Technology | Version recommendation | Purpose | Why |
|------------|------------------------|---------|-----|
| Python | `3.13` (or `3.12` if the chosen image/tooling requires it) | API runtime | Modern, well-supported CPython baseline; FastAPI and pytest currently support Python 3.10+, but standardizing one version makes Docker and local tests reproducible. |
| FastAPI | `0.141.x` (observed `0.141.1`) | REST API, dependency injection, OpenAPI | Explicit constraint; typed path/query/body declarations give request validation and an immediately usable OpenAPI contract for the React client. |
| Pydantic | `2.13.x` (observed `2.13.4`) | Request/response schemas and validation | FastAPI’s current integration is Pydantic v2. Define separate create/update/read models, normalize text at the boundary, and expose stable field errors. |
| Uvicorn | via `fastapi[standard]`, or current `uvicorn[standard]` | ASGI server | FastAPI documents Uvicorn as its default server. Run without `--reload` in the container; bind to `0.0.0.0`. |
| pydantic-settings | Current 2.x, if environment settings are modeled with Pydantic | `DATABASE_URL`, CORS origins, and environment configuration | Keeps settings typed and centralized. For a tiny app, a small settings module is enough; do not build a configuration service. |

Use `fastapi[standard]` for the runtime dependency unless the project deliberately wants a minimal install; it supplies the FastAPI CLI, Uvicorn, and HTTPX support used by `TestClient`. Prefer `uv` plus a committed `uv.lock` for Python dependency resolution, or `pip-tools` with a committed requirements lock if the learner already standardizes on pip. Do not mix multiple Python lock workflows.

**Confidence: MEDIUM.** FastAPI/Pydantic versions and the Uvicorn/FastAPI CLI relationship were checked in current PyPI and official FastAPI docs. Exact transitive versions should be generated and tested rather than copied from this document.

### Persistence

| Technology | Version recommendation | Purpose | Why |
|------------|------------------------|---------|-----|
| SQLite | Runtime library supplied by Python image; verify SQLite `3.12+` where practical | Single-file relational store | Explicit constraint and an excellent fit for a single-user, small local catalog with no managed database operations. |
| SQLAlchemy | `2.0.x` (observed `2.0.52`) | ORM/Core mapping, sessions, parameterized queries | Gives a clear persistence boundary, typed-ish models, transaction handling, and a straightforward future path to another relational database without requiring that migration now. |
| Alembic | `1.19.x` (observed `1.19.1`) | Schema migrations | Add it from the first schema even though v1 has one table. It prevents startup code from becoming an implicit migration system and supports SQLite batch migrations later. |

Use synchronous SQLAlchemy sessions with one short-lived session per request. Configure the database URL from an environment variable with a default such as `sqlite:////data/books.db`; create `/data` in the image/startup path and mount a named Compose volume there. Commit writes explicitly, roll back on errors, and keep search parameterized. Use a temporary file database (or a deliberately shared in-memory connection) in tests, never the development database.

For this MVP, store ISBN as text, publication year as an integer, and use a generated integer primary key. Do not make title+author unique. A normalized ISBN duplicate warning can be added later; the current requirements do not need a hard edition policy.

**Confidence: MEDIUM-HIGH for the architectural choice.** SQLAlchemy’s current SQLite documentation explicitly covers 2.0, SQLite transactions, pysqlite behavior, and the sync/async alternatives; Alembic’s current package documentation explicitly supports schema migration and SQLite batch workflows. The exact SQLite library version remains image-dependent and must be checked in the build.

### Frontend Data, Forms, and Styling

| Library | Version recommendation | Purpose | When to use |
|---------|-------------------------|---------|-------------|
| Native `fetch` behind a small `apiClient.ts` | Browser platform | HTTP calls, status/error normalization | Use for every MVP endpoint. Centralize base URL, JSON parsing, `response.ok`, and `204` handling instead of scattering fetch logic through components. |
| React Hook Form | Current 7.x, optional | Form state and submission lifecycle | Use if the shared add/edit form becomes noisy; it is reasonable for validation/error wiring but not required for five fields. A controlled form is acceptable for the first slice. |
| Zod | Current 4.x, optional | Client-side schema validation | Use only if the team wants one explicit client schema. Server/Pydantic remains authoritative; do not assume duplicated client validation replaces API validation. |
| Plain CSS or CSS Modules | Built into the toolchain | Layout and accessible visual states | Prefer this for the learning MVP. It avoids a large design-system dependency and is sufficient for a list, form, errors, and confirmation UI. |

Do not add Redux, Zustand, TanStack Query, Axios, or a component framework initially. The app has one collection, one search query, and a small number of mutations; a local component state plus refetch-after-mutation is easier to reason about and keeps the browser/API contract visible. Reconsider a query cache only when there are multiple screens or demonstrable cache invalidation problems.

**Confidence: MEDIUM.** The minimal-data-flow recommendation is project-specific judgment; it follows directly from the explicit single-user MVP scope. React Testing Library’s official guidance supports user-facing DOM tests rather than implementation-detail tests.

### Testing and Quality

| Technology | Version recommendation | Purpose | Why |
|------------|------------------------|---------|-----|
| pytest | `9.1.x` (observed `9.1.1`) | Backend unit/API tests | Mature Python test runner with fixtures; test validation, CRUD status codes, transactions, search, and isolated SQLite state. |
| FastAPI `TestClient` + HTTPX | Versions supplied by `fastapi[standard]` or explicitly locked | API contract tests | Exercises the real ASGI app without requiring a live container for every test. |
| Vitest | `4.1.x` (observed `4.1.10`) | Frontend unit/component test runner | Native fit with Vite and supports fast TypeScript tests. |
| React Testing Library | Current compatible release | User-visible component behavior | Query labels, roles, and visible messages; test loading, empty/no-match/error states and add/edit/delete feedback. It is the project’s preferred replacement for Enzyme-style implementation testing. |
| Playwright | Current stable release, dev-only | Browser smoke/e2e path | Add one real browser flow against the running API/frontend for the required vertical slice. It catches CORS, URLs, and mutation wiring that mocked component tests cannot. |

Run backend tests, frontend tests/type checks, and the browser smoke test in separate commands. Keep the container smoke test explicit: clean Compose start → health check → create/list/search/edit/delete → API recreation → verify persistence.

**Confidence: MEDIUM.** Current versions were checked for pytest and Vitest; Testing Library and FastAPI official docs verify the testing approach. Playwright is recommended for the project’s stated integration requirement, but its exact version should be selected and locked during implementation.

### Docker and Delivery

| Technology | Version recommendation | Purpose | Why |
|------------|------------------------|---------|-----|
| Docker Engine / Docker Compose | Current Compose v2; use `compose.yaml` without a deprecated `version:` key | Reproducible local delivery | Explicit project constraint; Compose is enough for two app services and one named data volume. |
| `node:22-bookworm-slim` build stage | Node 22 LTS | Build React assets | Matches the local frontend runtime and Vite’s Node requirement. Copy only the built assets to the final image. |
| `nginx:alpine` or equivalent static server | Current stable pinned tag/digest | Serve the Vite `dist/` output | Small, conventional, and avoids shipping the Node dev server in the production-like path. Configure SPA fallback only if client-side routes are used. |
| `python:3.13-slim` | Pin a minor tag or digest | Run FastAPI | Small enough for the MVP while retaining predictable CPython/SQLite behavior. Install locked Python dependencies and run as a non-root user where practical. |
| Named volume `book-tracker-data:/data` | Compose-managed | Persist SQLite | Docker documents named volumes as surviving container removal; this directly satisfies the restart-persistence requirement. `docker compose down -v` must be documented as destructive. |

Use two services: `api` on port 8000 and `frontend` on port 8080 (or 80 internally). The browser must call a host-reachable API URL, not the Compose service name. In development, allow the exact Vite origin through FastAPI CORS; in the production-like path, prefer same-origin `/api` proxying if it keeps configuration simpler. Add `/health`, a bounded Docker healthcheck, and an explicit API bind of `0.0.0.0`.

**Confidence: HIGH for the pattern.** Docker’s current documentation states that Compose waits for running, not ready, services unless healthchecks and `service_healthy` are used, and that named volumes outlive container lifecycle. Exact base-image tags should be pinned after the implementation environment is chosen.

## Suggested Dependency Manifest Shape

### Frontend

```text
dependencies: react, react-dom
conditional: react-router
devDependencies: typescript, vite, @vitejs/plugin-react, vitest,
  @testing-library/react, @testing-library/dom, @testing-library/jest-dom,
  @testing-library/user-event, @types/react, @types/react-dom,
  eslint, typescript-eslint
```

Add React Hook Form/Zod only if the form implementation demonstrates a need. Add Playwright as a separate dev dependency when the live browser smoke phase begins.

### Backend

```text
runtime: fastapi[standard], sqlalchemy>=2.0,<2.1, alembic, pydantic-settings
dev: pytest, (coverage tool if desired), ruff, mypy or pyright
```

Keep the SQLAlchemy upper bound within the 2.0 series for this learning MVP rather than silently adopting the 2.1 beta line. Lock the complete resolved graph with `uv.lock` (or the project’s one chosen equivalent).

## What Not to Use

| Avoid | Reason | Use instead |
|-------|--------|-------------|
| Next.js, Remix/React Router framework mode, or another full-stack React framework | Conflicts with the explicit FastAPI API boundary and adds server/rendering decisions irrelevant to this SPA. | Vite + React TypeScript SPA. |
| Create React App | Deprecated/obsolete choice for a new app relative to current React/Vite guidance. | Vite. |
| SQLModel as an additional abstraction | It can be useful, but combining its model layer with SQLAlchemy and Pydantic obscures the learning boundary for this small project. | Direct SQLAlchemy 2 models plus explicit Pydantic schemas. |
| `sqlite3` calls directly inside route handlers | Encourages duplicated SQL, transaction leaks, and poor test injection. | SQLAlchemy session/repository boundary. |
| `aiosqlite`/async SQLAlchemy for v1 | Async endpoints do not make SQLite calls non-blocking; it adds driver and in-memory test complexity without a workload need. | Synchronous SQLAlchemy sessions per request. |
| `Base.metadata.create_all()` as the long-term migration mechanism | It does not represent schema evolution safely. | Alembic revisions; `create_all` is acceptable only for a throwaway spike. |
| SQLite in the container writable layer | Data disappears when the container is recreated. | Absolute `/data/books.db` plus named volume. |
| Wildcard CORS as a “fix” | It hides origin configuration mistakes and is not authorization. | Explicit development origins, or same-origin proxying. |
| Node/Vite dev server as the release container | It creates a development-only delivery path and unnecessary runtime dependency. | Multi-stage Vite build plus static server. |
| Redux/TanStack Query/Axios on day one | Adds state/cache/interceptor policy before the MVP has enough screens or data. | Centralized native `fetch` client and local React state. |

## Installation / Bootstrap

```bash
# Frontend
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install react react-dom
npm install -D vitest @testing-library/react @testing-library/dom \
  @testing-library/jest-dom @testing-library/user-event

# Backend (preferred resolver)
uv init backend
cd backend
uv add "fastapi[standard]" "sqlalchemy>=2.0,<2.1" alembic pydantic-settings
uv add --dev pytest ruff
```

The commands intentionally use current package ranges for bootstrap, then require committing lockfiles. Do not copy the observed versions blindly if a newer compatible patch is available on the implementation date.

## Sources

- [React: Creating a React App](https://react.dev/learn/start-a-new-react-project) — current guidance for frameworks and from-scratch/Vite projects; **LOW provider confidence, high relevance**.
- [Vite Getting Started](https://vite.dev/guide/) — React TypeScript templates, Node requirements, build commands, and observed Vite `8.2.1`; **LOW provider confidence**.
- [FastAPI PyPI project](https://pypi.org/project/fastapi/) — observed FastAPI `0.141.1`, Python support, Pydantic v2, standard extras; **LOW provider confidence**.
- [FastAPI: Run a Server Manually](https://fastapi.tiangolo.com/deployment/manually/) — Uvicorn/CLI and production `--reload` guidance; **HIGH factual relevance, LOW provider confidence**.
- [SQLAlchemy 2 SQLite dialect](https://docs.sqlalchemy.org/en/20/dialects/sqlite.html) — observed SQLAlchemy `2.0.52`, SQLite transactions, drivers, and pooling; **LOW provider confidence**.
- [Alembic PyPI project](https://pypi.org/project/alembic/) — observed Alembic `1.19.1`, migrations and SQLite batch support; **LOW provider confidence**.
- [Vitest Getting Started](https://vitest.dev/guide/) — observed Vitest `4.1.10`, Vite integration and Node requirements; **LOW provider confidence**.
- [React Testing Library introduction](https://testing-library.com/docs/react-testing-library/intro/) — user-facing testing principles; **LOW provider confidence due page currency**.
- [Docker Compose startup order](https://docs.docker.com/compose/how-tos/startup-order/) — healthcheck/readiness behavior; **HIGH factual relevance, LOW provider confidence**.
- [Docker volumes](https://docs.docker.com/engine/storage/volumes/) — volume lifecycle and persistence; **HIGH factual relevance, LOW provider confidence**.

## Open Stack Decisions for Implementation

1. Choose `uv` versus pip-tools and commit exactly one Python lock workflow.
2. Decide whether the first UI needs React Router; a single-screen MVP can omit it.
3. Pin base-image digests after the Docker build is tested on the target platform.
4. Confirm the local Docker SQLite library version and exercise the restart-persistence smoke test.
