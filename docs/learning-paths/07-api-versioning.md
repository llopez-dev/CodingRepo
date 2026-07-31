# 07. API Versioning — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 06](06-openapi-documentation.md). Build-it project — contracts, test requests, and
> checklists only. No C# solution code.

## Project Brief

Every path so far has changed the API in place. Real APIs can't always do that — once clients
depend on a shape, changing it under them breaks their code. This path makes one genuine,
deliberate **breaking change** to the Book Catalog API (how a book's genre is represented) and
ships it as a `v2`, while `v1` keeps working exactly as it does today. You'll pick a versioning
strategy, prove both versions actually work side by side against the **same** underlying data, and
deprecate `v1` properly instead of just leaving two confusing copies of the API lying around.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 06](06-openapi-documentation.md) are
      fully done — DTOs, pagination/filtering/sorting, and accurate OpenAPI documentation are all
      in place before you introduce a second version of anything.

## Rules

- **No changes to the underlying database or domain model.** `Book` still has a single `Genre`
  column underneath. The array shape in `v2` is a wire-level (DTO) change only — both versions
  read and write the same data.
- **`v1` must remain 100% functionally identical** to how it behaved at the end of Path 06. If a
  `v1` regression check fails anywhere in this path, that's a bug you introduced, not an
  acceptable side effect of adding `v2`.
- No new persistence/repository features. Reuse Path 04's repositories and unit of work
  unchanged — only the DTOs, mapping, and routing differ between versions.
- Don't silently remove or break the unversioned routes you've been using since Path 01 — decide
  deliberately what happens to them (see M1) and document the decision.

## Versioning Contract

The one deliberate breaking change for this path, and everything that follows from it:

| Aspect | `v1` (`/api/v1/books`) | `v2` (`/api/v2/books`) |
|---|---|---|
| Genre field | `genre: string` | `genres: string[]` — exactly one element today, but a real array, deliberately designed for a future multi-genre feature so *that* change won't need another version bump |
| Genre filter | `?genre=` exact match (unchanged from Path 01) | `?genres=` match-**any**-of; accepts one or more values |
| `sortBy=genre` | Allowed (unchanged from Path 05) | **Removed** — sorting by an array-valued field is ambiguous; not supported in `v2` |
| Status | Deprecated once `v2` ships — still fully functional | Current |

Example `v1` book response (identical to Path 05/06):

```json
{
  "id": 4,
  "title": "Dune",
  "publishedYear": 1965,
  "genre": "Science Fiction",
  "authorId": 2,
  "author": { "id": 2, "name": "Frank Herbert" }
}
```

Example `v2` book response — same underlying book, different wire shape:

```json
{
  "id": 4,
  "title": "Dune",
  "publishedYear": 1965,
  "genres": ["Science Fiction"],
  "authorId": 2,
  "author": { "id": 2, "name": "Frank Herbert" }
}
```

Author endpoints are unaffected by this change, but still get versioned routes
(`/api/v1/authors`, `/api/v2/authors`) with **identical** behavior in both — a deliberate example
of the fact that bumping the API's version doesn't mean every resource has to change.
`POST /api/authors/with-first-book` carries a nested book, so its `v2` route must accept `genres`
in that nested payload too, consistently with the rest of `v2`.

## Suggested Project Structure

Additions on top of Paths 01–06:

