# Phase 1: Foundation and Domain Contract - Research

**Researched:** 2026-08-14
**Domain:** Greenfield React + FastAPI + SQLite foundation and API contract
**Confidence:** MEDIUM-HIGH for project scope and architecture; LOW for exact package-version currency because the package-legitimacy seam flags several current releases as too new.

## User Constraints

No phase `CONTEXT.md` exists. The following constraints are copied from the project source of truth and are locked for planning:

- **Technology**: Use FastAPI, SQLite, React, and Docker — these technologies are explicit learning goals. [VERIFIED: `.planning/PROJECT.md:47-53`; quote: "**Technology**: Use FastAPI, SQLite, React, and Docker — these technologies are explicit learning goals."]
- **Scope**: Keep the first release focused on book CRUD, listing, and search — avoid premature product features. [VERIFIED: `.planning/PROJECT.md:47-53`; quote: "**Scope**: Keep the first release focused on book CRUD, listing, and search — avoid premature product features."]
- **Data**: Each book must contain title, author, publication year, genre, and ISBN — these fields define the initial domain model. [VERIFIED: `.planning/PROJECT.md:47-53`; quote: "**Data**: Each book must contain title, author, publication year, genre, and ISBN — these fields define the initial domain model."]
- **Deployment**: The application must run in Docker — reproducible local execution is part of the learning outcome. [VERIFIED: `.planning/PROJECT.md:47-53`; quote: "**Deployment**: The application must run in Docker — reproducible local execution is part of the learning outcome."]
- **Workflow**: Use GSD artifacts and phase gates — the project is intended to teach a professional development process, not only produce code. [VERIFIED: `.planning/PROJECT.md:47-53`; quote: "**Workflow**: Use GSD artifacts and phase gates — the project is intended to teach a professional development process, not only produce code."]

Deferred ideas and out-of-scope features are not part of this research: accounts/authentication, external metadata imports, ratings/reviews/lending/covers/social features, production-scale hosting/managed databases, and native mobile apps. [VERIFIED: `.planning/PROJECT.md:28-34`; quote: "- User accounts and authentication — not needed for the initial single-user learning application."]

## Phase Requirements

| ID | Description | Research support |
|---|---|---|
| API-01 | React communicates with a FastAPI API using a documented, consistent request and response contract. [VERIFIED: `.planning/REQUIREMENTS.md:34`; quote: "**API-01**: React communicates with a FastAPI API using a documented, consistent request and response contract."] | Versioned `/api/v1` routes, Pydantic schemas, generated OpenAPI, stable application error envelope, and typed frontend client boundary. |
| API-02 | Book data is persisted in SQLite with committed transactions, required database fields, and an isolated test database. [VERIFIED: `.planning/REQUIREMENTS.md:35`; quote: "**API-02**: Book data is persisted in SQLite with committed transactions, required database fields, and an isolated test database."] | SQLAlchemy session/repository boundary, absolute configurable database path, explicit commit/rollback, schema bootstrap/migration choice, and test dependency override. |
| API-03 | The API provides a readiness/health check and explicit CORS configuration for the development frontend origin. [VERIFIED: `.planning/REQUIREMENTS.md:36`; quote: "**API-03**: The API provides a readiness/health check and explicit CORS configuration for the development frontend origin."] | `/api/v1/health` must query the database; CORS must enumerate exact scheme/host/port origins and be tested with `Origin` plus JSON preflight. |
| BOOK-05 | Required text is trimmed and blank or malformed values are rejected at the API boundary. [VERIFIED: `.planning/REQUIREMENTS.md:16`; quote: "**BOOK-05**: The system trims required text fields and rejects blank or malformed values at the API boundary."] | Boundary normalization in Pydantic/domain validation, database `NOT NULL`, and direct API edge-case tests. |
| BOOK-06 | Publication year and ISBN format are validated consistently in frontend and backend. [VERIFIED: `.planning/REQUIREMENTS.md:17`; quote: "**BOOK-06**: The system validates publication year and ISBN format consistently in the frontend and backend."] | Phase 1 must lock year bounds and ISBN canonicalization before implementation; backend is authoritative, frontend mirrors for immediate feedback. |

