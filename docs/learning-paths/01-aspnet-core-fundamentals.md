# 01. ASP.NET Core Web API Basics — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). This is a build-it project,
> not a reading assignment. Code first; only open the reference docs when you're actually stuck on
> a milestone. Nothing in this document is C# solution code — contracts, test requests, and
> checklists only. You design and write the implementation yourself.

## Project Brief

Build a small **in-memory** CRUD API that manages a catalog of books. No database (that's Path 03),
no validation library, no auth — just routing, model binding, HTTP verbs, and status codes, done
right.

The domain (books) is deliberately boring. You already understand what a book is, so none of your
effort goes into figuring out the domain — all of it goes into the HTTP mechanics: how a request
becomes a C# object, how your return value becomes a response, and why the status code you pick
matters. This exact project (the "Book Catalog API") is also the **running project** for the rest
of the roadmap: Path 02 adds configuration to it, Path 03 gives it a real database, and so on. Get
the shape right now and every later path gets easier.

## Environment Setup (M0)

Do this once, before M1:

- [ ] Confirm the .NET SDK is installed: run `dotnet --version` in a terminal.
- [ ] Create a working folder for the project (e.g. `book-catalog-api/`).
- [ ] Make sure the folder is tracked by git (either as part of this repo or its own), with a
      standard .NET `.gitignore` (`bin/`, `obj/`) so build output never gets committed.
- [ ] Pick your HTTP testing tool now: a `.http` file in VS Code (REST Client extension), curl, or
      Postman/Insomnia. You'll use it constantly starting at M1 — don't wait until the end to test.
- [ ] Create an empty test file (e.g. `requests.http`) next to the project so you can add requests
      to it as you complete each milestone.

## Rules

- No EF Core / database — storage is in-memory for this project. That's Path 03.
- No validation library (FluentValidation, etc.) — do required-field and range checks by hand.
  That rigor is the point of this path; libraries that hide it come later (Path 08).
- No authentication/authorization — every endpoint is open for now. That's Path 09.
- No DTOs/mapping libraries — returning the `Book` type directly is an accepted shortcut here.
  You'll notice why that's a shortcut in Path 05.
- No copying a tutorial's CRUD API — look up individual pieces (a route constraint, a status code)
  as needed, but design and wire up the endpoints yourself.
- Every milestone must be manually tested with real HTTP requests before you check it off. Don't
  eyeball the code and assume it works — send the request and read the actual response.

## Resource Contract

Whatever you build must match this shape exactly — it's the contract every later path assumes.

| Field | Type | Required | Rule |
|---|---|---|---|
| `id` | integer | server-assigned | Never accepted from the client; you generate it on create. |
| `title` | string | yes | Non-empty after trimming; reasonable max length (e.g. 200 chars). |
| `author` | string | yes | Non-empty after trimming; reasonable max length (e.g. 150 chars). |
| `publishedYear` | integer | yes | Must be between `1450` and the current year — no future books, and nothing pre-dating the printing press. |
| `genre` | string | yes | Non-empty after trimming; reasonable max length (e.g. 50 chars). |

Example representation of a single book, as returned by the API:

```json
{
  "id": 1,
  "title": "Clean Code",
  "author": "Robert C. Martin",
  "publishedYear": 2008,
  "genre": "Software Engineering"
}
```

Example payload for creating/replacing a book (no `id` — the client never sends one):

```json
{
  "title": "Clean Code",
  "author": "Robert C. Martin",
  "publishedYear": 2008,
  "genre": "Software Engineering"
}
```

## Suggested Project Structure

You don't have to match this exactly, but you should end up with a similar separation of concerns
rather than everything crammed into one file:

- [ ] A file for the `Book` type itself.
- [ ] A file/interface for the store abstraction (e.g. something like `IBookStore`) plus its
      in-memory implementation — this makes M2's thread-safety requirement easier to isolate and
      test, and sets you up well for Path 02/03 when the implementation behind the interface
      changes.
- [ ] `Program.cs` wiring: DI registration, middleware, endpoint mapping.
- [ ] `requests.http` (or your tool of choice) holding every request you used to test.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Scaffold

- [ ] Create a new ASP.NET Core Web API project (Minimal API template — look up the exact CLI
      command yourself).
- [ ] Run it and confirm it responds to whatever sample endpoint the template ships with.
- [ ] Strip out anything from the template you don't understand yet, so your `Program.cs` only
      contains code you can explain.
- [ ] Commit.

### M2 — Model + Store

- [ ] Define the `Book` type matching the [Resource Contract](#resource-contract) above.
- [ ] Implement an in-memory store behind an interface (see
      [Suggested Project Structure](#suggested-project-structure)).
- [ ] Register the store in DI as a **singleton** — data must survive across requests.
- [ ] Make the store safe under **concurrent** access. Kestrel can process multiple requests at
      the same time; a plain unprotected `List<Book>` mutated from two requests at once is a real
      bug, not a theoretical one. Research a concurrency-safe approach.
- [ ] Have the store generate the next `id` itself (e.g. an incrementing counter) — the API must
      never trust a client-supplied `id` on create.

**Edge cases to think about now, even though there's no endpoint yet:**

| Scenario | What should happen |
|---|---|
| Two "create" calls arrive at almost the same instant | Both must succeed with two different, correct ids — no lost update, no duplicate id. |
| Store starts empty | Reading from it returns an empty collection, not `null` or an exception. |

### M3 — List Books

- [ ] Implement `GET /api/books` → `200 OK` with a JSON array of all books (an empty array, not an
      error, if there are none yet).
- [ ] Support an optional `?genre=` query string parameter. When present, return only books whose
      `genre` matches **case-insensitively**; when absent, return everything.

Example:

```http
GET /api/books?genre=fantasy HTTP/1.1
```

Expected response:

```json
[
  {
    "id": 3,
    "title": "The Hobbit",
    "author": "J.R.R. Tolkien",
    "publishedYear": 1937,
    "genre": "Fantasy"
  }
]
```

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| No query string at all | `200` + every book. |
| `?genre=` with a value that matches nothing | `200` + `[]` (empty array) — **not** `404`. |
| `?genre=` with different casing than stored (e.g. `FANTASY` vs `Fantasy`) | Still matches. |
| `?genre=` present but empty (`?genre=`) | Decide on and document a consistent behavior (e.g. treat as "no filter"), don't leave it undefined. |

### M4 — Get One Book

- [ ] Implement `GET /api/books/{id}` → `200 OK` + the book, or `404 Not Found` if no book with
      that id exists.
- [ ] Constrain `{id}` to its real type directly in the route template (don't just parse a string
      and check manually — use the routing feature built for this).

**Edge cases to verify:**

| Request | Expected result |
|---|---|
| `GET /api/books/1` where `1` exists | `200` + that book's exact JSON shape. |
| `GET /api/books/9999` where it doesn't exist | `404`, with no body content that leaks internals (no stack trace). |
| `GET /api/books/abc` (non-numeric id) | Confirm what actually happens with your route constraint in place, and make sure it's a sensible client error rather than a `500`. |
| `GET /api/books/-1` or `/0` | Decide whether these are valid ids in your scheme and be consistent — don't let them accidentally 500. |

### M5 — Create a Book

- [ ] Implement `POST /api/books` reading the request body as JSON.
- [ ] Validate every required field and rule from the [Resource Contract](#resource-contract) by
      hand. On failure, return `400 Bad Request` with a body that actually names the problem field
      (don't just return an empty 400).
- [ ] On success: generate the id, store the book, and return `201 Created` with a `Location`
      header pointing at `GET /api/books/{id}` for the new resource, and the created book (with
      its id) as the response body.

Example request:

```http
POST /api/books HTTP/1.1
Content-Type: application/json

{
  "title": "Dune",
  "author": "Frank Herbert",
  "publishedYear": 1965,
  "genre": "Science Fiction"
}
```

Expected response:

```http
HTTP/1.1 201 Created
Location: /api/books/4
Content-Type: application/json

{
  "id": 4,
  "title": "Dune",
  "author": "Frank Herbert",
  "publishedYear": 1965,
  "genre": "Science Fiction"
}
```

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Missing `title` | `400`, message identifies `title`. |
| Missing `author` | `400`, message identifies `author`. |
| Missing `publishedYear` | `400`. |
| `publishedYear` in the future (e.g. `3000`) | `400`. |
| `publishedYear` before `1450` | `400`. |
| `genre` is an empty string | `400` (empty after trimming still counts as missing). |
| Client sends an `id` field in the body | Ignored — the server-generated id always wins. |
| Client sends an unrecognized extra field | Decide and document the behavior (commonly: ignored, not an error). |
| Body is not valid JSON at all | Notice what happens *before* your own handler code runs. |
| `Content-Type` header missing or wrong (e.g. `text/plain`) | Test it and note the actual behavior — don't assume. |

### M6 — Replace a Book

- [ ] Implement `PUT /api/books/{id}` as a **full replace**: the request body must satisfy every
      rule in the [Resource Contract](#resource-contract), exactly like create.
- [ ] `404 Not Found` if no book with that id exists (this project does **not** upsert on `PUT` —
      a `PUT` to a nonexistent id is a client error, not a create).
- [ ] `400 Bad Request` on the same validation failures as `POST`.
- [ ] On success, return `200 OK` (with the updated book) or `204 No Content` — pick one and be
      consistent across every success case in this project.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Valid replace of an existing book | Success code of your choice, and a subsequent `GET` shows the new values. |
| `PUT` to an id that doesn't exist | `404`, and no book is created as a side effect. |
| `PUT` with the same invalid-field cases as create | `404`. |
| Calling the exact same valid `PUT` twice in a row | Both calls succeed and the end state is identical — this is what "idempotent" means in practice; verify it, don't just assume it. |

### M7 — Delete a Book

- [ ] Implement `DELETE /api/books/{id}` → `204 No Content` on success, `404 Not Found` if no book
      with that id exists.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Delete an existing book | `204`, and a following `GET` for the same id now returns `404`. |
| Delete the same id a second time | `404` — note that the *server state* is idempotent (still gone), even though the *response* differs between the first and second call. |
| Delete an id that never existed | `404`. |

### M8 — Full Manual Test Pass

- [ ] Start the API fresh (restart the process so the in-memory store is empty).
- [ ] Run through every scenario in the [Manual Test Script](#manual-test-script) below, in order,
      confirming the actual status code and body match what's expected at each step.
- [ ] Fix anything that doesn't match before checking this milestone off.
- [ ] Save the final version of your test requests in the repo alongside the project.

## Manual Test Script

A suggested order of requests to exercise the whole API end-to-end, written in `.http`-file style
(adapt to curl/Postman if you prefer). Run them **in order** against a freshly started API — later
requests depend on the state created by earlier ones.

```http
@baseUrl = https://localhost:5001

### 1. List books on an empty store -> 200 + []
GET {{baseUrl}}/api/books

### 2. Create a valid book -> 201 + Location header + body with generated id
POST {{baseUrl}}/api/books
Content-Type: application/json

{
  "title": "Dune",
  "author": "Frank Herbert",
  "publishedYear": 1965,
  "genre": "Science Fiction"
}

### 3. Create a second valid book, different genre -> 201
POST {{baseUrl}}/api/books
Content-Type: application/json

{
  "title": "The Hobbit",
  "author": "J.R.R. Tolkien",
  "publishedYear": 1937,
  "genre": "Fantasy"
}

### 4. Create with a missing field -> 400
POST {{baseUrl}}/api/books
Content-Type: application/json

{
  "author": "Unknown",
  "publishedYear": 2000,
  "genre": "Mystery"
}

### 5. Create with an out-of-range year -> 400
POST {{baseUrl}}/api/books
Content-Type: application/json

{
  "title": "Book From The Future",
  "author": "Nobody",
  "publishedYear": 3000,
  "genre": "Sci-Fi"
}

### 6. List all books -> 200 + the 2 successfully created books
GET {{baseUrl}}/api/books

### 7. List filtered by genre -> 200 + only the matching book
GET {{baseUrl}}/api/books?genre=fantasy

### 8. List filtered by a genre nothing matches -> 200 + []
GET {{baseUrl}}/api/books?genre=horror

### 9. Get an existing book by id -> 200 + exact body
GET {{baseUrl}}/api/books/1

### 10. Get a nonexistent book -> 404
GET {{baseUrl}}/api/books/9999

### 11. Replace an existing book with valid data -> 200 or 204
PUT {{baseUrl}}/api/books/1
Content-Type: application/json

{
  "title": "Dune (Deluxe Edition)",
  "author": "Frank Herbert",
  "publishedYear": 1965,
  "genre": "Science Fiction"
}

### 12. Confirm the replace took effect -> 200 + updated fields
GET {{baseUrl}}/api/books/1

### 13. Replace a nonexistent book -> 404, no book created
PUT {{baseUrl}}/api/books/9999
Content-Type: application/json

{
  "title": "Ghost",
  "author": "Nobody",
  "publishedYear": 2000,
  "genre": "Mystery"
}

### 14. Replace with invalid data -> 400
PUT {{baseUrl}}/api/books/1
Content-Type: application/json

{
  "title": "",
  "author": "Frank Herbert",
  "publishedYear": 1965,
  "genre": "Science Fiction"
}

### 15. Delete an existing book -> 204
DELETE {{baseUrl}}/api/books/2

### 16. Confirm deletion -> 404
GET {{baseUrl}}/api/books/2

### 17. Delete the same book again -> 404
DELETE {{baseUrl}}/api/books/2

### 18. Delete a book that never existed -> 404
DELETE {{baseUrl}}/api/books/12345
```

## Stretch Goals

Only after every milestone above is checked off and the full test pass is clean:

- [ ] Add `PATCH /api/books/{id}` for a partial update (e.g. changing only `genre`). Decide what
      happens if the body includes fields that don't need changing, and what happens if it
      includes an invalid value for a field it does touch.
- [ ] Add custom middleware (not an endpoint) that logs method, path, resulting status code, and
      duration for every request. This is your first real touch of `HttpContext` and the
      middleware pipeline outside of what the template gave you.
- [ ] Add a route constraint edge-case test: confirm what happens when a request uses a verb your
      route doesn't support at all (e.g. `PATCH` before you've built it, or `POST` to
      `/api/books/{id}`).
- [ ] Rebuild the same API using Controllers instead of Minimal APIs (or vice versa if you started
      with Controllers), reusing the same `Book` type and store, so you feel the difference in
      how endpoints are declared instead of just reading about it.
- [ ] Add an OPTIONS/HEAD exploration: send a manual `HEAD /api/books/1` and `OPTIONS /api/books`
      and observe what ASP.NET Core does automatically without you writing anything for either.

## Definition of Done

- [ ] M1–M8 all checked off, in order, each with manual test evidence.
- [ ] Every scenario in the [Manual Test Script](#manual-test-script) produces the documented
      status code and body shape, run against a freshly started instance.
- [ ] `Location` header present and correct on every successful `POST` response.
- [ ] `id` is always server-generated; a client-supplied `id` in a `POST` body is never used.
- [ ] The in-memory store is verified safe under concurrent access, not just assumed to be.
- [ ] No endpoint ever returns a raw exception/stack trace to the caller.
- [ ] `PUT` and `DELETE` are verified idempotent (repeating the same call doesn't change the
      outcome), even though a repeated `DELETE` correctly returns `404` the second time.

## Self-Review Checklist

Go through your own code against this list before calling the project done — this is what a senior
reviewer would actually look for in a pull request at this stage:

- [ ] Every endpoint returns exactly the status codes specified in its milestone — no "close
      enough" substitutions (e.g. `200` where `204` was specified).
- [ ] Validation logic lives in one place per rule, not copy-pasted between `POST` and `PUT`.
- [ ] The store's public surface (its interface) doesn't leak implementation details (e.g. no
      `List<Book>` exposed directly for external mutation).
- [ ] Nothing in `Program.cs` is there because a template put it there and you never asked why.
- [ ] You can point at the exact line where the id is generated and explain why it can't be
      supplied by the client.
- [ ] You can point at the exact mechanism that makes the store safe under concurrent access and
      explain, specifically, what would break without it.

## Suggested Git Workflow

- [ ] Commit after each milestone (M1 through M8), not just once at the end — small, working
      commits make it much easier to see (and revert) exactly what each step added.
- [ ] Use a consistent commit message convention, e.g. `feat(m3): list books with genre filter`,
      so the history itself documents your progress through the milestones.
- [ ] Tag or note the commit where the [Definition of Done](#definition-of-done) is fully
      satisfied — you'll want to diff against it once Path 02 starts changing this same project.

## Reference Docs (use only when stuck)

General:
- [ASP.NET Core web API overview](https://learn.microsoft.com/aspnet/core/web-api/)
- [Minimal APIs overview](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis)

For M3/M4 (routing & querying):
- [Routing in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/routing)
- [Route constraint reference](https://learn.microsoft.com/aspnet/core/fundamentals/routing#route-constraint-reference)

For M5/M6 (binding & validation):
- [Model binding in ASP.NET Core](https://learn.microsoft.com/aspnet/core/mvc/models/model-binding)
- [Parameter binding in Minimal APIs](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/parameter-binding)

For every milestone (responses & status codes):
- [Controller action return types](https://learn.microsoft.com/aspnet/core/web-api/action-return-types)
- [Minimal API responses (`TypedResults`)](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/responses)

For the stretch-goal middleware:
- [ASP.NET Core Middleware](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/)
