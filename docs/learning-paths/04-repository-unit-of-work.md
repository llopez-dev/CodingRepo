# 04. Repository & Unit of Work Patterns — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 03](03-entity-framework-core.md). Build-it project — contracts, test requests, and
> checklists only. No C# solution code.

## Project Brief

`DbContext` is already, more or less, a Unit of Work — and `DbSet<T>` is already, more or less, a
repository. So this path has a deliberately different shape from the others: you're going to build
the classic Repository + Unit of Work layer on top of EF Core anyway, **then use your own project
as evidence** to decide whether it actually earned its keep, or whether it's ceremony on top of an
abstraction that already existed.

You'll build a generic repository first and feel exactly where it breaks down, move to specific
repositories, add a real Unit of Work, prove it does something a direct `DbContext` call couldn't
(an atomic multi-entity operation), check whether it actually made anything easier to test, and
end by writing a short, real decision record — not a quiz answer — stating whether you're keeping
this layer or ripping it back out.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 03](03-entity-framework-core.md) are
      fully done. In particular, `Book` and `Author` are already persisted through EF Core, and
      Path 03's Definition of Done (persistence, nested author data, `AuthorId` validation,
      tracking audit, N+1 fix) is holding.

## Rules

- The **wire-level API contract stays the same** as Path 03 for every existing endpoint — this
  path only changes what sits *behind* the endpoints, with one deliberate exception (M5's new
  endpoint).
- No new persistence technology — still the same `DbContext` and provider from Path 03.
- No mocking library for M6 — write a fake/in-memory implementation of your own interfaces by
  hand. Mocking frameworks are Path 11; doing it by hand here keeps this path self-contained and
  actually teaches you what a mock would have done for you anyway.
- The generic repository you build in M1 is deliberately a **rough draft** you're likely to partly
  undo in M3. Don't over-invest in making it elegant — the point is to feel where it breaks down.
- Every milestone must still be manually tested with real HTTP requests before you check it off.

## Repository & Unit of Work Contract

### Generic repository (M1–M2 — temporary)

| Member | Purpose |
|---|---|
| `GetByIdAsync(id)` | Fetch a single entity by id, or `null`/not-found if it doesn't exist. |
| `ListAsync()` | Fetch every entity of that type. |
| `AddAsync(entity)` | Stage an insert. **Does not save.** |
| `Remove(entity)` | Stage a delete. **Does not save.** |

### Specific repositories (from M3 onward)

`IBookRepository`:

| Member | Purpose |
|---|---|
| `GetByIdAsync(id)` | Same as generic. |
| `ListAsync(genre?, take)` | The genre-filter + `take`-clamped list from Paths 01/02, now living here instead of directly in the endpoint handler. |
| `AddAsync(book)` / `Remove(book)` | Same as generic. |

`IAuthorRepository`:

| Member | Purpose |
|---|---|
| `GetByIdAsync(id)` | Same as generic. |
| `ListWithBookCountAsync()` | The single-query, N+1-free projection from Path 03 M9. |
| `ExistsAsync(id)` | Backing the `AuthorId` validation from Path 03 M7. |
| `HasBooksAsync(id)` | Backing the stretch-goal `409 Conflict` on author delete. |
| `AddAsync(author)` / `Remove(author)` | Same as generic. |

`IUnitOfWork`:

| Member | Purpose |
|---|---|
| `Books` | Exposes `IBookRepository`. |
| `Authors` | Exposes `IAuthorRepository`. |
| `SaveChangesAsync()` | The **only** place any repository call actually gets persisted. Everything staged through `Books`/`Authors` before this point is just tracked in memory. |

### New endpoint contract: `POST /api/authors/with-first-book`

The one wire-level addition in this path — an operation that only makes sense with a real unit of
work, because it must succeed or fail as a whole.

Request:

```json
{
  "author": { "name": "Ursula K. Le Guin" },
  "book": {
    "title": "A Wizard of Earthsea",
    "publishedYear": 1968,
    "genre": "Fantasy"
  }
}
```

