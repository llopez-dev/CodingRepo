# 05. RESTful API Design Best Practices — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 04](04-repository-unit-of-work.md). Build-it project — contracts, test requests, and
> checklists only. No C# solution code.

## Project Brief

Every endpoint you've built so far works. This path is about the difference between "works" and
"pleasant to actually consume": you'll stop returning EF entities directly over the wire (finally
retiring the circular-reference workaround from Path 03 for good), introduce real request/response
DTOs, replace the ad-hoc `take` from Path 02 with real pagination, extend filtering, add sorting —
and, like Path 04, you'll do the boring/manual version of mapping first so you actually feel what a
mapping library buys you before reaching for one.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 04](04-repository-unit-of-work.md)
      are fully done. In particular, books and authors are persisted through EF Core behind
      repositories and a unit of work, and `GET /api/books` already supports `?genre=` and
      `?take=`.

## Rules

- No further changes to persistence or the repository/unit-of-work layer from Path 04 — this path
  only changes the **wire-facing shapes** (DTOs) and **query string handling**. If you find
  yourself editing `IBookRepository` or `IUnitOfWork`, stop and reconsider.
- Write the entity ↔ DTO mapping **by hand first** (M1) before introducing any mapping library
  (M2) — same "feel it, then evaluate" approach as Path 04's repositories.
- No FluentValidation — keep the manual validation approach; a fully consistent error response
  shape is Path 08's job, not this one.
- By the end of this path, **no EF entity should ever be serialized directly** as an HTTP response
  body anywhere in the project. If you can still find one, that's not done yet.

## DTO Contract

| DTO | Fields | Used by |
|---|---|---|
| `AuthorResponse` | `id`, `name` | `GET /api/authors`, `GET /api/authors/{id}`, nested inside `BookResponse` |
| `AuthorWithBookCountResponse` | `id`, `name`, `bookCount` | `GET /api/authors` (the Path 03 M9 projection) |
| `BookResponse` | `id`, `title`, `publishedYear`, `genre`, `authorId`, `author` (an `AuthorResponse`) | All book read endpoints |
| `BookCreateRequest` | `title`, `publishedYear`, `genre`, `authorId` | `POST /api/books` |
| `BookUpdateRequest` | Same shape as `BookCreateRequest` | `PUT /api/books/{id}` |
| `PagedResult<T>` | `items` (array of `T`), `page`, `pageSize`, `totalCount`, `totalPages` | Any paginated list endpoint |

Example `BookResponse`:

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

Example paginated list response for `GET /api/books`:

```json
{
  "items": [
    { "id": 4, "title": "Dune", "publishedYear": 1965, "genre": "Science Fiction", "authorId": 2, "author": { "id": 2, "name": "Frank Herbert" } }
  ],
  "page": 1,
  "pageSize": 10,
  "totalCount": 23,
  "totalPages": 3
}
```

## Query Contract for `GET /api/books`

| Parameter | Default | Rule |
|---|---|---|
| `page` | `1` | Integer, must be `>= 1`; `< 1` → `400`. |
| `pageSize` | `BookCatalog:DefaultPageSize` (from Path 02) | Integer, `1..MaxPageSize`; above `MaxPageSize` is clamped, `<= 0` → `400`. |
| `genre` | none | Same case-insensitive exact-match filter as Path 01. |
| `authorId` | none | Exact match; formalizes the Path 03 stretch-goal filter. |
| `publishedYearMin` / `publishedYearMax` | none | Inclusive range filter; `min > max` → `400`. |
| `sortBy` | `id` | One of `id`, `title`, `publishedYear`, `genre`; anything else → `400` listing the allowed values. |
| `sortDir` | `asc` | One of `asc`, `desc`; anything else → `400`. |

All filters combine with **AND** logic when more than one is supplied. Sorting is always applied
**before** paginating, and filtering always happens **before** sorting.

## Suggested Project Structure

Additions on top of Paths 01–04:

