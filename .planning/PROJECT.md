# Book Tracker

## What This Is

Book Tracker is a full-stack web application for maintaining a searchable personal book database. Users can add, view, edit, delete, and search books using a React frontend backed by a Python FastAPI API and SQLite database.

The project is also a structured learning exercise for practicing professional software development workflow with GSD, including requirements, phased planning, testing, containerization, and verification.

## Core Value

Users can reliably find and maintain accurate book records through a simple end-to-end web application.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Users can add books with title, author, publication year, genre, and ISBN.
- [ ] Users can view all books and search the collection.
- [ ] Users can edit and delete existing books.
- [ ] The React frontend communicates with a FastAPI backend backed by SQLite.
- [ ] The application runs consistently through Docker.
- [ ] The project demonstrates a professional GSD development workflow.

### Out of Scope

- User accounts and authentication — not needed for the initial single-user learning application.
- External book metadata imports — manual entry keeps the first release focused on CRUD fundamentals.
- Ratings, reviews, lending workflows, covers, and social features — useful future extensions but not required to validate the core database workflow.
- Production-scale hosting and managed databases — Docker and SQLite are sufficient for the initial learning deployment.
- Native mobile applications — the initial product is web-first.

## Context

This is a greenfield application with an intentionally small, explicit stack:

- Backend: Python FastAPI.
- Database: SQLite initially.
- Frontend: React.
- Deployment: Dockerized application.

The main learning objective is to build a complete vertical slice while practicing professional workflow habits: clear requirements, small phases, automated tests, API/UI integration, containerized execution, and verification against observable user behavior.

## Constraints

- **Technology**: Use FastAPI, SQLite, React, and Docker — these technologies are explicit learning goals.
- **Scope**: Keep the first release focused on book CRUD, listing, and search — avoid premature product features.
- **Data**: Each book must contain title, author, publication year, genre, and ISBN — these fields define the initial domain model.
- **Deployment**: The application must run in Docker — reproducible local execution is part of the learning outcome.
- **Workflow**: Use GSD artifacts and phase gates — the project is intended to teach a professional development process, not only produce code.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| FastAPI for the backend | Explicit user requirement and a lightweight fit for a CRUD API | — Pending |
| SQLite initially | Minimal operational overhead while learning database-backed application development | — Pending |
| React for the frontend | Explicit user requirement and a clear client/API integration boundary | — Pending |
| Dockerized deployment | Makes the full-stack app reproducible across environments | — Pending |
| Vertical MVP structure | Delivers an end-to-end working application early and supports incremental learning | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-08-13 after initialization*
