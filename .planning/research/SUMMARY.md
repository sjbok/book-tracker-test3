# Project Research Summary

**Project:** Book Tracker  
**Domain:** Single-user personal book database with CRUD and search  
**Researched:** 2026-08-13  
**Confidence:** MEDIUM-HIGH

## Executive Summary

Book Tracker is a deliberately small full-stack web application whose core value is reliable maintenance and retrieval of book records. Research consistently recommends a vertical, contract-first build: React/TypeScript in the browser, a FastAPI HTTP boundary, SQLite persistence behind SQLAlchemy, and Docker Compose for reproducible delivery. The v1 should remain focused on five required fields, deterministic listing, simple case-insensitive search, edit, and explicitly confirmed delete.

The strongest implementation choice is to keep responsibilities separated: Pydantic owns API validation, services and repositories own business and persistence behavior, and React owns presentation and transient UI state. Use synchronous request-scoped SQLAlchemy sessions, an absolute `/data/books.db` path on a named Docker volume, explicit CORS, centralized frontend fetch/error handling, and server-authoritative mutation responses. The main risks are integration failures rather than feature complexity: validation drift, lost SQLite data, unsafe or ambiguous search, CORS/configuration errors, and tests that do not exercise the real database and browser path.

## Key Findings

### Recommended Stack

Use a TypeScript React SPA scaffolded with Vite, Node 22 LTS, FastAPI with Pydantic v2, Python 3.13 (or 3.12 if image compatibility requires it), synchronous SQLAlchemy 2, SQLite, and Alembic before schema evolution. Keep the client deliberately light: native `fetch` behind one API client, local React state, plain CSS/CSS Modules, and no Redux, Axios, or query-cache framework initially. Use pytest/FastAPI TestClient, Vitest/React Testing Library, and one Playwright browser smoke path.

**Core technologies:**
- **React + TypeScript + Vite:** accessible typed SPA and fast, simple build pipeline; matches the explicit frontend constraint.
- **FastAPI + Pydantic v2 + Uvicorn:** typed REST contract, OpenAPI, dependency injection, and authoritative request validation.
- **SQLAlchemy 2 + SQLite:** explicit repository/session boundary and transactions with minimal operational overhead for a local single-user catalog.
- **Docker Compose + static frontend container:** reproducible two-service delivery with a named `/data` volume for persistence.
- **pytest, Vitest/Testing Library, Playwright:** layered API, UI behavior, browser integration, and container smoke verification.

Exact fast-moving package versions should be resolved into one committed lock workflow during implementation; do not copy observed versions blindly.

### Expected Features

**Must have (table stakes):**
- Shared add/edit form for required title, author, publication year, genre, and ISBN.
- Deterministically ordered collection list with loading, empty, no-match, and API-failure states.
- Case-insensitive substring search, at minimum across title and author, with clear-query behavior.
- Edit by stable ID and single-record delete requiring confirmation that names the book.
- Server-authoritative trimming and validation, including year and ISBN rules, with field-level and general feedback.
- Visible success feedback, pending-submit protection, and accessible labels/errors.

**Should have (competitive):**
- Simple search across genre and normalized ISBN if it preserves one uncomplicated query contract.
- Simple sort controls, the best lightweight differentiator once the default list is reliable.
- A non-blocking duplicate normalized-ISBN warning, only after the edition policy is clear.

**Defer (v2+):**
- Accounts, authentication, sharing, external metadata lookup, covers, ratings, reviews, reading status, shelves, and lending.
- Imports/exports, detail pages, bulk operations, pagination, fuzzy/full-text search, and hard uniqueness rules without an edition policy.

### Architecture Approach

Use a layered two-tier system: React calls `/api/v1`; FastAPI routes use Pydantic schemas, a book service, and a repository backed by request-scoped SQLAlchemy sessions; SQLite is API-only. Keep a single collection controller as the frontend source of truth, with separate server and form state, explicit loading/mutation states, stable IDs, and a centralized API client. Compose should run an API service and static Nginx frontend, with `/health`, explicit origins, `0.0.0.0` binding, and a named volume at `/data`.

