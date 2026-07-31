# 06. API Documentation with OpenAPI/Swagger — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 05](05-restful-api-design.md). Build-it project — contracts, test requests, and checklists
> only. No C# solution code.

## Project Brief

This path is purely additive: you're not changing what the API does, you're making sure the
**generated OpenAPI spec accurately and completely describes** what it already does. A generated
spec that's missing status codes, has no examples, or quietly documents a field as required when
your code treats it as optional, is worse than no spec at all — consumers will trust it, and get
burned. Most of this path is an audit: find every place the spec and the real behavior disagree,
and fix it.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 05](05-restful-api-design.md) are
      fully done — DTOs are in place everywhere, pagination/filtering/sorting work, and you have
      each path's Definition of Done listing exactly which status codes every endpoint can return.
      You'll use those as the source of truth in this path, not invent new ones.

## Rules

- **No behavior changes.** If the audit in M6 turns up an actual bug (not just a documentation
  gap), note it separately — fixing real behavior is incidental to this path, not its point.
- Every status code you document must come from your own Definition-of-Done tables in Paths
  01–05. Don't invent new ones here, and don't leave any of the real ones out.
- If you kept the Path 02 diagnostics endpoint around (gated to `Development`), it must be excluded
  from the generated spec entirely, in every environment. If you already removed it back in
  Path 02, there's nothing to do here — skip that part.
- Don't commit the generated spec file itself if your tooling writes one to disk — it's build
  output, the same as `bin`/`obj`.

## Documentation Coverage Contract

Every endpoint in the final spec must satisfy all five of these:

| # | Requirement | Where it comes from |
|---|---|---|
| 1 | A one-line summary of what the endpoint does | XML `<summary>` on the handler/action |
| 2 | A longer description for any non-obvious behavior (e.g. "returns 200 even for an empty page") | XML `<remarks>` or equivalent |
| 3 | Every status code the endpoint can **actually** return, each with an accurate response type | Your own Definition-of-Done tables from Paths 01–05 |
| 4 | At least one realistic example for the request body (if any) and the primary success response | Your own seeded data (e.g. "Dune" / "Frank Herbert"), not placeholder text |
| 5 | Not exposed unless it's meant to be public | No internal/diagnostic endpoints, no leftover EF entity fields |

## Worked Example

Two fully worked examples to mirror when you write your own — this is the level of detail every
endpoint's documentation should reach.

`POST /api/books` — example request body:

```json
{
  "title": "Dune",
  "publishedYear": 1965,
  "genre": "Science Fiction",
  "authorId": 2
}
```

`POST /api/books` — example `201` response, including the header that goes with it:

```http
HTTP/1.1 201 Created
Location: /api/books/4
Content-Type: application/json

{
  "id": 4,
  "title": "Dune",
  "publishedYear": 1965,
  "genre": "Science Fiction",
  "authorId": 2,
  "author": { "id": 2, "name": "Frank Herbert" }
}
```

`POST /api/books` — example `400` response, documented as its own distinct example instead of
being skipped just because it's an error case:

```json
{
  "title": "",
  "publishedYear": 1965,
  "genre": "Science Fiction",
  "authorId": 2
}
```

`GET /api/books` — example paginated `200` response, reusing the **same** seeded book and author
so a reader can tell the examples describe one consistent story, not a different random book every
time:

```json
{
  "items": [
    {
      "id": 4,
      "title": "Dune",
      "publishedYear": 1965,
      "genre": "Science Fiction",
      "authorId": 2,
      "author": { "id": 2, "name": "Frank Herbert" }
    }
  ],
  "page": 1,
  "pageSize": 10,
  "totalCount": 1,
  "totalPages": 1
}
```

## Suggested Project Structure

Additions on top of Paths 01–05:

- [ ] XML documentation file generation turned on for the project (a compiler/project setting).
- [ ] `///` doc comments on every endpoint and every DTO property from the
      [DTO Contract in Path 05](05-restful-api-design.md#dto-contract).
- [ ] Your chosen OpenAPI tooling's configuration in your composition root (`Program.cs` or
      equivalent) — look up what's current and recommended for your installed SDK version; recent
      .NET versions include built-in OpenAPI document generation, and Swashbuckle/NSwag remain
      common choices for a richer UI and more configuration options on top of that.
- [ ] Wherever your tooling wants response-type/example metadata attached (attributes, fluent
      configuration, or XML tags depend on your choice) — pick one consistent approach and use it
      everywhere.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Add OpenAPI Generation to the Project

