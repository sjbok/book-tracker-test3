# Walking Skeleton — Book Tracker

**Phase:** 1
**Generated:** 2026-08-14

## Capability Proven End-to-End

A user can start the documented local runtime, see API readiness, submit one complete book payload through the React shell, and see the persisted server record read back from SQLite.

## Architectural Decisions

| Decision | Choice | Rationale |
|---|---|---|
| Framework | Vite React TypeScript SPA + synchronous FastAPI | Preserves the explicit React/FastAPI learning boundary and keeps browser code separate from API code. |
| Data layer | SQLite at `sqlite:////data/books.db` behind synchronous SQLAlchemy sessions | Fits the single-user local scope while making transactions, dependency injection, and named-volume persistence explicit. |
| Auth | No authentication | The initial product is intentionally single-user and local. |
| Deployment target | Docker Compose API/frontend services plus documented local development commands | Reproduces the process boundary and browser/API origin needed by later delivery verification. |
| Directory layout | `backend/app/{core,db,books}` and `frontend/src/{api,types,validation}` | Keeps HTTP, validation, persistence, and browser contract responsibilities visible and independently testable. |
| Domain policy | Year `1..current calendar year`; ISBN separators removed, ISBN-10 `X` retained, no ISBN-10→13 conversion | Makes the on-wire and stored representation deterministic before CRUD and UI expansion. |

## Stack Touched in Phase 1

- [x] Project scaffold (framework, build, test runner)
- [x] Routing — `/api/v1/health` and versioned book routes
- [x] Database — one real read and one real write
- [x] UI — one interactive tracer path wired to the API
- [x] Deployment — documented Docker Compose and local full-stack run commands

## Out of Scope (Deferred to Later Slices)

- Complete collection list/search/edit/delete UX and confirmation flow
- Production browser smoke and container restart-persistence release gate
- Authentication, accounts, external metadata imports, covers, ratings, reviews, lending, imports/exports, bulk operations, pagination, fuzzy search, sorting, and duplicate ISBN warnings

## Subsequent Slice Plan

- Phase 2: Complete persisted CRUD and deterministic title/author search API.
- Phase 3: Connect the accessible React collection workflow, form feedback, search, edit, and confirmed delete.
- Phase 4: Build release images and verify clean startup, browser behavior, and named-volume persistence.