**Major components:**
1. **React shell, search, list, and shared form** — collection state, accessible interaction, mutation feedback, and distinct UI states.
2. **API client and FastAPI routes/schemas** — typed HTTP contract, normalized errors, status codes, CORS, and health readiness.
3. **Book service/repository and database session** — use-case orchestration, parameterized queries, transaction boundaries, and isolated test injection.
4. **Docker/Compose delivery** — immutable frontend assets, API process, health checks, browser-reachable URLs, and durable SQLite volume.

### Critical Pitfalls

1. **Validation only in React** — require typed Pydantic create/update models, boundary normalization, database `NOT NULL` constraints, and direct API edge-case tests.
2. **SQLite data disappearing** — use `/data/books.db`, explicit commits/rollbacks, a named volume, and a stop/recreate persistence test.
3. **Unsafe or ambiguous search** — parameterize values, define fields/case/substring semantics, escape wildcard behavior deliberately, and use deterministic ordering.
4. **CORS and browser URL mistakes** — configure exact scheme/host/ports, test JSON preflight from the real origin, and never confuse browser `localhost` with the Compose service name.
5. **False confidence from weak tests or startup races** — isolate test databases, test user-visible states and real round trips, add `/health` and bounded healthchecks, and run a clean Docker CRUD smoke flow.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Reproducible Foundation and Domain Contract
**Rationale:** Every later layer depends on a stable runtime, schema, validation policy, and test seam.  
**Delivers:** React/FastAPI project skeleton, pinned lockfiles, Compose skeleton, `/health`, SQLAlchemy session/repository structure, `books` schema, migrations/bootstrap decision, API error envelope, and isolated test database.  
**Addresses:** API-01, API-02, API-03, BOOK-05, BOOK-06.  
**Avoids:** validation drift, uncommitted/relative SQLite writes, unsafe global connections, and schema initialization surprises.

### Phase 2: Backend CRUD and Search Vertical Slice
**Rationale:** Stabilize and test the real API before UI work; this is the durable dependency for all browser behavior.  
**Delivers:** `/api/v1/books` list/create/get/update/delete endpoints, deterministic ordering, title/author search, normalized validation, correct status codes, transactions, and API contract tests.  
**Addresses:** BOOK-01 through BOOK-04, SRCH-01 and SRCH-02.  
**Avoids:** SQL interpolation, unspecified ISBN/year semantics, arbitrary ordering, and contract/error-shape drift.

### Phase 3: React Collection and Mutation UX
**Rationale:** Connect the browser to the real API before adding polish or optional product features.  
**Delivers:** typed fetch client, collection list, search control, shared add/edit form, named delete confirmation, loading/empty/no-match/failure states, field errors, success feedback, accessible labels, and pending guards.  
**Addresses:** all Book Records and Search requirements plus UX-01 through UX-04.  
**Avoids:** CORS failures, stale optimistic state, double submissions, client-only validation, and inaccessible forms.

### Phase 4: Automated Behavioral Verification
**Rationale:** Verification should cover the complete user contract while implementation context is still fresh, not be deferred to release.  
**Delivers:** backend round-trip and persistence tests, frontend user-visible tests, API/UI integration coverage, CORS preflight coverage, type/lint checks, and a browser CRUD smoke flow.  
**Addresses:** SHIP-03 and the Definition of Done across all v1 requirements.  
**Avoids:** shared test state, implementation-detail tests, mock-only confidence, and untested failure states.