- [ ] Add OpenAPI document generation to the project (built-in tooling, and/or Swashbuckle/NSwag —
      look up what's current for your SDK).
- [ ] Confirm you can fetch the raw generated document (JSON or YAML) at whatever endpoint your
      tooling exposes it at, and that it already lists every endpoint you've built so far — even
      before you've written a single doc comment.
- [ ] Add an interactive UI (Swagger UI or equivalent) and confirm you can browse it in a
      browser.

### M2 — Enable and Wire Up XML Documentation Comments

- [ ] Turn on XML documentation file generation for the project.
- [ ] Write a `///` summary comment for every endpoint, and for every DTO and DTO property from
      the Path 05 DTO Contract.
- [ ] Confirm those comments now show up in the generated spec/UI — not just in your editor's
      IntelliSense.

**Edge case to decide:** enabling this typically surfaces a compiler warning for any public member
missing an XML comment. Decide, deliberately, whether to fix every instance or suppress the
warning with a documented reason — don't just silence it project-wide without thinking about it.

### M3 — Annotate Every Possible Response, Not Just the Happy Path

- [ ] For every endpoint, explicitly declare **every** status code it can actually return and the
      response type/shape for each — not just whatever single "happy path" type your framework
      infers automatically by default.
- [ ] Build yourself audit tables like these and fill them in honestly, endpoint by endpoint —
      don't skip a row just because you're fairly sure it's fine:

**Books:**

| Endpoint | Status codes per your Path 01–05 Definitions of Done | Currently in the generated spec | Every code has a response type? | Mismatch? |
|---|---|---|---|---|
| `GET /api/books` | 200 | | | |
| `GET /api/books/{id}` | 200, 404 | | | |
| `POST /api/books` | 201, 400 | | | |
| `PUT /api/books/{id}` | 200/204, 400, 404 | | | |
| `DELETE /api/books/{id}` | 204, 404 | | | |

**Authors:**

| Endpoint | Status codes per your Path 01–05 Definitions of Done | Currently in the generated spec | Every code has a response type? | Mismatch? |
|---|---|---|---|---|
| `GET /api/authors` | 200 | | | |
| `GET /api/authors/{id}` | 200, 404 | | | |
| `POST /api/authors` | 201, 400 | | | |
| `POST /api/authors/with-first-book` | 201, 400 | | | |

**Internal-only (must NOT appear in the spec at all):**

| Endpoint | Expected in the spec? | Actually absent? |
|---|---|---|
| Path 02 diagnostics endpoint (if kept) | No — not even in `Development` | |

- [ ] Fix every row where the columns don't match, including making sure the internal-only table
      really does come up empty when you search the generated document.

### M4 — Add Realistic Examples

- [ ] Add at least one realistic example request and response body for every endpoint that
      accepts or returns one, using your own attached technique (XML `<example>` tags, attributes,
      or filters — whichever your tooling supports).
- [ ] Use your own actual seeded data in every example ("Dune" / "Frank Herbert" / `1965`) instead
      of placeholder values like `"string"` or `0` — an example that looks like real data is far
      more useful to a consumer than one that technically type-checks.

### M5 — Hide What Shouldn't Be Public

- [ ] Confirm the Path 02 diagnostics endpoint (if you kept it) is excluded from the generated
      spec entirely, in every environment — not merely gated behind `Development` for the endpoint
      itself.
- [ ] Re-check every DTO from the Path 05 contract for any field that shouldn't be exposed. This
      doubles as a second, spec-driven pass over Path 05's "no EF entity crosses the wire" rule —
      this time verified by reading the generated document itself, not just your own source code.

### M6 — Accuracy Audit: Make the Spec Lie-Proof

- [ ] Deliberately go looking for at least one place where the generated spec currently disagrees
      with the API's real behavior. Common places to check:

| Where to look | What a mismatch looks like |
|---|---|
| Required vs. optional fields | Spec marks a field required that your code actually treats as optional, or vice versa. |
| Nullable fields | Spec doesn't mark something nullable that your code can genuinely return as `null`. |
| Enums / allowed values | `sortBy`/`sortDir`'s allowed values (Path 05) aren't reflected anywhere a consumer would see them before getting a `400`. |
| Pagination metadata | `PagedResult<T>`'s fields aren't fully documented on every endpoint that returns it. |

- [ ] Fix every mismatch you find. This milestone is the actual point of the whole path.

### M7 — Consume Your Own Spec

- [ ] Import the generated spec into a tool that can turn it into a generated HTTP client or a
      Postman/Insomnia collection (look up how), and use that generated client/collection to
      exercise a handful of endpoints instead of your own hand-written `.http` requests.
- [ ] Specifically try: creating a book, listing books with a filter/sort/page combination, and
      triggering at least one `400` and one `404` through the generated client.
- [ ] Note anywhere the generated client is awkward, wrong, or missing something entirely — that's
      a direct signal of a spec problem, seen from a consumer's perspective instead of your own.
- [ ] If the generated client got something noticeably wrong (a bad type, a missing field, an
      endpoint it couldn't call at all), trace it back to the specific documentation gap that
      caused it and fix that gap, not just the symptom.

## Common Documentation Smells

Things to actively look for, not just wait to notice by accident:

- A `200` documented as the only possible response on an endpoint that your own Path 01–05
  Definition of Done says can also return `404` or `400`.
- An example request body that wouldn't actually pass your own validation rules if you sent it for
  real (e.g. an example `publishedYear` outside the Path 01 range).
- A DTO property with a generic placeholder name/value (`"string"`, `"example"`, `0`) instead of
  something that looks like real catalog data.
- A required field in the spec that your code actually defaults when missing, or vice versa.
- A response shape documented once and then silently drifting out of sync after a later change in
  this same path (re-check M3's audit tables after finishing M4–M6, not just once at the start).
- Anything from the Path 02 diagnostics endpoint or any other non-public route showing up in the
  document at all, in any environment.

### M8 — Full Regression + Documentation Coverage Check

- [ ] Walk every endpoint against the [Documentation Coverage Contract](#documentation-coverage-contract)
      and confirm all five requirements are met for each one.
- [ ] Full functional regression pass across Paths 01–05 — nothing about actual behavior should
      have changed; this path only added descriptions on top of it.

## Manual Test Script

This path is about the generated documentation, not new endpoints, so "testing" here means
fetching and reviewing the spec itself alongside your existing regression requests.

```http
@baseUrl = https://localhost:5001

### 1. Fetch the raw generated OpenAPI document (path depends on your tooling - note your actual one)
GET {{baseUrl}}/openapi/v1.json

### 2. Confirm the interactive UI loads in a browser (not scriptable - open it manually)
# GET {{baseUrl}}/swagger  (or whatever path your tooling uses)

### 3. Regression - core book/author scenarios from Paths 01-05 still behave identically
GET {{baseUrl}}/api/books?page=1&pageSize=5
GET {{baseUrl}}/api/authors
```

Manual (non-HTTP) review steps:

1. Open the fetched document (or the interactive UI) and, endpoint by endpoint, check it against
   the [Documentation Coverage Contract](#documentation-coverage-contract).
2. Search the document's raw text for the name of your Path 02 diagnostics endpoint (if you kept
   it) — it must not appear anywhere, in any environment.
3. Search the document for any DTO field name that shouldn't be public and confirm none exist.

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Try the OpenAPI tool you *didn't* pick in M1 (Swashbuckle vs. NSwag, or either against the
      built-in generator) purely to compare the developer experience — same evaluative approach as
      Path 05's mapping library comparison.
- [ ] Group endpoints into logical tags (e.g. "Books", "Authors") in the generated UI, and decide
      whether that's genuinely clearer for a consumer than a flat list.
- [ ] Add a placeholder security scheme to the spec describing the JWT bearer auth you'll actually
      implement in Path 09 — just the documentation shape for now, no real authentication yet.
- [ ] Generate a typed HTTP client from your spec and call it from a tiny throwaway script, instead
      of only inspecting the generated code.
- [ ] Add a short "API Documentation" pointer to the main [README.md](../../README.md) linking to
      where the interactive UI runs locally, so future-you (or anyone else) doesn't have to
      rediscover the route by reading `Program.cs`.
- [ ] Diff the generated spec before and after a small deliberate change (e.g. renaming a DTO
      field) to see exactly how sensitive the generated document is to changes you make elsewhere
      in the codebase.

## Definition of Done

- [ ] M1–M8 all checked off, in order, each with manual test evidence.
- [ ] Every endpoint has a summary, and a description for anything non-obvious.
- [ ] Every endpoint's documented status codes match the [audit table](#m3--annotate-every-possible-response-not-just-the-happy-path)
      exactly — no mismatches left.
- [ ] Every endpoint accepting or returning a body has at least one realistic example.
- [ ] No internal/diagnostic endpoint and no unintended field appears anywhere in the generated
      spec.
- [ ] At least one real, previously-existing mismatch between the spec and actual behavior was
      found and fixed in M6 — if you genuinely found zero, double-check you looked hard enough
      before checking this off.
- [ ] The full Path 01–05 regression pass still succeeds, unchanged.

## Self-Review Checklist

- [ ] You can name, specifically, the mismatch you found and fixed in M6 — not just "everything
      matched," which is rarely true the first time through.
- [ ] You didn't just copy a placeholder example from documentation somewhere — every example
      reflects your own actual seeded data.
- [ ] You personally opened the generated UI and clicked through it, rather than only trusting
      that the raw JSON "looks complete."
- [ ] The diagnostics endpoint decision from Path 02/05 is still being honored here — you checked,
      not assumed.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `docs(m3): document all real status codes for book endpoints`.
- [ ] Keep the M6 accuracy-audit fixes as their own commit(s) with messages that name the actual
      mismatch fixed, so the history itself is a record of what was wrong.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 07 builds a new API
      version directly on top of this documented contract.

## Reference Docs (use only when stuck)

OpenAPI generation:
- [Generate OpenAPI documents in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/overview)
- [Swashbuckle (Swagger) for ASP.NET Core](https://learn.microsoft.com/aspnet/core/tutorials/getting-started-with-swashbuckle)
- [NSwag documentation](https://github.com/RicoSuter/NSwag/wiki)

XML documentation comments:
- [XML documentation comments (C#)](https://learn.microsoft.com/dotnet/csharp/language-reference/xmldoc/)

Examples and response types:
- [Customize and add examples to your OpenAPI document](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/customize-openapi)
- [Controller action return types (response type annotations)](https://learn.microsoft.com/aspnet/core/web-api/action-return-types)
