# Feature Landscape

**Domain:** Personal book database / small book CRUD and search application  
**Project:** Book Tracker  
**Researched:** 2026-08-13  
**Scope recommendation:** Keep v1 to one local user, five required book fields, list/search, and reliable CRUD. Do not turn the first release into a reading-management or social product.

## Product Shape

The expected v1 loop is: **open collection → scan or search → add a record → inspect it in the list → edit it later → delete it deliberately**. The list is the primary surface; add/edit should use the same field model and validation rules. Search should be useful on a collection of dozens or hundreds of books without requiring advanced query syntax.

## Table Stakes

Features users expect. Missing = the product does not validate its core value.

| Feature | Why Expected | Complexity | Dependencies | Acceptance considerations |
|---------|--------------|------------|--------------|---------------------------|
| Add book form | The collection has no value without a dependable way to create records. | Med | API create endpoint, database schema | Form exposes title, author, publication year, genre, ISBN; successful save returns the new book and refreshes or updates the list; failed save preserves entered values. |
| Edit existing book | Bibliographic data is often corrected after entry. | Med | Stable book ID, API update endpoint, shared form validation | Edit is pre-filled; save changes only the intended record; success is visible; cancel leaves data unchanged. |
| Delete a book | A personal database needs correction and cleanup. | Low/Med | Stable book ID, API delete endpoint | Delete action identifies the book; requires a specific confirmation such as “Delete *Title*?”; cancel does nothing; success removes it from the list and reports completion. No silent or accidental deletion. |
| Collection list | Users need an immediate, browsable view of all records. | Med | API list endpoint, loading/error states | Shows all five fields or a compact but complete row/card; has loading, empty, and failure states; each row exposes edit/delete actions; ordering is deterministic (recommend title ascending, with a documented tie-breaker). |
| Search collection | Finding one record is the core value, not merely storing records. | Med | List/query API support and indexed/searchable fields | One search input filters by title and author at minimum; case-insensitive and substring matching; blank query restores all books; “no matches” is distinct from “no books”; clearing search is obvious. |
| Clear success/error feedback | CRUD operations otherwise feel unreliable. | Low | Consistent API error shape and UI notification area | Show field-level validation errors near the field, a general server/network error when needed, and a success message after create/update/delete; do not report success before the API confirms it. |
| Shared create/edit behavior | Divergent rules create records that cannot later be edited consistently. | Low/Med | Shared frontend schema and backend model validation | Create and update accept the same field formats and normalize values identically; server remains authoritative even if client validation is bypassed. |
| Accessible, labeled forms | Required-field and error behavior must work for keyboard and assistive-technology users. | Low/Med | Semantic React form controls | Every input has a visible label; required fields are identified; errors are textual and associated with controls; focus moves to the first invalid field or an error summary. |

### Required field contract

Treat all five fields as required for v1, matching the project requirement. Reject blank or whitespace-only values after trimming. Do not silently invent metadata or accept partial records.

| Field | Recommended v1 validation and normalization | Behavior to accept/reject |
|-------|---------------------------------------------|--------------------------|
| **Title** | Required string; trim leading/trailing whitespace; reject empty; cap at a practical maximum (recommend 255 characters). Preserve internal punctuation and capitalization. | Accept normal titles, punctuation, subtitles, and non-ASCII text. Reject only blank/oversized values; do not enforce an artificial title grammar. |
| **Author** | Required string; trim; recommend 255-character maximum. Store as entered rather than forcing `Last, First`, because names and group authors vary. | Accept multiple authors, initials, punctuation, diacritics, and “Various authors.” Empty is invalid. |
| **Publication year** | Required integer, not free-form text after parsing. Recommend range `0..current year` for ordinary books, with a deliberate allowance for year `0` only if the product wants BCE/unknown-era records; simpler v1 choice is `1..current year`. | Reject decimals, alphabetic text, future years, and implausible short/long values. Decide and document whether reprints use original publication year; do not accept a full date in a year field. |
| **Genre** | Required short string; trim; recommend 100-character maximum. Use free text in v1 rather than a closed dropdown, because genre taxonomies are subjective and users may need “Historical fiction,” “Essays,” etc. | Accept a user’s genre label. Reject blank/oversized input. A later controlled vocabulary can be added without making v1’s form brittle. |
| **ISBN** | Required string, not numeric storage: preserve leading zeroes and permit hyphens/spaces on input. Normalize for comparison by removing separators; validate 10- or 13-digit ISBN structure and check digit. Store a canonical display value (recommend ISBN-13 when conversion is implemented; otherwise normalized entered form). | Accept ISBN-10 and ISBN-13 with common separators; reject wrong length, non-digit characters other than separators, and invalid check digits. Do not silently convert a malformed value. ISBN-10 `X` is valid only in the check-digit position. |