### Phase 5: Docker Packaging and Release Gate
**Rationale:** Docker is a product requirement and must prove the assembled system, not merely build images.  
**Delivers:** multi-stage Vite/Nginx frontend image, FastAPI image, explicit environment configuration, named volume, healthcheck/readiness behavior, clean-checkout startup, browser-reachable API configuration, and create/list/search/edit/delete plus restart-persistence smoke verification.  
**Addresses:** SHIP-01 and SHIP-02, with final validation of SHIP-03.  
**Avoids:** lost data, `127.0.0.1` container binding, service-start races, wildcard CORS, dev-server leakage, and destructive `down -v` confusion.

### Phase Ordering Rationale

- The dependency chain is runtime → persistence/schema → API contract → React state/UI → integrated tests → container release verification.
- A first backend round trip and then a real browser connection minimize mock drift and expose contract/CORS problems early.
- Search belongs with the API contract but should remain simple; delete belongs after stable list state because it depends on IDs, reconciliation, and confirmation UX.
- The optional differentiators should not displace the five-field CRUD workflow; choose sorting only after core acceptance passes.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1:** Resolve exact package/image versions, choose `uv` versus pip-tools, and settle year `0`/BCE and ISBN canonicalization policy.
- **Phase 5:** Validate Docker/browser networking, healthcheck timing, image tags/digests, and persistence behavior on the target platform.

Phases with standard patterns (skip research-phase):
- **Phase 2:** FastAPI/Pydantic CRUD, SQLAlchemy sessions, parameterized SQLite queries, and OpenAPI are well documented.
- **Phase 3:** Small React forms, Testing Library-oriented states, and explicit fetch handling are established patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | Architectural choices are sound, but observed 2026 versions are fast-moving and source-provider confidence was often low; lock and test exact versions. |
| Features | HIGH for project scope; MEDIUM for external comparison | Project requirements strongly define v1; competitor research was not independently verified. |
| Architecture | HIGH | Official documentation and clear dependency boundaries converge on the layered vertical-slice design. |
| Pitfalls | HIGH for mechanics; MEDIUM for frequency | Framework, SQLite, CORS, testing, and Compose failure modes are well documented; project-specific likelihood is inferred. |

**Overall confidence:** MEDIUM-HIGH

### Gaps to Address

- Decide and document whether publication year accepts `0`/BCE or uses `1..current year`.
- Decide whether valid ISBN-10 is displayed as entered/normalized or converted to ISBN-13; keep server and client behavior identical.
- Choose one Python dependency resolver and commit its lockfile; pin compatible base-image tags/digests after a real build.
- Confirm whether the first UI needs routing; a single-screen MVP can omit React Router.
- Validate exact API base URL/CORS behavior in both Vite development and Compose-served modes.

## Sources

### Primary (HIGH confidence)
- `.planning/PROJECT.md` and `.planning/REQUIREMENTS.md` — authoritative scope, constraints, v1 requirements, and out-of-scope decisions.
- FastAPI request bodies, CORS, and testing documentation — typed validation, API contracts, browser access, and TestClient patterns.
- SQLAlchemy SQLite documentation and Python `sqlite3` documentation — sessions, transactions, connection behavior, and persistence.
- Docker Compose networking/startup-order and Docker volumes documentation — service readiness, host/container networking, and volume lifecycle.
- React documentation, MDN HTTP/CORS guidance, and React Testing Library principles — state flow, browser semantics, and user-facing test strategy.

### Secondary (MEDIUM confidence)
- `.planning/research/ARCHITECTURE.md` — synthesized component boundaries, API/UI data flow, and dependency-driven build order.
- `.planning/research/PITFALLS.md` — phase-mapped prevention strategies based on official mechanics.

### Tertiary (LOW confidence)
- `.planning/research/STACK.md` package/version observations — useful starting points, but exact versions require fresh resolution and lockfiles.
- `.planning/research/FEATURES.md` W3C/NN/g guidance and competitor landscape — feature/UX recommendations are sensible, but external provider confidence and competitor verification were limited.

---
*Research completed: 2026-08-13*  
*Ready for roadmap: yes*