Response on success — `201 Created`, `Location` pointing at the new author:

```json
{
  "author": { "id": 5, "name": "Ursula K. Le Guin" },
  "book": {
    "id": 10,
    "title": "A Wizard of Earthsea",
    "publishedYear": 1968,
    "genre": "Fantasy",
    "authorId": 5,
    "author": { "id": 5, "name": "Ursula K. Le Guin" }
  }
}
```

Both `author` and `book` follow the exact same field rules as the [Data Contract in
Path 03](03-entity-framework-core.md#data-contract). If either fails validation, the whole request
fails with `400` and **neither** the author nor the book is persisted.

## Suggested Project Structure

Additions on top of Paths 01–03:

- [ ] A temporary generic `IRepository<T>` (M1–M2), likely superseded by M3.
- [ ] `IBookRepository` / `IAuthorRepository` and their EF-Core-backed implementations.
- [ ] `IUnitOfWork` and its EF-Core-backed implementation.
- [ ] A hand-written fake/in-memory `IUnitOfWork` implementation used only by the M6 test.
- [ ] A short-lived scratch note (or your ADR draft from M8) capturing the pain points you hit in
      M2 — you'll need it later, don't rely on memory.
- [ ] A `docs/adr/` folder (or similar) for the decision record you write in M8.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Define the Repository Interfaces (Generic First)

- [ ] Build a generic `IRepository<TEntity>` matching the
      [Generic repository contract](#generic-repository-m1m2--temporary).
- [ ] Register `Book` and `Author` behind it (e.g. `IRepository<Book>`, `IRepository<Author>`),
      backed by your existing `DbContext`.
- [ ] Don't touch your endpoint handlers yet — this milestone only stands the generic layer up
      alongside what already exists.

### M2 — Feel the Pain: Reimplement Path 03's Custom Queries Through the Generic Repository

- [ ] Using **only** the generic repository's four members, try to reimplement:
      - The genre filter + `take` clamp on the book list.
      - The "does this `AuthorId` exist" check from Path 03 M7.
      - The single-query authors-with-book-count projection from Path 03 M9.
- [ ] At least one of these will force an uncomfortable choice. Notice which, and why:

| Temptation | What it actually costs you |
|---|---|
| Leak `IQueryable<T>` out of the generic repository so callers can compose their own filters | The "repository" no longer hides EF Core at all — callers are writing LINQ-to-Entities directly, so what did the abstraction actually buy you? |
| Add a `bookCount` field by looping and querying per author outside the repository | You've just reintroduced the exact N+1 you fixed in Path 03, one layer further away from the query that causes it. |
| Duplicate the genre/`take` logic in the endpoint handler instead of the repository | Now the same rule lives in two places, and the repository stops being the source of truth for how books are queried. |

- [ ] Write down (a plain note is fine — you'll fold it into your M8 ADR) exactly which of these
      you hit and why the generic repository couldn't cleanly avoid it.

### M3 — Move to Specific Repositories

- [ ] Replace the generic repository with `IBookRepository` and `IAuthorRepository` matching the
      [specific repository contract](#specific-repositories-from-m3-onward), each exposing exactly
      the methods your endpoints actually need.
- [ ] Move the genre/`take` query, the `AuthorId` existence check, and the book-count projection
      into these repositories as first-class methods — not leaked `IQueryable`.
- [ ] Confirm, by re-reading your own code, that nothing outside the repository implementations
      references EF Core types directly anymore.

### M4 — Introduce the Unit of Work

- [ ] Add `IUnitOfWork` matching its [contract](#specific-repositories-from-m3-onward) above,
      exposing both repositories plus a single `SaveChangesAsync()`.
- [ ] Change every endpoint handler so that `AddAsync`/`Remove` calls only **stage** changes;
      saving happens exactly once, through the unit of work, at the end of the request.
- [ ] Re-run the full Path 01–03 regression suite to confirm nothing broke now that saving is
      decoupled from each individual repository call.

**Edge case to verify:** stage two different changes in the same request (e.g. update a book and
delete an author's stale draft, or any two unrelated staged changes you can construct) and confirm
a single `SaveChangesAsync()` call persists both.

### M5 — Prove the Unit of Work Actually Does Something

- [ ] Implement `POST /api/authors/with-first-book` per the
      [new endpoint contract](#new-endpoint-contract-post-apiauthorswith-first-book): create the
      author and their first book in one request, staged through both repositories, saved with one
      `SaveChangesAsync()` call.
- [ ] Deliberately send a request where the `book` portion fails validation (e.g. missing
      `title`), and confirm:
      - The request fails with `400`.
      - **No author was persisted either** — check `GET /api/authors` afterward and prove it.
- [ ] This is the first concrete, observable evidence in this path that the Unit of Work is doing
      something a bare `IBookRepository`/`IAuthorRepository` calling `SaveChanges` independently,
      per repository, could not have guaranteed.

### M6 — Testability Check

- [ ] Write one test for the M5 endpoint's atomicity behavior (success case and the
      forced-failure case) using a **hand-written fake** `IUnitOfWork` (and fake repositories)
      instead of a real database. (A full test-writing workflow is Path 11 — here, the test itself
      is just the instrument you're using to answer this path's real question.)
- [ ] Answer honestly, based on what you just did, not on what you've read elsewhere: was writing
      that fake meaningfully easier because of the repository/unit-of-work interfaces, or would
      swapping out `DbContext` directly (e.g. against a different EF Core provider for testing)
      have been just as easy? Keep this observation for M8.

### M7 — Regression + Performance Sanity Check

- [ ] Full regression pass across every Path 01–04 scenario.
- [ ] Re-run the Path 03 N+1 check (SQL logging) against the repository-mediated version of
      `GET /api/authors` and confirm the abstraction did **not** quietly reintroduce the N+1 you
      fixed there.

### M8 — Write Your Verdict

- [ ] Write a short Architecture Decision Record — a real markdown file in your project (e.g.
      `docs/adr/0001-repository-unit-of-work.md`), not an answer to a quiz question — covering:
      1. What problem this pattern is usually claimed to solve.
      2. What you actually observed while building it: the specific pain point(s) from M2, the
         atomicity proof from M5, and the testability finding from M6.
      3. Your decision: **keep** this layer as-is, **simplify** it (e.g. drop the generic
         interface but keep the Unit of Work, or vice versa), or **revert** to Path 03's
         direct-`DbContext`-backed stores.
      4. The concrete reasons for that decision, referencing what you actually saw in *this*
         project — not a general opinion about the pattern.
- [ ] There is no expected answer here. A well-reasoned "I'm reverting this" is exactly as valid a
      deliverable as "I'm keeping it" — what matters is that the decision is backed by what you
      observed, not by what a tutorial told you to do.

## Manual Test Script

Run in order, against your Path 03 database (existing books/authors are fine to keep).

```http
@baseUrl = https://localhost:5001

### 1. Baseline regression - list authors with book count (Path 03 behavior preserved)
GET {{baseUrl}}/api/authors

### 2. Baseline regression - list books with genre filter + take (Path 01/02 behavior preserved)
GET {{baseUrl}}/api/books?genre=fantasy&take=5

### 3. Create an author + first book atomically -> 201, Location on the new author
POST {{baseUrl}}/api/authors/with-first-book
Content-Type: application/json

{
  "author": { "name": "Ursula K. Le Guin" },
  "book": {
    "title": "A Wizard of Earthsea",
    "publishedYear": 1968,
    "genre": "Fantasy"
  }
}

### 4. Confirm both the author and the book actually exist now
GET {{baseUrl}}/api/authors
GET {{baseUrl}}/api/books?genre=fantasy

### 5. Attempt the same operation with an invalid book -> 400
POST {{baseUrl}}/api/authors/with-first-book
Content-Type: application/json

{
  "author": { "name": "Should Not Be Saved" },
  "book": {
    "title": "",
    "publishedYear": 1968,
    "genre": "Fantasy"
  }
}

### 6. Confirm the author from step 5 was NOT persisted -> not present in the list
GET {{baseUrl}}/api/authors

### 7. Attempt the same operation with an invalid author -> 400, no book created either
POST {{baseUrl}}/api/authors/with-first-book
Content-Type: application/json

{
  "author": { "name": "   " },
  "book": {
    "title": "Orphan Book",
    "publishedYear": 2000,
    "genre": "Mystery"
  }
}

### 8. Confirm no book titled "Orphan Book" exists
GET {{baseUrl}}/api/books
```

Manual (non-HTTP) step: with EF Core SQL logging still enabled from Path 03, repeat request #1 and
count the queries fired — it must still be a small constant number, not one-per-author.

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Add an explicit EF Core transaction (`BeginTransactionAsync`) inside the unit of work and
      explain, specifically, what it protects against that a single `SaveChangesAsync()` call
      alone does not (hint: think about multiple `SaveChangesAsync()` calls that need to succeed
      or fail together, not just multiple staged changes before one save).
- [ ] Add the `409 Conflict` author-delete behavior from Path 03's stretch goals, now backed by
      `IAuthorRepository.HasBooksAsync`.
- [ ] Try deliberately breaking atomicity — call `SaveChangesAsync()` once after staging the
      author, and again after staging the book, in the same request — and reproduce the orphaned
      Author bug the Unit of Work was supposed to prevent, to see the failure mode with your own
      eyes.
- [ ] Extend your M8 ADR with a follow-up entry after finishing Path 05, once DTOs exist —
      does introducing DTOs change your verdict about how much the repository layer helps?

## Definition of Done

- [ ] M1–M8 all checked off, in order, each with manual test evidence.
- [ ] The generic repository from M1 has been superseded by specific repositories, or you've
      explicitly documented in your ADR why you kept it.
- [ ] Every existing Path 01–03 endpoint still behaves identically (full regression pass).
- [ ] `POST /api/authors/with-first-book` is proven atomic: a forced mid-operation failure leaves
      **no** partial data behind, verified by an actual follow-up request, not assumed.
- [ ] At least one test exists exercising that atomicity using a hand-written fake, not a real
      database.
- [ ] The Path 03 N+1 fix is confirmed still in effect through the new repository layer.
- [ ] A real ADR file exists in the repo with an explicit, reasoned keep/simplify/revert decision.

## Self-Review Checklist

- [ ] You can point to the exact spot in M2 where the generic repository broke down, and explain
      why in terms of *this* codebase, not in the abstract.
- [ ] You can explain the difference between "staged" and "saved" in your own implementation, and
      point to the one line where saving actually happens.
- [ ] You did not leave any `IQueryable<T>` leaking out of a repository's public interface.
- [ ] Your ADR's decision is backed by something you directly observed in M2, M5, or M6 — not a
      general opinion copied from an article.
- [ ] Nothing about the wire-level contract of the pre-existing endpoints changed as a side effect
      of this refactor.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `feat(m4): introduce IUnitOfWork, decouple save from repository calls`.
- [ ] Keep the M1–M2 "generic repository" commits separate from the M3 "specific repositories"
      commits, so the history shows the actual evolution, not a single rewritten result.
- [ ] Commit the ADR from M8 as its own commit with a clear message, e.g.
      `docs(adr): decision on repository/unit-of-work pattern`.

## Reference Docs (use only when stuck)

Pattern background:
- [Repository pattern with EF Core (design guidance)](https://learn.microsoft.com/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/infrastructure-persistence-layer-design)
- [EF Core `DbContext` as a Unit of Work](https://learn.microsoft.com/ef/core/saving/transactions)

Transactions:
- [Transactions in EF Core](https://learn.microsoft.com/ef/core/saving/transactions)

Testing with a hand-written fake (Path 11 covers testing in depth — this is just enough to answer
M6):
- [Integration tests in ASP.NET Core](https://learn.microsoft.com/aspnet/core/test/integration-tests)

Writing the ADR:
- [Architecture Decision Records overview](https://learn.microsoft.com/azure/well-architected/architect-role/architecture-decision-record)