ISBN is an identifier, not a safe universal uniqueness key: different editions/formats can have different ISBNs, and older records may lack one. For this intentionally small v1, recommend a **soft duplicate warning** when normalized ISBN already exists, not a hard rejection. If the learning goal favors strict data integrity, make ISBN unique only after confirming the product supports one record per edition; never make title+author unique.

### Workflow acceptance baseline

- **Add:** Submit is disabled while saving (or guarded against double-submit); API errors leave the form populated; on success, return to the list or show the new record immediately.
- **Edit:** The record ID comes from the route/action context, not a title lookup; a failed update must not overwrite the displayed data with optimistic state.
- **Delete:** Confirmation names the title and uses an explicit destructive action label. A toast alone is insufficient if the record disappears without context; provide a short-lived undo only if it is cheap, otherwise document permanent deletion.
- **List:** Loading, empty collection, no search matches, and API failure are separate states. Avoid a blank screen and avoid showing stale “all books” while a new search is in flight without an indicator.
- **Search:** Search at minimum title and author, case-insensitively. Genre and ISBN search are useful and inexpensive, so include them if the API contract remains simple; do not require fuzzy ranking, relevance scoring, or a search DSL in v1.

These validation and feedback recommendations align with W3C guidance: validate both client and server side, identify errors in text, be forgiving of input formats, and confirm or provide undo for irreversible actions (confidence: LOW for this research run because external provider access was unavailable). NN/g similarly recommends specific, non-routine deletion confirmations rather than generic “Are you sure?” prompts (LOW confidence).

## Differentiators

Useful additions that improve a small personal catalog without changing its identity. Select at most one for v1; the rest belong after CRUD is proven.

| Feature | Value proposition | Complexity | Dependencies | Recommendation |
|---------|-------------------|------------|--------------|----------------|
| Sort by title, author, year, or genre | Makes browsing practical as the collection grows. | Low/Med | Stable list query and deterministic ordering | **Best v1 differentiator** if list implementation is already clean; default to title and add simple sort controls. |
| Search across title, author, genre, and ISBN | Users remember different facts about a book. | Low/Med | Normalized ISBN and API search semantics | Include as a small extension to table-stakes search, provided it does not complicate the UI. |
| Soft duplicate warning | Prevents accidental duplicate entry while preserving legitimate different editions. | Med | Normalized ISBN query and non-blocking warning UI | Recommended over strict uniqueness for a personal catalog. |
| Detail view | Gives a focused read-only view and a clear edit/delete entry point. | Low/Med | Get-by-ID endpoint or list payload | Defer if rows/cards already show all fields; add when the list becomes crowded. |
| Keyboard-friendly quick add | Speeds repetitive manual entry. | Low/Med | Accessible form and reliable focus handling | Defer until the basic form is stable; do not add a modal workflow first. |
| Import/export CSV or JSON | Protects data and enables migration. | Med/High | Stable schema, file validation, duplicate policy | Valuable after v1, but explicitly post-v1 because it multiplies validation and error cases. |