- [ ] A `Contracts` (or `Dtos`) folder holding every type from the [DTO Contract](#dto-contract).
- [ ] A generic `PagedResult<T>` wrapper type.
- [ ] A single type bundling all of `GET /api/books`'s query parameters (look up how Minimal APIs
      let you bind several query parameters into one type in a single step) instead of six
      separate method parameters.
- [ ] A hand-written mapping layer (M1), later reimplemented with a mapping library (M2) —keep
      both versions in your git history rather than only the final one, so the comparison in M2
      is actually visible later.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Manual DTOs and Mapping (Feel the Boilerplate First)

- [ ] Define every DTO from the [DTO Contract](#dto-contract).
- [ ] Write the entity ↔ DTO mapping **by hand** — plain code you write and can step through in a
      debugger, no library.
- [ ] Replace every place that currently returns/accepts an EF entity directly with the matching
      DTO instead.
- [ ] This directly and permanently retires the Path 03 M5 circular-reference workaround: a DTO
      that simply never includes the back-navigation can't cycle, by construction. Remove whatever
      cycle workaround you applied there — you shouldn't need it anymore.
- [ ] Re-run the full Path 01–04 regression pass against the new DTO-shaped responses/requests.

### M2 — Evaluate: Introduce a Mapping Library

- [ ] Look up the difference between a reflection/convention-based mapper (e.g. AutoMapper) and a
      compile-time source-generator mapper (e.g. Mapperly). Pick one.
- [ ] Reimplement the same mappings from M1 using the library instead of your hand-written code.
- [ ] Compare, concretely, using your own two implementations sitting in your git history:

| Dimension | What to actually check |
|---|---|
| Amount of code | How many lines did the library save you, if any? |
| Discoverability | Can you find exactly what maps to what by reading the code, or do you have to run it to find out? |
| Failure mode | If you add a new DTO field and forget to map it, does it fail at compile time, at startup, or silently at runtime with a default value? |
| Debuggability | Can you put a breakpoint on the actual mapping logic? |

- [ ] Write a short note (a paragraph is enough — add it to your Path 04 ADR log if you want to
      keep that habit going, or a new short one) stating which approach you're keeping for this
      project's current size, and why.

### M3 — Resource Naming Audit

- [ ] Go through every endpoint you've built across Paths 01–04 and check each one against this
      checklist:

| Check | What to look for |
|---|---|
| Collections are plural nouns | `/api/books`, `/api/authors` — not `/api/book`. |
| No verbs in the path | The HTTP verb already says what's happening; the URL should name a resource, e.g. `DELETE /api/books/{id}`, not `/api/books/{id}/delete`. |
| Consistent casing | Pick one casing convention for path segments and query parameters and use it everywhere. |
| Sub-resources read like a hierarchy | e.g. a resource that only ever exists under another would nest under it in the path. |
| Query parameters, not path segments, for optional filters/sorting/paging | `?genre=` not `/genre/{genre}`. |

- [ ] Specifically revisit `POST /api/authors/with-first-book` from Path 04 against this
      checklist — it doesn't name a resource, it names an action. Decide: keep it as a deliberate,
      pragmatic exception (a genuine command that doesn't map cleanly to a single resource is a
      common, accepted reason to break pure REST naming), or restructure it. Either is fine — just
      write down which you picked and why, the same way you did in Path 04's ADR.

### M4 — Real Pagination

- [ ] Add `page` and `pageSize` to `GET /api/books`, replacing the Path 02 `take`-only approach,
      per the [Query Contract](#query-contract-for-get-apibooks).
- [ ] Return the `PagedResult<BookResponse>` shape, including accurate `totalCount` and
      `totalPages`.
- [ ] Reuse the **exact same** `BookCatalog:DefaultPageSize` / `MaxPageSize` configuration values
      from Path 02 as `pageSize`'s default and upper bound — you already built the configuration
      plumbing for this in Path 02; this milestone just points a new parameter at it.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| `page=1`, no `pageSize` | `pageSize` equals the configured default; `totalCount`/`totalPages` are correct for the full data set. |
| `page` beyond the last real page (e.g. `page=999`) | `200` with an **empty** `items` array and still-correct `totalCount`/`totalPages` — not `404`. |
| `page=0` or negative | `400`. |
| `pageSize=0` or negative | `400`. |
| `pageSize` above `MaxPageSize` | Clamped down, same as Path 02's `take` behavior. |

### M5 — Extend Filtering

- [ ] Formalize the existing `genre` filter and add `authorId` (the Path 03 stretch goal) as an
      officially documented, tested filter.
- [ ] Add `publishedYearMin` / `publishedYearMax` as an inclusive range filter — your first filter
      that isn't a simple exact match.
- [ ] Confirm multiple filters combine with **AND**, not **OR** — e.g. `genre` + `authorId`
      together should narrow the results, not widen them.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| `genre` + `authorId` both supplied, both match some books but not the same ones | Only books matching **both** are returned. |
| `publishedYearMin` greater than `publishedYearMax` | `400`. |
| A filter combination matching nothing | `200` + empty `items`, correct `totalCount: 0`. |

### M6 — Sorting

- [ ] Add `sortBy` (`id`, `title`, `publishedYear`, `genre`) and `sortDir` (`asc`, `desc`) per the
      [Query Contract](#query-contract-for-get-apibooks).
- [ ] Reject an invalid `sortBy` or `sortDir` with `400` that **names the allowed values** — don't
      silently fall back to a default when the client clearly asked for something you don't
      support.
- [ ] Default to sorting by `id` ascending when `sortBy` isn't supplied. This isn't just a
      placeholder — **pagination without a deterministic sort order is unreliable**: two requests
      for "page 2" against a data set with no stable ordering can return overlapping or missing
      rows if the underlying order happens to differ between calls. Always sort by something
      stable, even when the caller didn't ask for a particular order.
- [ ] Combine sorting with filtering and pagination in a single request and confirm the pipeline
      order documented in the [Query Contract](#query-contract-for-get-apibooks) (filter → sort →
      paginate) is what's actually happening.

### M7 — Full DTO Coverage for Authors Too

- [ ] Apply `AuthorResponse` / `AuthorWithBookCountResponse` to every author-related endpoint,
      including the `POST /api/authors/with-first-book` request/response shapes from Path 04.
- [ ] Search your entire project for anywhere an EF entity type is still the declared return type
      or parameter type of an endpoint. There should be none left.

### M8 — Full Regression + Combined-Query Verification

- [ ] Full regression pass across every Path 01–05 scenario.
- [ ] Run at least one request that combines **every** query feature at once (a filter, a
      `publishedYearMin`/`Max` range, a `sortBy`/`sortDir`, and `page`/`pageSize`) and manually
      verify the result against what you'd expect by reasoning about the underlying data.

## Manual Test Script

Run in order. Assumes a reasonable spread of existing books/authors from earlier paths; add more
via `POST` first if you don't have enough to make pagination/sorting meaningfully observable
(aim for at least 15 books across at least 2 genres and 2 authors).

```http
@baseUrl = https://localhost:5001

### 1. Default paged list -> pageSize matches configured default, totalCount/totalPages correct
GET {{baseUrl}}/api/books

### 2. Explicit page/pageSize
GET {{baseUrl}}/api/books?page=2&pageSize=5

### 3. Page beyond the data -> 200 + empty items, correct totalCount/totalPages
GET {{baseUrl}}/api/books?page=999

### 4. Invalid page -> 400
GET {{baseUrl}}/api/books?page=0

### 5. Invalid pageSize -> 400
GET {{baseUrl}}/api/books?pageSize=-1

### 6. Combined filters (AND) -> only books matching both
GET {{baseUrl}}/api/books?genre=fantasy&authorId=2

### 7. Published-year range filter
GET {{baseUrl}}/api/books?publishedYearMin=1900&publishedYearMax=1970

### 8. Invalid range (min > max) -> 400
GET {{baseUrl}}/api/books?publishedYearMin=2000&publishedYearMax=1900

### 9. Sort by title descending
GET {{baseUrl}}/api/books?sortBy=title&sortDir=desc

### 10. Invalid sortBy -> 400 naming the allowed values
GET {{baseUrl}}/api/books?sortBy=nonsense

### 11. Invalid sortDir -> 400
GET {{baseUrl}}/api/books?sortBy=title&sortDir=sideways

### 12. Everything combined at once
GET {{baseUrl}}/api/books?genre=fantasy&publishedYearMin=1900&sortBy=publishedYear&sortDir=asc&page=1&pageSize=5

### 13. Author list still returns DTOs, no raw entity, no book-count regression
GET {{baseUrl}}/api/authors

### 14. Single book response still includes nested author, now via DTO not the raw entity
GET {{baseUrl}}/api/books/1
```

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Add `Location`-style navigation links to `PagedResult<T>` (`nextPage`/`previousPage` URLs)
      instead of just numeric metadata, and decide whether that's worth the extra complexity for
      this API's actual consumers.
- [ ] Add a `fields=` (sparse fieldset) query parameter letting a client request only specific
      `BookResponse` fields, and think through why this interacts awkwardly with a fixed C# DTO
      shape.
- [ ] Try Mapperly (if you picked AutoMapper in M2) or vice versa, purely to compare the developer
      experience of both approaches on the exact same mappings.
- [ ] Add API examples (sample request/response bodies) directly as XML doc comments on your DTOs
      — you'll wire these into generated documentation in Path 06.

## Definition of Done

- [ ] M1–M8 all checked off, in order, each with manual test evidence.
- [ ] No EF entity is returned or accepted directly by any endpoint anywhere in the project.
- [ ] `GET /api/books` supports `page`, `pageSize`, `genre`, `authorId`,
      `publishedYearMin`/`Max`, `sortBy`, and `sortDir`, all combinable, matching the
      [Query Contract](#query-contract-for-get-apibooks) exactly.
- [ ] Pagination beyond the last page returns `200` with empty results, never `404`.
- [ ] An invalid `sortBy`/`sortDir` is rejected with `400` naming the allowed values, not silently
      ignored.
- [ ] The resource-naming audit from M3 is complete, including an explicit decision about
      `POST /api/authors/with-first-book`.
- [ ] A written comparison exists (M2) between the hand-written mapping and the library-based one,
      with a stated decision.
- [ ] The full Path 01–04 regression pass still succeeds against the new DTO-shaped contract.

## Self-Review Checklist

- [ ] You can explain, from your own git history, exactly what the mapping library saved you
      versus your hand-written version — not a general opinion, your own before/after.
- [ ] You can point to the exact line where a default, deterministic sort order is applied even
      when the caller didn't request one, and explain why it's there.
- [ ] Every 400 your new query parameters produce actually tells the caller what was wrong,
      instead of a generic unhelpful message.
- [ ] Filtering, sorting, and pagination are applied in the documented order everywhere, not
      accidentally reordered in one endpoint versus another.
- [ ] Nothing in your naming audit was skipped just because "it already works."

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `feat(m4): real pagination replacing take-based limiting`.
- [ ] Keep the M1 (manual mapping) commit separate from the M2 (library-based mapping) commit —
      you want both visible in history for the comparison to mean anything later.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 06 documents this
      exact contract.

## Reference Docs (use only when stuck)

DTOs and mapping:
- [Use DTOs to shape API responses (design guidance)](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/microservice-domain-model)
- [AutoMapper documentation](https://docs.automapper.org/en/stable/)
- [Mapperly documentation](https://mapperly.riok.app/)

API design conventions:
- [ASP.NET Core web API design guidance](https://learn.microsoft.com/aspnet/core/web-api/)
- [Microsoft REST API guidelines](https://github.com/microsoft/api-guidelines)

Pagination, filtering, sorting:
- [Minimal API parameter binding (`[AsParameters]`)](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/parameter-binding)
- [Sorting, filtering, and paging guidance (Microsoft REST API guidelines)](https://github.com/microsoft/api-guidelines/blob/vNext/graph/patterns/list-operations.md)
