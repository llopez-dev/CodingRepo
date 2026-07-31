# 08. Validation & Error Handling — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 07](07-api-versioning.md). Build-it project — contracts, test requests, and checklists
> only. No C# solution code.

## Project Brief

Since Path 01 you've been validating input with hand-rolled `if` checks, scattered across every
create/replace endpoint — and after Path 07, duplicated across **two API versions**. This path
replaces all of it with a real validation approach, and makes every error response — validation
failures, not-found errors, and genuinely unexpected exceptions alike — come back in one
consistent, predictable shape. You'll try the built-in approach first, feel exactly where it
breaks down, move to a proper validation library, and finally make sure an actual unhandled bug
can never leak a stack trace to a client again.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 07](07-api-versioning.md) are fully
      done — both `v1` and `v2` of the Book Catalog API work, each with their own scattered manual
      validation checks that this path will replace.

## Rules

- **No new endpoints and no new business rules.** Every validation rule you enforce today (title
  required, `publishedYear` range, `authorId` must exist, `v2`'s `genres` must be non-empty, etc.)
  keeps the exact same meaning — only **how** it's enforced and **how** the resulting error looks
  changes.
- By the end of this path, there should be **zero** hand-rolled `if`-based validation checks left
  anywhere in the project. If you find one, it's not done yet.
- Full exception details (stack traces, exception messages) may be visible **only** in
  `Development` — never in anything else — same environment-gating discipline as Path 02.
- Apply everything in this path to **both** `v1` and `v2` consistently; if the two versions end up
  with different validation/error behavior for the same underlying rule, that's a regression.

## Error Response Contract

Every error response, of every kind, follows [RFC 7807](https://www.rfc-editor.org/rfc/rfc7807)
`ProblemDetails`, with these fields:

| Field | Always present? | Purpose |
|---|---|---|
| `type` | Yes | A URI identifying the error category (your framework's default type URIs are fine — consistency matters more than matching any particular reference exactly). |
| `title` | Yes | A short, human-readable summary. |
| `status` | Yes | The HTTP status code, repeated in the body. |
| `detail` | For non-validation errors | A specific, human-readable explanation of *this* occurrence. |
| `errors` | Validation errors only | A dictionary of field name → list of messages. |
| `traceId` | Yes | A correlation id tying this response back to your server-side logs (see M7). |

Example validation error (`400`):

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-01",
  "errors": {
    "Title": ["Title is required."],
    "PublishedYear": ["PublishedYear must be between 1450 and 2026."]
  }
}
```

Example not-found error (`404`):

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.5.4",
  "title": "Book not found.",
  "status": 404,
  "detail": "No book exists with id 9999.",
  "traceId": "00-9e1c3a9e2f3b4a2d8a6e1c9f0a2b3c4d-01"
}
```

Example unhandled-exception error (`500`) — note there is deliberately **no** exception message or
stack trace here, even though one exists server-side:

```json
{
  "type": "https://tools.ietf.org/html/rfc7231#section-6.6.1",
  "title": "An unexpected error occurred.",
  "status": 500,
  "traceId": "00-1a2b3c4d5e6f70819243546576879a0b-01"
}
```

## Suggested Project Structure

Additions on top of Paths 01–07:

- [ ] A short audit note/table (M1) capturing every validation rule and error shape that exists
      today, before you change anything.
- [ ] DataAnnotations attributes added directly to request DTOs (M2), later removed for the rules
      that don't fit (M3).
- [ ] A FluentValidation validator class per request DTO (M4), replacing the manual checks
      entirely.
- [ ] Global exception-handling wired into the composition root (M6), plus `ProblemDetails`
      configuration applied project-wide.
- [ ] Nothing version-specific here — this infrastructure sits underneath both `v1` and `v2`.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Audit Current Validation and Error Responses

- [ ] Before changing anything, write down every validation rule currently enforced, where it
      lives in your code today, and what the resulting error response actually looks like right
      now. A table like this is enough:

| Rule | Where it's currently checked | Current error shape |
|---|---|---|
| `Title` required | Manual `if` in `v1`/`v2` create+replace | (whatever your code returns today) |
| `PublishedYear` range | Manual `if`, duplicated across `v1`/`v2` | |
| `Genre`/`Genres` required/non-empty | Manual `if`, differs between `v1` and `v2` | |
| `AuthorId` must reference an existing author | Manual DB lookup + `if` (Path 03 M7) | |
| `Author.Name` required | Manual `if` (Path 03 M6) | |
| `sortBy`/`sortDir` allowed values | Manual `if` (Path 05 M6) | |

- [ ] Keep this table around — you'll use it in M4 to confirm nothing was lost in the rewrite.

### M2 — Try DataAnnotations First

- [ ] Add DataAnnotations attributes (`[Required]`, `[StringLength]`, `[Range]`, etc.) to every
      request DTO, covering as many rules from your M1 audit as a plain attribute can express.
- [ ] Research and confirm, for your specific SDK version, whether Minimal API endpoints validate
      DataAnnotations automatically, or whether you need to trigger validation yourself (this has
      changed across .NET versions — don't assume based on something you read that may be
      outdated; verify it against your own running project).
- [ ] Get at least one endpoint genuinely rejecting invalid input through DataAnnotations alone,
      with your old manual `if` check for that same field removed.

### M3 — Feel DataAnnotations' Limits

- [ ] Try to express these three rules using only DataAnnotations attributes, and note exactly
      where each one breaks down:

| Rule | Why a plain attribute struggles |
|---|---|
| `PublishedYear` must be between `1450` and **the current year** | The upper bound is dynamic (today's date), not a fixed constant a simple attribute can hold. |
| `AuthorId` must reference an **existing** author | This needs a database lookup — attributes don't get dependency-injected access to your repositories. |
| `v2`'s `Genres` must have at least one non-empty element | Validating the *contents* of a collection, not just its presence, pushes past what the built-in attributes cover cleanly. |

- [ ] Write a short note (a paragraph is enough) stating exactly which rules you couldn't cleanly
      express this way — you'll need this list to confirm nothing is missing once you move to
      FluentValidation in M4.

### M4 — Introduce FluentValidation

- [ ] Add FluentValidation and write one validator class per request DTO, reimplementing **every**
      rule from your M1 audit table — including the three that broke down in M3. Look up how
      FluentValidation validators can take constructor-injected dependencies (needed for the
      `AuthorId`-exists check).
- [ ] Wire validation into the request pipeline so it actually runs before your handler code for
      Minimal APIs (this needs explicit wiring — research the current recommended approach for
      your SDK, e.g. an endpoint filter).
- [ ] Delete every remaining hand-rolled `if`-based validation check. Cross-check against your M1
      table: every rule listed there must now be enforced by a validator, not by leftover manual
      code.

### M5 — Standardize Error Responses with `ProblemDetails`

- [ ] Add ASP.NET Core's built-in `ProblemDetails` support project-wide.
- [ ] Confirm every validation failure now returns the exact shape from the
      [Error Response Contract](#error-response-contract), with one entry per invalid field.
- [ ] Go back through every non-validation error status your API can return (`404`s for missing
      books/authors, `400`s for invalid `sortBy`/`sortDir`, `400`s for a nonexistent `AuthorId`
      reference, `400` for an empty `v2` `genres` array) and confirm **all of them** now use the
      same consistent `ProblemDetails` shape too — not just the ones FluentValidation produces
      directly.

### M6 — Global Exception-Handling Middleware

- [ ] Add a global exception handler (research the current recommended approach for your SDK —
      there's a dedicated interface for this in modern ASP.NET Core, as well as an older
      middleware-based way of doing the same thing) that catches any unhandled exception and turns
      it into the `500` shape from the [Error Response Contract](#error-response-contract) — no
      stack trace, no raw exception message, ever, in the response body.
- [ ] Deliberately trigger a genuinely unhandled exception on purpose (a temporary, obviously fake
      bug is fine) and confirm:
      - The client receives the safe, well-formed `500` response.
      - The **real** exception detail (message, stack trace) is still visible to you as the
        developer, through logging — you haven't lost the ability to actually debug the problem,
        you've just stopped handing it to the client.
- [ ] Confirm full exception detail is only ever exposed in the response body itself when running
      in `Development`, and never otherwise — remove your temporary fake bug once this is proven.

### M7 — Correlate Errors for Debugging

- [ ] Confirm every error response includes a `traceId` (or wire one up if it doesn't already)
      that also appears in your server-side logs for that same request.
- [ ] Prove you can take a `traceId` from an error response and find the corresponding log entry
      for that exact request — this is the entire point of including it.

### M8 — Apply Consistently Across `v1` and `v2`

- [ ] Confirm both API versions from Path 07 use the exact same validation and error-handling
      infrastructure — this should mostly fall out naturally from Path 07 M5's shared-core rule,
      but verify it explicitly rather than assuming it.
- [ ] Trigger the same underlying rule violation (e.g. a missing `Title`) through both `v1` and
      `v2` and confirm the `ProblemDetails` shape is identical between them, aside from whatever is
      genuinely version-specific (e.g. the `Genres` vs. `Genre` field name in the `errors`
      dictionary).

### M9 — Full Regression + Error Contract Verification

- [ ] Full regression pass across every Path 01–07 scenario.
- [ ] Specifically re-verify every error case documented in every earlier path's Definition of
      Done — each one must now return the same consistent `ProblemDetails` shape, with the same
      underlying rule still enforced.

## Manual Test Script

Run in order against both `v1` and `v2` where applicable.

```http
@baseUrl = https://localhost:5001

### 1. Missing title -> 400 ProblemDetails with errors.Title
POST {{baseUrl}}/api/v1/books
Content-Type: application/json

{
  "publishedYear": 1965,
  "genre": "Science Fiction",
  "authorId": 2
}

### 2. PublishedYear out of range -> 400 with errors.PublishedYear
POST {{baseUrl}}/api/v1/books
Content-Type: application/json

{
  "title": "Future Book",
  "publishedYear": 3000,
  "genre": "Science Fiction",
  "authorId": 2
}

### 3. Nonexistent AuthorId -> 400 with errors.AuthorId (now via FluentValidation's DB-backed rule)
POST {{baseUrl}}/api/v1/books
Content-Type: application/json

{
  "title": "Orphan Book",
  "publishedYear": 2000,
  "genre": "Mystery",
  "authorId": 9999
}

### 4. v2 empty genres array -> 400 with errors.Genres
POST {{baseUrl}}/api/v2/books
Content-Type: application/json

{
  "title": "No Genre Book",
  "publishedYear": 2000,
  "genres": [],
  "authorId": 2
}

### 5. Get a nonexistent book -> 404 ProblemDetails shape
GET {{baseUrl}}/api/v1/books/9999

### 6. Invalid sortBy -> 400 ProblemDetails, detail names the allowed values
GET {{baseUrl}}/api/v1/books?sortBy=nonsense

### 7. Same missing-title rule through v2 -> identical ProblemDetails shape (aside from field name)
POST {{baseUrl}}/api/v2/books
Content-Type: application/json

{
  "publishedYear": 1965,
  "genres": ["Science Fiction"],
  "authorId": 2
}
```

Manual (non-HTTP) steps:

1. Temporarily introduce a deliberate unhandled exception somewhere reachable by a request, call
   it, and confirm you get the safe `500` shape from the
   [Error Response Contract](#error-response-contract) — then remove the deliberate bug.
2. Check your application logs for that same request and confirm the real exception details are
   there, and that the `traceId` from the response matches the log entry.
3. Repeat the log check for a couple of the earlier requests (#1–#6) and confirm each `traceId` is
   traceable the same way.

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Add a custom, machine-readable `errorCode` extension field to your `ProblemDetails`
      responses (beyond the RFC 7807 standard fields) so a client could handle specific error
      cases programmatically instead of string-matching on `title`/`detail`.
- [ ] Try `IValidatableObject` directly on one DTO for a cross-field rule, purely to compare it
      against the equivalent FluentValidation rule you already wrote.
- [ ] Write one or two automated tests (a light preview of Path 11) asserting the exact
      `ProblemDetails` shape for a validation error and a not-found error.
- [ ] Look into localizing validation messages, and decide whether it's worth doing for this
      project at its current size.

## Definition of Done

- [ ] M1–M9 all checked off, in order, each with manual test evidence.
- [ ] Zero hand-rolled `if`-based validation checks remain anywhere in the project.
- [ ] Every rule from the M1 audit table is still enforced, now through FluentValidation.
- [ ] Every error response — validation, not-found, or unexpected exception — matches the
      [Error Response Contract](#error-response-contract) exactly, including a `traceId`.
- [ ] An unhandled exception can be triggered and produces a safe `500` with no leaked internals,
      while still being fully visible in your own logs.
- [ ] `v1` and `v2` produce identical `ProblemDetails` shapes for the same underlying rule
      violations.
- [ ] The full Path 01–07 regression pass still succeeds, with every documented error case now
      returning the standardized shape.

## Self-Review Checklist

- [ ] You can point to the exact three rules from M3 that DataAnnotations couldn't cleanly
      express, and explain why in terms of *this* codebase.
- [ ] You searched your entire project for leftover manual `if` validation and found none — not
      just the ones you remembered to check.
- [ ] You personally triggered a real unhandled exception and confirmed, with your own eyes, both
      the safe client response and the detailed server-side log entry.
- [ ] Nothing about *what* is validated changed during this refactor — only *how* it's validated
      and *how* the resulting error looks.
- [ ] `v1` and `v2` were both checked, not just whichever one you happened to test first.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `feat(m4): replace manual validation with FluentValidation`.
- [ ] Keep the M2 (DataAnnotations) and M4 (FluentValidation) commits separate, even though M2's
      code mostly gets replaced — the history should show you actually tried the simpler option
      first, not jumped straight to the more powerful one.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 09 adds
      authentication on top of an API that now fails predictably.

## Reference Docs (use only when stuck)

Validation:
- [Model validation in ASP.NET Core](https://learn.microsoft.com/aspnet/core/mvc/models/validation)
- [FluentValidation documentation](https://docs.fluentvalidation.net/)

Error handling and `ProblemDetails`:
- [Handle errors in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/error-handling)
- [RFC 7807: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807)

Logging and correlation:
- [Logging in .NET](https://learn.microsoft.com/dotnet/core/extensions/logging)
- [Distributed tracing and the `TraceIdentifier`](https://learn.microsoft.com/aspnet/core/fundamentals/logging/#logging-scopes)