## Anti-Features

Features to explicitly **not** build in the focused v1.

| Anti-feature | Why avoid | What to do instead |
|--------------|------------|-------------------|
| User accounts, authentication, sharing | Project is single-user; adds security, session, and authorization work unrelated to CRUD learning goals. | Run as a local/private app with no account layer. |
| External metadata lookup or ISBN auto-import | Creates API keys, rate limits, conflicting metadata, network failure, and edition-selection complexity. | Require manual entry; keep ISBN validation deterministic and local. |
| Ratings, reviews, reading status, shelves, lending | Expands the domain model and navigation before the collection workflow is validated. | Store only the five required bibliographic fields. |
| Cover images and file uploads | Adds storage, image processing, privacy, and broken-resource concerns. | Display text metadata only. |
| Advanced fuzzy/full-text search, facets, saved queries | Overkill for a small SQLite collection and risks hiding simple search semantics. | Case-insensitive substring search on core fields; add sorting later. |
| Bulk delete/edit | High consequence and requires selection state, partial-failure handling, and stronger recovery. | One-record actions with specific confirmation. |
| Hard uniqueness on title+author or ISBN without edition policy | Rejects legitimate editions, translations, formats, and duplicate personal copies. | Use generated ID; soft-warn on duplicate normalized ISBN. |
| Pagination, virtualized lists, or premature performance tuning | Complexity is unjustified for the expected small local collection. | Return all records initially with deterministic ordering; measure before adding limits. |
| Generic browser-only validation | Client validation can be bypassed and produces inconsistent API behavior. | Duplicate core validation in FastAPI/Pydantic and return structured field errors. |

## Feature Dependencies

```text
Database schema + generated book ID
  → server validation and CRUD API
    → list/loading/empty/error states
      → add/edit forms
      → delete confirmation
      → search and sorting

Normalized ISBN validation
  → ISBN search
  → duplicate ISBN warning

Consistent API error shape
  → field-level form errors
  → reliable save feedback
```

## MVP Recommendation

Prioritize:

1. A single shared add/edit form with required title, author, publication year, genre, and ISBN fields.
2. Server-authoritative validation, including ISBN-10/ISBN-13 check digits, year bounds, trimming, and useful field errors.
3. Deterministic collection list with loading, empty, no-results, and failure states.
4. Case-insensitive search across title and author; include genre and normalized ISBN if the query remains one simple input.
5. Edit and explicitly confirmed single-record delete with clear success/failure feedback.
6. **One lightweight differentiator:** simple sort controls or a non-blocking duplicate ISBN warning. Choose sorting first if implementation simplicity is the priority.

Defer: accounts, imports, covers, ratings/reviews, lending/reading status, bulk operations, pagination, advanced search, and strict duplicate policy. These are not needed to validate “reliably find and maintain accurate book records.”

## Sources

- [W3C WAI, Validating Input](https://www.w3.org/WAI/tutorials/forms/validation/) — required fields, forgiving formats, client/server validation, accessible errors, confirmation/undo guidance. **Confidence: LOW in this run** (official source fetched, but research-provider confidence seam classified webfetch as LOW).
- [Nielsen Norman Group, Confirmation Dialogs Can Prevent User Errors—If Not Overused](https://www.nngroup.com/articles/confirmation-dialog/) — specific confirmation for destructive actions; avoid routine generic prompts. **Confidence: LOW in this run**.
- Project context: `.planning/PROJECT.md` — authoritative v1 scope and explicit out-of-scope items. **Confidence: HIGH**.

## Research Gaps

- Competitor feature comparison was not independently verified because the configured Brave search provider had no API key and a representative catalog site blocked fetching. Revisit only if the product expands beyond the focused CRUD MVP.
- Decide during API phase whether publication year permits `0`/BCE values and whether ISBN canonical display should convert valid ISBN-10 to ISBN-13. Either choice is acceptable if consistently documented and tested.
