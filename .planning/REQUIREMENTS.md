# Requirements: Book Tracker

**Defined:** 2026-08-13
**Core Value:** Users can reliably find and maintain accurate book records through a simple end-to-end web application.

## v1 Requirements

Requirements for the initial single-user release.

### Book Records

- [ ] **BOOK-01**: User can add a book with title, author, publication year, genre, and ISBN.
- [ ] **BOOK-02**: User can view every saved book in a deterministic collection list.
- [ ] **BOOK-03**: User can edit any existing book and see the saved changes in the collection.
- [ ] **BOOK-04**: User can delete one existing book only after an explicit confirmation naming the book.
- [ ] **BOOK-05**: The system trims required text fields and rejects blank or malformed values at the API boundary.
- [ ] **BOOK-06**: The system validates publication year and ISBN format consistently in the frontend and backend.

### Search

- [ ] **SRCH-01**: User can search the collection with one case-insensitive substring query.
- [ ] **SRCH-02**: Search matches at least title and author, and clearing the query restores the full collection.
- [ ] **SRCH-03**: The collection distinguishes loading, empty, no-match, and API-failure states.

### Feedback And Accessibility

- [ ] **UX-01**: User receives visible success feedback after a confirmed create, update, or delete.
- [ ] **UX-02**: User receives field-level validation feedback and a useful general message for server or network failures.
- [ ] **UX-03**: All book form controls have visible labels, required state, and text errors associated with the relevant control.
- [ ] **UX-04**: Mutation controls prevent accidental double submission while a request is pending.

### Application Foundation

- [ ] **API-01**: React communicates with a FastAPI API using a documented, consistent request and response contract.
- [ ] **API-02**: Book data is persisted in SQLite with committed transactions, required database fields, and an isolated test database.
- [ ] **API-03**: The API provides a readiness/health check and explicit CORS configuration for the development frontend origin.

### Delivery

- [ ] **SHIP-01**: The application starts from a clean checkout with Docker and exposes the usable web application.
- [ ] **SHIP-02**: Book data survives API/container recreation when the documented Docker volume is retained.
- [ ] **SHIP-03**: Automated backend, frontend, integration, and container smoke checks verify the core user workflow.

## v2 Requirements

Deferred until the focused CRUD workflow is proven.

### Catalog Enhancements

- **CAT-01**: User can sort the collection by author, year, or genre.
- **CAT-02**: User receives a non-blocking warning for a duplicate normalized ISBN.
- **CAT-03**: User can import or export the collection as CSV or JSON.

### Product Features

- **PROD-01**: User can view a dedicated book detail page.
- **PROD-02**: User can track ratings, reviews, reading status, shelves, or lending.

## Out of Scope

| Feature | Reason |
|---------|--------|
| User accounts and authentication | The initial application is intentionally single-user and local. |
| External metadata lookup or ISBN imports | Manual entry keeps the first release deterministic and avoids API/rate-limit complexity. |
| Covers, uploads, social sharing, and native mobile apps | These expand the product beyond the core database learning objective. |
| Bulk operations, advanced fuzzy search, pagination, and production-scale hosting | The expected collection is small and the first release prioritizes clear CRUD behavior. |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| BOOK-01 | Phase 2 | Pending |
| BOOK-02 | Phase 2 | Pending |
| BOOK-03 | Phase 2 | Pending |
| BOOK-04 | Phase 2 | Pending |
| BOOK-05 | Phase 1 | Pending |
| BOOK-06 | Phase 1 | Pending |
| SRCH-01 | Phase 2 | Pending |
| SRCH-02 | Phase 2 | Pending |
| SRCH-03 | Phase 3 | Pending |
| UX-01 | Phase 3 | Pending |
| UX-02 | Phase 3 | Pending |
| UX-03 | Phase 3 | Pending |
| UX-04 | Phase 3 | Pending |
| API-01 | Phase 1 | Pending |
| API-02 | Phase 1 | Pending |
| API-03 | Phase 1 | Pending |
| SHIP-01 | Phase 4 | Pending |
| SHIP-02 | Phase 4 | Pending |
| SHIP-03 | Phase 4 | Pending |

**Coverage:**
- v1 requirements: 19 total
- Mapped to phases: 19
- Unmapped: 0

## User Stories

- As a single user, I can add a complete book record so my collection has accurate metadata.
- As a single user, I can search and browse my collection so I can find a book quickly.
- As a single user, I can correct or remove a record so the collection stays trustworthy.
- As a learner, I can run and verify the full application through Docker so the workflow is reproducible.

## Definition Of Done

- The core add, list, search, edit, and confirmed-delete journey works through the browser against the real API and SQLite database.
- Invalid input produces understandable errors without corrupting stored data.
- Automated tests and the documented Docker smoke test pass from a clean start.

---
*Requirements defined: 2026-08-13*
*Last updated: 2026-08-13 after initialization completion*