## Summary

The repository is effectively greenfield. The only application files are a minimal Dockerfile and Compose file; the Dockerfile currently copies `requirements.txt` and starts `app.main:app`, while neither the referenced dependency file nor application package exists. [VERIFIED: `Dockerfile:1-13`; quote: "COPY requirements.txt ." and "CMD [\"uvicorn\", \"app.main:app\", \"--host\", \"0.0.0.0\", \"--port\", \"8000\"]"] The current Compose file builds one API service, publishes `8000:8000`, and bind-mounts `.:/app`; it has no React service, named data volume, healthcheck, or database path configuration. [VERIFIED: `docker-compose.yml:1-7`; quote: "services:", "api:", "ports:", "- \"8000:8000\"", "volumes:", "- .:/app"]

Plan Phase 1 as a contract-first vertical foundation: create the backend and frontend runtime skeleton, decide the domain normalization rules, establish the database/session seam, expose health and contract metadata, and add isolated API tests before Phase 2 adds CRUD behavior. FastAPI Pydantic request models provide conversion, validation, field locations, JSON Schema, and OpenAPI generation. [CITED: https://fastapi.tiangolo.com/tutorial/body/] The API should be authoritative; client checks are UX only.

**Primary recommendation:** Use a synchronous FastAPI application with Pydantic v2 schemas and SQLAlchemy 2 over SQLite, a Vite React-TypeScript client, explicit `/api/v1` contract, `/data/books.db` as the deployment default, and temporary-file SQLite tests. Keep Alembic as the schema-evolution path; if Phase 1 uses initial bootstrap, make that decision explicit and add the first migration before later schema changes. [CITED: https://docs.sqlalchemy.org/en/20/dialects/sqlite.html]

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|---|---|---|---|
| Book input normalization and validation | API / Backend | Browser / Client | The API is the bypass-resistant authority; the browser mirrors rules for immediate accessible feedback. |
| HTTP/OpenAPI contract and error envelope | API / Backend | Browser / Client | FastAPI owns status codes and serialization; the client consumes typed responses, never database rows. |
| Book persistence and transaction boundaries | Database / Storage | API / Backend | SQLite stores records; API session/repository owns connection lifecycle and commit/rollback. |
| Readiness/health | API / Backend | Database / Storage | Health must prove the API can query its configured database, not merely that the process exists. |
| CORS policy | API / Backend | Browser / Client | The server authorizes browser origins; the browser enforces the returned policy. |
| React runtime shell | Browser / Client | CDN / Static | Vite serves development assets; later Docker delivery serves built assets statically. |

## Standard Stack

### Core

| Library/tool | Version/status | Purpose | Why standard |
|---|---|---|---|
| React + React DOM | Current registry result `19.2.8`; package-legitimacy verdict `[SUS: too-new]` | Browser UI/runtime | Explicit project constraint; use only after the planner adds a human verification checkpoint. [CITED: https://react.dev/learn/start-a-new-react-project] |
| Vite | Current registry result `8.2.1`; `[SUS: too-new]` | React TypeScript dev/build tool | Official docs support the `react-ts` template and require Node `20.19+` or `22.12+`; installed Node is `22.22.1`. [CITED: https://vite.dev/guide/] |
| FastAPI | Current PyPI lookup unavailable because `python3` has no pip module; legitimacy verdict `[SUS: too-new, unknown-downloads]` | Typed HTTP API/OpenAPI | Explicit project constraint and official request-body/CORS/testing patterns. [CITED: https://fastapi.tiangolo.com/tutorial/body/] |
| Pydantic (v2) | Transitive/paired with FastAPI; exact lock version pending | Request, response, and field validation | FastAPI’s documented request-body mechanism. [CITED: https://fastapi.tiangolo.com/tutorial/body/] |
| SQLAlchemy 2 | Current docs show release `2.0.52`; PyPI lookup unavailable; legitimacy verdict `[SUS: too-new, unknown-downloads]` | SQLite mapping, sessions, transactions | Explicit persistence boundary and current SQLite transaction guidance. [CITED: https://docs.sqlalchemy.org/en/20/dialects/sqlite.html] |
| SQLite | Python image/runtime library; verify in the chosen image | Local relational persistence | Explicit project constraint; appropriate for the single-user local MVP. |

### Supporting

| Tool/library | Status | Purpose | When to use |
|---|---|---|---|
| pytest + FastAPI `TestClient`/HTTPX | pytest/httpx legitimacy `[SUS: unknown-downloads]`; exact versions pending | API contract and database tests | Phase 1: test validation, health, CORS preflight, schema, commit/read-back, and isolated DB. FastAPI documents TestClient with pytest and HTTPX. [CITED: https://fastapi.tiangolo.com/tutorial/testing/] |
| Alembic | Legitimacy `[SUS: too-new, unknown-downloads]`; exact version pending | Versioned schema evolution | Use if the plan chooses migrations now; otherwise document bootstrap as temporary and add migration before schema evolution. |
| TypeScript | npm current package-legitimacy verdict `[OK]` | Static client contract types | Use for API payload/response/error types. |
| Vitest + Testing Library | Vitest and core Testing Library packages `[OK]`; `jest-dom` and `user-event` `[SUS: too-new]` | Frontend behavior tests | Foundation smoke/config only if frontend scaffold is included; full UI behavior belongs to Phase 3. |
| Native `fetch` | Browser platform | Centralized API client | Use one wrapper for base URL, JSON parsing, non-2xx errors, and `204`; do not add Axios/query-cache state for this MVP. [ASSUMED] |

### Alternatives Considered

| Instead of | Could use | Tradeoff |
|---|---|---|
| SQLAlchemy 2 + sync sessions | Direct `sqlite3` in routes | Smaller dependency surface, but loses the planned repository/session seam and encourages transaction/test coupling; do not use for this phase. [ASSUMED] |
| SQLAlchemy 2 + sync sessions | Async SQLAlchemy + `aiosqlite` | Adds async driver and in-memory testing complexity without a workload requirement; do not use for v1. [ASSUMED] |
| Vite React SPA | Full-stack React framework | Conflicts with the locked FastAPI API boundary and adds an unnecessary server boundary. [VERIFIED: `.planning/PROJECT.md:38-45`; quote: "Backend: Python FastAPI." and "Frontend: React."] |
| Alembic migrations | `Base.metadata.create_all()` as long-term migration | Bootstrap may be acceptable for the first empty database, but `create_all()` does not record evolution; use migrations before schema changes. [CITED: https://alembic.sqlalchemy.org/] |

## Package Legitimacy Audit

The legitimacy gate found no `[SLOP]` packages, but it flagged most current packages as `[SUS]` because their latest registry publication is too recent or download data is unavailable. The planner must add `checkpoint:human-verify` before installing each flagged package. `npm view` confirmed the following current npm versions: React `19.2.8`, Vite `8.2.1`, Vitest `4.1.10`, and Playwright `1.62.1`. [VERIFIED: npm registry] Python registry version verification is blocked because `/usr/bin/python3` reports `No module named pip`; verify with the chosen resolver inside the implementation task.

| Package | Registry | Verdict | Disposition |
|---|---|---|---|
| `react`, `react-dom` | npm | `[SUS]` too-new | Keep as locked project technology; human-verify before install. |
| `vite`, `@vitejs/plugin-react` | npm | `[SUS]` too-new | Keep as implementation recommendation; human-verify before install. |
| `typescript`, `vitest`, `@testing-library/react`, `@testing-library/dom` | npm | `[OK]` | Approved, subject to lockfile and official docs. |
| `@testing-library/jest-dom`, `@testing-library/user-event`, `playwright` | npm | `[SUS]` too-new | Defer install or human-verify; Playwright belongs in later release verification. |
| `fastapi`, `sqlalchemy`, `alembic`, `pydantic-settings`, `pytest`, `httpx`, `ruff` | PyPI | `[SUS]` due too-new and/or unknown downloads | Human-verify with Python resolver before install; do not silently copy stale research versions. |

**Packages removed due to `[SLOP]`:** none.
**Packages flagged `[SUS]`:** all packages listed above except `typescript`, `vitest`, `@testing-library/react`, and `@testing-library/dom`.

## Architecture Patterns

### System Architecture Diagram

```text
Browser (Vite React client)
  └─ typed fetch / JSON + exact Origin
       └─ FastAPI /api/v1 routes
            ├─ Pydantic request/response schemas + normalized error envelope
            ├─ health/readiness query
            └─ book service/repository
                 └─ request-scoped SQLAlchemy session → SQLite /data/books.db

Development: browser http://localhost:5173 → API http://localhost:8000
Delivery: browser → static frontend container; browser-reachable API origin → API container
```

The browser must never access SQLite directly. Keep routes responsible for HTTP concerns, schemas responsible for boundary validation/serialization, service for use-case orchestration, repository for persistence, and session/database module for lifecycle. [ASSUMED]

### Recommended Project Structure

```text
backend/
├── app/main.py              # FastAPI app, middleware, router registration
├── app/core/config.py       # DATABASE_URL and explicit CORS origins
├── app/db/session.py        # engine/session dependency and bootstrap/migration hook
├── app/books/models.py      # SQLAlchemy books table
├── app/books/schemas.py     # create/update/read/error contract
├── app/books/validation.py  # shared server normalization and ISBN/year rules
├── app/books/repository.py  # persistence only
├── app/books/service.py      # use-case orchestration
└── tests/                   # isolated API/database tests
frontend/
├── src/api/client.ts        # one fetch/error boundary
├── src/types/book.ts        # client contract types
└── src/App.tsx              # initial collection shell
```

The paths above are proposed new paths, not existing repository files. [ASSUMED]

### Pattern 1: Contract-first schemas

**What:** Define separate `BookCreate`, `BookUpdate`, and `BookRead` models. Require the five fields in create/update, trim text before rejecting blank values, validate year and ISBN, and expose a stable error envelope rather than making React parse framework-native error details. FastAPI’s documented Pydantic models produce validation and OpenAPI schema automatically. [CITED: https://fastapi.tiangolo.com/tutorial/body/]

**When to use:** Immediately in Phase 1, before Phase 2 CRUD routes and before duplicating client types.

### Pattern 2: Explicit database dependency

**What:** Create one engine from a configurable URL, yield one short-lived synchronous session per request, commit mutations explicitly, rollback on exceptions, and close the session. Tests override the dependency with a temporary database. SQLAlchemy documents SQLite-specific transaction and memory-database behavior that makes implicit connection assumptions risky. [CITED: https://docs.sqlalchemy.org/en/20/dialects/sqlite.html]

### Pattern 3: Readiness is a database probe

**What:** `GET /api/v1/health` returns success only after a lightweight SQLite query succeeds; database errors produce a non-success readiness response without leaking paths or tracebacks. Add a simple process/container healthcheck later, but do not equate “process running” with “database usable.” [ASSUMED]

### Pattern 4: Explicit CORS configuration

**What:** Configure `CORSMiddleware` from exact origins including scheme, host, and port; allow only methods/headers used by JSON CRUD; test both an `Origin` response and an `OPTIONS` preflight. FastAPI explicitly defines origin as protocol, host, and port and documents preflight handling. [CITED: https://fastapi.tiangolo.com/tutorial/cors/]

**Anti-patterns to avoid:** wildcard CORS as a default; browser-facing `http://api:8000`; SQL in route handlers; client-only validation; a global SQLite connection; tests against the development database; relative database paths; and claiming readiness from an HTTP process check alone. [ASSUMED]

## Validation Policy Decisions Phase 1 Must Settle

These are not locked in `CONTEXT.md`; the planner should turn the recommendations into explicit implementation decisions and tests.

1. **Publication year:** recommend integer `1..current calendar year` for v1. Reject decimals, text, missing values, future years, and `0` unless the user explicitly wants BCE/unknown-era records. [ASSUMED]
2. **ISBN input/storage:** accept ISBN-10 and ISBN-13 with spaces/hyphens; remove separators for validation and comparison; preserve one documented canonical representation (recommend normalized digits with ISBN-10 `X` retained in the check-digit position rather than silently converting to ISBN-13). [ASSUMED]
3. **Text fields:** trim title, author, and genre; reject empty-after-trim; set practical maximum lengths (recommend title/author 255 and genre 100). Preserve internal punctuation, case, diacritics, and multiple-author text. [ASSUMED]
4. **Duplicate ISBN:** do not add a hard uniqueness constraint in Phase 1; editions and formats make ISBN a poor universal record identity without a product edition policy. Use generated integer ID. [ASSUMED]
5. **Error contract:** use one application envelope such as `{ "error": { "code": "VALIDATION_ERROR", "message": "...", "fields": { "isbn": "..." } } }`; normalize FastAPI validation errors at the boundary. [ASSUMED]
6. **Schema initialization:** choose either first-run Alembic migration or a clearly temporary idempotent bootstrap; do not leave implicit `create_all()` as the long-term evolution strategy. [ASSUMED]

## Don't Hand-Roll

| Problem | Don't build | Use instead | Why |
|---|---|---|---|
| HTTP request validation/OpenAPI | Dict parsing and manual field checks in routes | Pydantic models through FastAPI | FastAPI supplies conversion, validation, field locations, JSON Schema, and OpenAPI. [CITED: https://fastapi.tiangolo.com/tutorial/body/] |
| SQLite session/transaction lifecycle | Global connection or scattered `commit()` calls | SQLAlchemy session dependency and repository | Centralizes lifecycle, rollback, injection, and transaction tests. [CITED: https://docs.sqlalchemy.org/en/20/dialects/sqlite.html] |
| ISBN check-digit logic | Ad-hoc regex-only validation | A vetted ISBN implementation or a small well-tested domain validator | Check digits and ISBN-10 `X` semantics are easy to get wrong; if a package is added, run the legitimacy gate first. [ASSUMED] |
| Browser/API error parsing | Per-component `fetch` logic | One API client wrapper and stable server envelope | Prevents drift for JSON errors, `204`, network failures, and field mapping. [ASSUMED] |

**Key insight:** Phase 1’s hard problem is boundary consistency, not CRUD volume. Custom shortcuts create validation drift and untestable database state; the planner should spend complexity on seams and verification, not optional libraries.

## Common Pitfalls

### Pitfall 1: The existing Docker scaffold is mistaken for a runnable app

**What goes wrong:** `Dockerfile` references missing `requirements.txt` and `app.main:app`; current Compose mounts source into `/app` but has no frontend or persistent data volume. [VERIFIED: `Dockerfile:5-10`; quote: "COPY requirements.txt ." and "COPY . ."; VERIFIED: `docker-compose.yml:1-7`; quote: "- .:/app"]
**How to avoid:** Make the first plan task establish a runnable backend import path and dependency lock, then add the frontend and replace the bind-mounted database path with named `/data` storage.
**Warning sign:** `docker compose build` fails before application tests or API import succeeds.

### Pitfall 2: Validation exists only in the future React form

**How to avoid:** POST invalid payloads directly against the API; assert field-specific errors, no database write, and identical normalization for create/update. FastAPI’s typed body models are the appropriate boundary. [CITED: https://fastapi.tiangolo.com/tutorial/body/]

### Pitfall 3: Year/ISBN policy is left implicit

**How to avoid:** Lock the six policy decisions above in the phase contract and create boundary test matrices for whitespace, future/zero years, ISBN separators, ISBN-10 `X`, wrong check digits, and malformed lengths. [ASSUMED]

### Pitfall 4: Tests accidentally use `:memory:` across different connections

**How to avoid:** Prefer a per-test temporary file database for Phase 1; if memory is used, configure one deliberately shared connection/pool and test it. SQLAlchemy documents special memory-database pooling/thread behavior. [CITED: https://docs.sqlalchemy.org/en/20/dialects/sqlite.html]

### Pitfall 5: CORS is tested with curl but not a browser preflight

**How to avoid:** Test exact development origin and JSON `OPTIONS` request including requested method/headers. Origins differ by scheme, host, and port. [CITED: https://fastapi.tiangolo.com/tutorial/cors/]

### Pitfall 6: Health says “ok” without checking SQLite

**How to avoid:** Execute a database query in readiness and verify a failure path with an invalid/unavailable test database configuration. [ASSUMED]

## Code Examples

### FastAPI contract shape

```python
class BookCreate(BaseModel):
    title: str
    author: str
    publication_year: int
    genre: str
    isbn: str

@router.post("/books", response_model=BookRead, status_code=201)
def create_book(payload: BookCreate, session: Session = Depends(get_session)):
    ...
```

This follows the official FastAPI pattern of declaring a Pydantic model as the request-body parameter and using typed response metadata. [CITED: https://fastapi.tiangolo.com/tutorial/body/]

### CORS configuration shape

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_methods=["GET", "POST", "PUT", "DELETE", "OPTIONS"],
    allow_headers=["Content-Type"],
)
```

The exact origin values belong in environment/configuration and must include scheme, host, and port. [CITED: https://fastapi.tiangolo.com/tutorial/cors/]

### API test seam

```python
def test_invalid_book_is_rejected(client):
    response = client.post("/api/v1/books", json={"title": "   "})
    assert response.status_code == 422
    assert "title" in response.json()["error"]["fields"]
```

Use ordinary pytest functions with FastAPI `TestClient`; the database dependency should be overridden to an isolated test database. [CITED: https://fastapi.tiangolo.com/tutorial/testing/]

## Runtime State Inventory

This is a greenfield foundation, not a rename/refactor/migration phase. No stored data, live service configuration, OS registrations, secrets/env vars, or build artifacts were found in the repository during inspection. The existing Dockerfile/Compose files are source configuration, not runtime state; they require replacement/extension as implementation work. [VERIFIED: repository inspection and `Dockerfile:1-13`, `docker-compose.yml:1-7`]

## Environment Availability

| Dependency | Required by | Available | Version | Fallback |
|---|---|---|---|---|
| Node.js | Vite React scaffold | ✓ | `v22.22.1` | — |
| npm | Frontend package install | ✓ | `9.2.0` | — |
| Python | FastAPI backend | ✓ | `3.14.4` | Use Docker’s existing Python `3.12-slim` baseline until image version is deliberately chosen. |
| Python pip module | Python registry verification | ✗ | `python3 -m pip` → `No module named pip` | Use the selected resolver in Docker or install/verify a resolver as a gated setup task. |
| Docker Engine | Container foundation | ✓ | `29.7.2` | — |
| Docker Compose | Compose topology/health | ✓ | `v5.3.1` | — |
| `uv` | Preferred Python lock workflow from prior research | ✗ | Not found | Use pip/requirements lock if human verification selects it; do not mix workflows. |
| Browser binary | Later browser smoke | Not probed | — | Phase 4 can install/use Playwright behind its package checkpoint. |

## Validation Architecture

Validation is explicitly disabled by project configuration, so the formal Nyquist section is omitted. [VERIFIED: `.planning/config.json:26`; quote: `"nyquist_validation": false`] This does **not** remove Phase 1 verification: API-01/API-02/API-03/BOOK-05/BOOK-06 still require focused tests and a manual contract check.

### Phase 1 verification policy

- Backend quick command: `pytest -q` after the backend lock and test configuration exist. [ASSUMED]
- Contract checks: health success/failure, OpenAPI schema presence, valid/invalid create payloads, trimming, year boundaries, ISBN cases, commit/read-back using a fresh session, and CORS origin/preflight. [ASSUMED]
- Isolation check: each test uses a temporary database and never the configured development or Docker volume database. [ASSUMED]
- Manual gate: inspect `/docs` or `/openapi.json`, send a direct invalid HTTP request, and verify the CORS headers from the configured frontend origin. [ASSUMED]
- Phase 1 does not need Playwright or full UI behavior tests; Phase 3/4 own browser-visible state and end-to-end CRUD verification. [VERIFIED: `.planning/ROADMAP.md:40-63`; quote: "Phase 3: React Collection Experience" and "Phase 4: Docker Delivery and Behavioral Verification"]

## Security Domain

Security enforcement is enabled at ASVS level 1. [VERIFIED: `.planning/config.json:47-50`; quote: `"security_enforcement": true` and `"security_asvs_level": 1`] This is a local single-user app with authentication explicitly out of scope, but untrusted browser/API input still requires validation and safe database handling.

### Applicable ASVS categories

| ASVS category | Applies | Phase 1 control |
|---|---|---|
| V2 Authentication | No | Authentication is explicitly out of scope for the initial single-user app. [VERIFIED: `.planning/PROJECT.md:28-31`; quote: "User accounts and authentication — not needed for the initial single-user learning application."] |
| V3 Session Management | No | No login/session contract in this phase. [ASSUMED] |
| V4 Access Control | Limited | No multi-user authorization; keep SQLite and mutation API local/private by deployment convention. CORS is not authorization. [CITED: https://fastapi.tiangolo.com/tutorial/cors/] |
| V5 Input Validation | Yes | Pydantic models, trim/blank rejection, year bounds, ISBN check digit, bounded text lengths, stable field errors, and parameterized ORM queries. [CITED: https://fastapi.tiangolo.com/tutorial/body/] |
| V6 Cryptography | No | No secrets, passwords, tokens, or sensitive cryptographic material are required by the phase scope. [VERIFIED: `.planning/PROJECT.md:28-34`; quote: "User accounts and authentication" is out of scope. ] |

### Known threat patterns

| Pattern | STRIDE | Mitigation |
|---|---|---|
| SQL injection through future search/mutations | Tampering | SQLAlchemy bound parameters/repository queries; never concatenate user input into SQL. [ASSUMED] |
| Oversized/malformed JSON/text | Denial of service / Tampering | Pydantic type and length bounds at API boundary; reject before persistence. [CITED: https://fastapi.tiangolo.com/tutorial/body/] |
| Overly broad browser origin policy | Spoofing / Elevation of privilege | Exact configured origins; test preflight; do not claim CORS is authorization. [CITED: https://fastapi.tiangolo.com/tutorial/cors/] |
| Database file exposure or loss | Information disclosure / Tampering | API-only SQLite, no frontend mount, absolute configured path, named volume in delivery phase, and no secrets in error responses. [CITED: https://docs.docker.com/engine/storage/volumes/] |

## Open Questions

1. **Will the user accept the recommended year `1..current year` policy?** Current project state explicitly says Phase 1 must settle the year boundary, but no decision is locked. [VERIFIED: `.planning/STATE.md:61-63`; quote: "Phase 1 should settle the publication-year boundary and ISBN canonicalization policy before implementation."] Recommendation: lock `1..current year` unless BCE support is a stated learning requirement. [ASSUMED]
2. **Should ISBN-10 be stored as normalized ISBN-10 or converted to ISBN-13?** Recommendation: do not silently convert in Phase 1; preserve a documented normalized representation and defer duplicate/edition policy. [ASSUMED]
3. **Which Python resolver should be authoritative?** `uv` is not installed; choose one resolver and commit one lock workflow, then verify package versions in that environment. [ASSUMED]
4. **Should Phase 1 include a minimal React scaffold or only the API/foundation seam?** The phase goal includes React runtime readiness and API contract, while Phase 3 owns the collection experience. Recommendation: scaffold/build a minimal React app and typed client boundary, but defer full form/list behavior. [VERIFIED: `.planning/ROADMAP.md:9-12`; quote: "Establish the FastAPI/React runtime" and "Phase 3: React Collection Experience"]

## Sources

### Primary (official / authoritative)

- FastAPI request bodies and Pydantic/OpenAPI: https://fastapi.tiangolo.com/tutorial/body/
- FastAPI CORS and preflight: https://fastapi.tiangolo.com/tutorial/cors/
- FastAPI TestClient/pytest: https://fastapi.tiangolo.com/tutorial/testing/
- SQLAlchemy 2 SQLite dialect and transaction/pooling behavior: https://docs.sqlalchemy.org/en/20/dialects/sqlite.html
- Vite Getting Started and current Node requirement: https://vite.dev/guide/
- Docker volumes: https://docs.docker.com/engine/storage/volumes/
- Docker Compose startup order and healthchecks: https://docs.docker.com/compose/how-tos/startup-order/

### In-repository authority

- `.planning/PROJECT.md`, `.planning/ROADMAP.md`, `.planning/REQUIREMENTS.md`, `.planning/STATE.md`, `.planning/config.json` — scope, requirements, phase success criteria, workflow/security policy.
- `Dockerfile`, `docker-compose.yml` — current incomplete runtime scaffold.
- `.claude/CLAUDE.md` — project constraints, generated stack summary, and GSD workflow convention.

## Assumptions Log

| # | Claim | Section | Risk if wrong |
|---|---|---|---|
| A1 | Publication year should be `1..current year`. | Validation Policy | Historical/BCE records could be rejected; contract and UI tests would need revision. |
| A2 | ISBN separators are removed for validation/storage and ISBN-10 is not converted. | Validation Policy | Display/search/duplicate behavior could differ from user expectations. |
| A3 | Text maximums of title/author 255 and genre 100 are suitable. | Validation Policy | Legitimate metadata could be rejected or database constraints could change. |
| A4 | No hard ISBN uniqueness is needed in v1. | Validation Policy | Duplicate handling expectations could require a constraint and `409` contract. |
| A5 | A stable custom error envelope is preferable to exposing FastAPI’s native validation shape. | Validation Policy / Architecture | Frontend mapping and API tests would need to use native `detail` instead. |
| A6 | Native `fetch`, no query cache, and a minimal React scaffold are sufficient. | Standard Stack / Open Questions | Later UI complexity may justify a library, but it should not be added before evidence. |
| A7 | A temporary-file SQLite database is the simplest isolated test seam. | Architecture / Validation | Test speed or concurrency requirements could favor a configured shared in-memory connection. |

## Metadata

**Confidence breakdown:**
- Standard stack: MEDIUM — project constraints and official docs are clear; current package legitimacy/version checks are unsettled and mostly `[SUS]`.
- Architecture: HIGH — boundaries follow the explicit project requirements and official FastAPI/SQLAlchemy/Docker mechanics.
- Validation policy: LOW-MEDIUM — the requirements identify the unresolved decisions, but year and ISBN canonicalization are recommendations awaiting a project decision.
- Pitfalls: HIGH for mechanics; project is empty, so implementation-specific failure frequency is not yet observable.

**Research date:** 2026-08-14
**Valid until:** 2026-09-13 for stable architecture; re-check package versions and legitimacy immediately before installation.
