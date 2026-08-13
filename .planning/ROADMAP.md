# Roadmap: Book Tracker

## Overview

Deliver a focused vertical MVP of a single-user book collection: establish a reproducible full-stack foundation and authoritative data contract, prove the persisted CRUD/search API, connect it to an accessible React collection workflow, and finish with automated and Docker-based release verification.

## Phases

- [ ] **Phase 1: Foundation and Domain Contract** - Establish the FastAPI/React runtime, SQLite persistence boundary, validation rules, health endpoint, and stable API contract.
- [ ] **Phase 2: CRUD and Search API** - Deliver the tested backend workflow for creating, listing, searching, editing, and confirmed deletion of book records.
- [ ] **Phase 3: React Collection Experience** - Make the complete book workflow usable in the browser with accessible forms, feedback, and resilient UI states.
- [ ] **Phase 4: Docker Delivery and Behavioral Verification** - Prove the assembled application starts cleanly, preserves data, and passes end-to-end release checks.

## Phase Details

### Phase 1: Foundation and Domain Contract
**Goal**: Users and later phases have a reproducible application foundation with a trustworthy, documented boundary for valid book data and database readiness.
**Mode**: mvp
**Depends on**: Nothing (first phase)
**Requirements**: API-01, API-02, API-03, BOOK-05, BOOK-06
**Success Criteria** (what must be TRUE):
  1. The API exposes a documented, consistent book request/response contract that the React application can target.
  2. Invalid or whitespace-only required text, unsupported publication years, and malformed ISBNs are rejected consistently at the API boundary, with field-specific errors.
  3. Valid book records are stored in SQLite with required fields and committed transactions, while automated tests use an isolated database.
  4. A readiness/health check reports whether the API can use its database, and browser requests from the configured development frontend origin are accepted through explicit CORS configuration.
**Plans**: TBD

### Phase 2: CRUD and Search API
**Goal**: The backend reliably supports the complete personal collection data workflow, including deterministic browsing and simple search.
**Mode**: mvp
**Depends on**: Phase 1
**Requirements**: BOOK-01, BOOK-02, BOOK-03, BOOK-04, SRCH-01, SRCH-02
**Success Criteria** (what must be TRUE):
  1. A client can create a complete five-field book record and retrieve it from the collection in a deterministic order.
  2. A client can update any existing book by stable ID and retrieve the saved values afterward.
  3. A client can delete one existing book only through the delete operation for that record, and the record is absent from subsequent collection results.
  4. A single case-insensitive substring query matches at least title and author, while an empty or cleared query returns the full collection.
**Plans**: TBD

### Phase 3: React Collection Experience
**Goal**: Users can maintain and find their books through a clear, accessible React interface connected to the real API.
**Mode**: mvp
**Depends on**: Phase 2
**Requirements**: SRCH-03, UX-01, UX-02, UX-03, UX-04
**Success Criteria** (what must be TRUE):
  1. The collection screen distinguishes loading, empty collection, no search matches, and API-failure states without presenting a blank or misleading result.
  2. Users can submit the shared labeled add/edit form, receive visible success feedback after confirmed create or update, and retain useful entered values when a request fails.
  3. Users can edit a selected record and delete it only after a confirmation that names the book; successful deletion visibly removes the record and reports completion.
  4. Users receive field-level validation errors and useful general server/network errors, with required controls and errors accessible by label and association.
  5. Pending mutation controls prevent accidental double submission while a request is in progress.
**Plans**: TBD
**UI hint**: yes

### Phase 4: Docker Delivery and Behavioral Verification
**Goal**: Users can start and trust the complete Book Tracker application from a clean checkout, including durable local data and verified core behavior.
**Mode**: mvp
**Depends on**: Phase 3
**Requirements**: SHIP-01, SHIP-02, SHIP-03
**Success Criteria** (what must be TRUE):
  1. A clean checkout starts the usable web application through Docker with the API ready and the browser able to reach the frontend and API.
  2. A book created through the running Docker application remains available after API/container recreation when the documented named volume is retained.
  3. Automated backend, frontend, integration, and container smoke checks verify the core add, list/search, edit, and confirmed-delete journey.
**Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation and Domain Contract | 0/TBD | Not started | - |
| 2. CRUD and Search API | 0/TBD | Not started | - |
| 3. React Collection Experience | 0/TBD | Not started | - |
| 4. Docker Delivery and Behavioral Verification | 0/TBD | Not started | - |

**Coverage:** 19/19 v1 requirements mapped; no orphaned or duplicated requirements.