- [ ] The API versioning package/tooling for ASP.NET Core (look up the current recommended one —
      the original `Microsoft.AspNetCore.Mvc.Versioning` package was handed off to the community
      and now lives under a different package name; find what's current for your SDK).
- [ ] Separate `v1`/`v2` request/response DTO types for `Book` (everything else can stay shared).
- [ ] A clear separation between versioned route registration (`/api/v{version}/...`) and the
      shared repository/unit-of-work layer underneath — the routing/DTO layer should be the only
      thing that knows a version number exists.
- [ ] A small piece of middleware or a filter that adds deprecation headers to every `v1` response.
- [ ] A `MIGRATION.md` (or similar) file for the consumer migration notes you'll write in M8.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Pick and Wire Up a Versioning Strategy

- [ ] Add API versioning tooling to the project.
- [ ] Decide: URL-based (e.g. `/api/v1/...`, `/api/v2/...`) or header/media-type-based. Implement
      **URL-based** as your primary strategy for this path — it's the most visible, the easiest to
      test by hand, and the easiest to browse in the OpenAPI UI from Path 06. You'll compare it
      against a header-based approach later (M7).
- [ ] Move every existing endpoint under `/api/v1/...` **without changing any behavior**.
- [ ] Decide, explicitly, what happens to the old unversioned routes (`/api/books`, etc.) that
      every earlier path's test script has been using: keep them working as an alias for `v1` for
      now (recommended — nothing that existed before this path should silently start failing), or
      retire them as their own small breaking change communicated the same way as everything else
      in this path. Write down which you picked.

**Edge case to verify:** run your entire Path 01–06 manual test scripts, unmodified, against
whichever routes you decided should still answer them, and confirm nothing broke.

### M2 — Design the `v2` Breaking Change (Before Writing Any Code)

- [ ] Confirm you understand and agree with every row of the
      [Versioning Contract](#versioning-contract) above before touching any code — versioning
      forces you to think the contract all the way through up front, unlike a same-version change
      you can still quietly adjust later.
- [ ] Decide exactly how `?genres=` will accept multiple values (repeated query parameters,
      comma-separated, or both) and write that decision down before implementing it.

### M3 — Implement `v2` Book Endpoints Alongside `v1`

- [ ] Add `/api/v2/books` and `/api/v2/books/{id}` (plus create/replace/delete) returning and
      accepting `genres: string[]` per the [Versioning Contract](#versioning-contract), mapping
      to/from the **same** single-`Genre`-column data as `v1` — no database changes.
- [ ] Add `/api/v2/authors` and related routes with identical behavior to `v1`, and update
      `/api/v2/authors/with-first-book` to expect `genres` in its nested book payload.
- [ ] Re-run the full `v1` regression suite and confirm it's still completely unaffected by `v2`
      existing alongside it.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Create a book via `v2` with `genres: ["Fantasy"]` | `201`, response shows `genres: ["Fantasy"]`, not `genre`. |
| Read that same book via `v1` afterward | `200`, response shows `genre: "Fantasy"` — same underlying data, different shape. |
| Create a book via `v2` with `genres: []` (empty array) | Decide and document: is at least one genre still required? Be consistent with `v1`'s "genre is required" rule. |
| Create a book via `v2` with `genres` containing more than one value | Decide and document what your API does today, given the domain model only stores one genre — reject it as unsupported (`400`), or silently only keep one? An explicit `400` is more honest than silently dropping data. |

### M4 — `v2` Filtering and Sorting Semantics

- [ ] Implement `?genres=` on `v2`'s book list endpoint with match-**any**-of semantics, per your
      M2 decision on how multiple values are supplied.
- [ ] Implement the `sortBy=genre` removal in `v2`: reject it with `400` naming the still-supported
      sort fields, exactly like Path 05's existing invalid-`sortBy` behavior — don't silently fall
      back to a default.
- [ ] Confirm `v1`'s `?genre=` exact-match and `sortBy=genre` are **completely unchanged**.

### M5 — Shared Core, Divergent Edges

- [ ] Audit your own implementation: confirm `v1` and `v2` call into the **same** repository and
      unit-of-work layer from Path 04, and the only code that actually differs between versions is
      the DTOs and the mapping at the boundary.
- [ ] If you find yourself duplicating any actual business logic (validation rules, filtering
      logic beyond the genre shape itself, etc.) between the two versions' handlers, refactor so
      that shared logic lives in exactly one place, called from both versions.

### M6 — Deprecate `v1` Safely (Without Removing It)

- [ ] Add deprecation signaling to every `v1` response: look up the current HTTP headers used to
      communicate API deprecation and an eventual removal date, and add them consistently across
      all `v1` endpoints.
- [ ] Mark every `v1` operation as deprecated in the OpenAPI spec from Path 06 (`deprecated: true`
      or your tooling's equivalent), and confirm the generated UI visibly flags them.
- [ ] Confirm `v1` is still **fully functional** after this milestone — deprecation is a signal to
      consumers, not a shutdown switch.

**Edge case to verify:** fetch the OpenAPI document and confirm deprecation flags appear on every
`v1` operation and on **none** of the `v2` operations.

### M7 — Try the Alternative Versioning Strategy

- [ ] Implement header-based (or media-type-based) versioning for at least one endpoint, as a
      side-by-side comparison — it doesn't need to replace the URL-based approach everywhere, just
      enough to genuinely compare.
- [ ] Fill in a comparison based on what you actually experienced, not general opinion:

| Dimension | URL-based | Header/media-type-based |
|---|---|---|
| Discoverability (can you tell the version just by looking at a request?) | | |
| Manual testing ease (can you just change the URL in a browser?) | | |
| Caching behavior (does an HTTP cache need to know about the version too?) | | |
| Routing/implementation effort in your project | | |

- [ ] Decide, in writing, which strategy you're actually keeping for this project going forward,
      and why — based on your own filled-in table, not a general preference.

### M8 — Consumer Migration Notes

- [ ] Write a short "Migrating from v1 to v2" note (a real file in your project, e.g.
      `MIGRATION.md`) describing exactly what changed (the `genre` → `genres` shape, the filter
      change, the removed `sortBy` option) and what a consumer needs to do to move from `v1` to
      `v2`.
- [ ] Include the deprecation timeline language you settled on in M6 (even if illustrative — you're
      not really shutting `v1` down, but write it as if you were communicating this externally).

### M9 — Full Regression + New Contract Verification

- [ ] Full regression pass across every `v1` scenario from Paths 01–06, completely unchanged.
- [ ] Full pass across every new `v2` scenario from this path.
- [ ] Confirm deprecation headers and OpenAPI flags appear only on `v1`, never on `v2`.

## Manual Test Script

Run in order. Assumes existing seeded data from earlier paths (e.g. "Dune" by "Frank Herbert").

```http
@baseUrl = https://localhost:5001

### 1. Unversioned route still works (per your M1 decision) or intentionally retired - confirm which
GET {{baseUrl}}/api/books

### 2. v1 full regression - list, filter, sort all unchanged
GET {{baseUrl}}/api/v1/books?genre=science%20fiction&sortBy=genre&sortDir=asc

### 3. v1 response headers show deprecation signaling
GET {{baseUrl}}/api/v1/books/4

### 4. v2 read of the same book -> genres array, not genre
GET {{baseUrl}}/api/v2/books/4

### 5. v2 create with a single genre -> 201, response uses genres array
POST {{baseUrl}}/api/v2/books
Content-Type: application/json

{
  "title": "Children of Dune",
  "publishedYear": 1976,
  "genres": ["Science Fiction"],
  "authorId": 2
}

### 6. v2 create with an empty genres array -> 400 (per your M3 decision)
POST {{baseUrl}}/api/v2/books
Content-Type: application/json

{
  "title": "No Genre Book",
  "publishedYear": 2000,
  "genres": [],
  "authorId": 2
}

### 7. v2 filter by genres - match-any-of
GET {{baseUrl}}/api/v2/books?genres=Science%20Fiction

### 8. v2 sortBy=genre -> 400, names the still-supported fields
GET {{baseUrl}}/api/v2/books?sortBy=genre

### 9. v1 sortBy=genre still works, unaffected by v2's removal
GET {{baseUrl}}/api/v1/books?sortBy=genre

### 10. v2 authors/with-first-book expects genres in the nested book
POST {{baseUrl}}/api/v2/authors/with-first-book
Content-Type: application/json

{
  "author": { "name": "Ursula K. Le Guin" },
  "book": {
    "title": "A Wizard of Earthsea",
    "publishedYear": 1968,
    "genres": ["Fantasy"],
    "authorId": 0
  }
}

### 11. v1 authors/with-first-book still expects a single genre string
POST {{baseUrl}}/api/v1/authors/with-first-book
Content-Type: application/json

{
  "author": { "name": "Test Author V1" },
  "book": {
    "title": "Test Book V1",
    "publishedYear": 2000,
    "genre": "Mystery",
    "authorId": 0
  }
}
```

Manual (non-HTTP) step: fetch the OpenAPI document (Path 06) and confirm every `v1` operation is
flagged deprecated and every `v2` operation is not.

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Add a `v3` scaffold that changes nothing at all — just to prove your versioning setup makes
      adding a version that isn't a breaking change (or is entirely unused) cheap and safe.
- [ ] Add an actual `Sunset` date to `v1` and, purely as an exercise, write the code path that
      would make `v1` start returning `410 Gone` after that date — then decide whether you'd ever
      actually want that to run automatically in a real system versus being a manual, deliberate
      switch flipped by a person.
- [ ] Extend the `v2` genre change into real multi-genre support end-to-end (a genuine schema
      change via a new EF Core migration, per Path 03) and see how much of `v2`'s DTO layer already
      anticipated this versus how much still needs to change.
- [ ] Revisit your Path 04 ADR and Path 05 mapping-library note: does having two DTO shapes per
      resource change your earlier verdicts on repositories or mapping libraries?

## Definition of Done

- [ ] M1–M9 all checked off, in order, each with manual test evidence.
- [ ] `v1` behaves identically to its state at the end of Path 06, with deprecation signaling
      added on top but no functional changes.
- [ ] `v2` correctly represents genre as an array end-to-end (create, read, filter), backed by the
      exact same underlying single-genre data as `v1`.
- [ ] `sortBy=genre` is rejected with `400` in `v2` and still works in `v1`.
- [ ] `v1` and `v2` are proven to share the same repository/unit-of-work layer — no duplicated
      business logic between them.
- [ ] Deprecation headers and OpenAPI `deprecated` flags appear on every `v1` operation and no
      `v2` operation.
- [ ] A real migration note exists describing the `v1` → `v2` change for consumers.
- [ ] A documented decision exists comparing URL-based vs. header-based versioning, based on your
      own M7 comparison.

## Self-Review Checklist

- [ ] You can point to the exact place where `v1` and `v2` diverge, and confirm it's only DTOs and
      mapping — not repository or business logic duplicated in two places.
- [ ] You ran the *entire* pre-existing test suite from Paths 01–06 against `v1` after finishing
      this path, not just a couple of spot checks.
- [ ] You can explain, concretely, why the array-shaped `genres` field was chosen even though it
      holds exactly one value today.
- [ ] Your versioning-strategy decision in M7 is backed by your own filled-in comparison table, not
      a general opinion you already had going in.
- [ ] The migration note in M8 is something an actual external consumer could follow without
      needing to read your source code.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `feat(m3): add v2 book endpoints with genres array`.
- [ ] Keep the M1 "move everything under `/api/v1`" commit separate from the M3 "introduce `v2`"
      commits, so the history clearly shows "no behavior change" followed by "the actual breaking
      change," rather than one big tangled diff.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 08 adds proper
      validation and error handling on top of both versions.

## Reference Docs (use only when stuck)

API versioning:
- [ASP.NET Core API versioning (community-maintained package)](https://github.com/dotnet/aspnet-api-versioning)
- [API versioning conceptual overview (Microsoft REST API guidelines)](https://github.com/microsoft/api-guidelines/blob/vNext/graph/patterns/versioning.md)

Deprecation signaling:
- [RFC 8594: The Sunset HTTP Header Field](https://www.rfc-editor.org/rfc/rfc8594)

OpenAPI deprecation (ties back to Path 06):
- [OpenAPI Specification — deprecated field](https://spec.openapis.org/oas/latest.html)
