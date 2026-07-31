# 09. Authentication & Authorization — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 08](08-validation-error-handling.md). Build-it project — contracts, test requests, and
> checklists only. No C# solution code.

## Project Brief

Every endpoint has been open to anyone this whole time. This path locks it down: reading the
catalog stays anonymous, but creating, updating, and deleting now require a real logged-in user —
and not every logged-in user gets to do everything. You'll add registration and login backed by
ASP.NET Core Identity, issue and validate JWT bearer tokens, enforce role-based rules
(`Contributor` vs. `Admin`), and build one genuinely custom policy that combines a role **and** a
claim — because not every real-world authorization rule fits neatly into "has this role or not."

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 08](08-validation-error-handling.md)
      are fully done — in particular, validation and error responses are standardized (Path 08),
      since every new failure mode this path introduces (unauthenticated, unauthorized) needs to
      fit the same `ProblemDetails` contract, not a new one-off shape.

## Rules

- **Read endpoints stay anonymous.** `GET` on books and authors, in both API versions, must not
  require authentication. If you find yourself locking down a `GET`, stop — that's not this path's
  job.
- No secrets (signing keys, connection strings) hardcoded or committed to source control — read
  them from configuration (User Secrets locally), the same discipline as Path 02.
- Reuse Path 08's validation and `ProblemDetails` infrastructure for every new failure mode here —
  don't invent a separate error shape just for login/register.
- Apply every rule in this path identically to **both** `v1` and `v2` — authorization is not a
  wire-shape concern, it sits underneath both versions the same way Path 04's repositories do.
- `/api/auth/...` endpoints themselves are **not versioned** (no `/api/v1/auth/...`) — a deliberate
  decision: authentication is infrastructure shared by the whole API, not a resource that evolves
  the same way `Book` does.

## Identity & Token Contract

### `POST /api/auth/register`

Request:

```json
{ "email": "new.contributor@example.com", "password": "SomeStr0ngP@ss!" }
```

Response `201 Created` (no password, no token yet — registering and logging in are separate
steps):

```json
{ "id": "3f2504e0-4f89-11d3-9a0c-0305e82c3301", "email": "new.contributor@example.com" }
```

Every new user is registered with the `Contributor` role by default. Nobody can self-register as
`Admin` — see M1 for how the first `Admin` account gets created.

### `POST /api/auth/login`

Request:

```json
{ "email": "new.contributor@example.com", "password": "SomeStr0ngP@ss!" }
```

Response `200 OK`:

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresAtUtc": "2026-07-31T18:00:00Z"
}
```

`401 Unauthorized` (via the Path 08 `ProblemDetails` shape) for **any** login failure — wrong
password or an email that doesn't exist at all get the **exact same** generic error. Letting a
client tell those two cases apart is a real vulnerability (it lets an attacker enumerate which
emails have accounts) — don't do it, even by accident through a slightly different message.

### Authorization Matrix

| Operation | Requirement |
|---|---|
| `GET` books / authors (either version) | Anonymous |
| `POST` / `PUT` books (either version) | Authenticated, role `Contributor` or `Admin` |
| `DELETE` books (either version) | Authenticated, role `Admin` |
| `POST` authors, `POST` authors/with-first-book | Authenticated, role `Contributor` or `Admin` |
| `DELETE` authors (Path 03 stretch goal) | Authenticated, **and** (role `Admin`) **or** (role `Contributor` **and** the custom `trusted` claim) |

### `POST /api/auth/users/{userId}/trusted-contributor`

Admin-only. Grants the custom `trusted` claim to the given user, enabling them to delete authors
without being a full `Admin`.

- `204 No Content` on success.
- `404 Not Found` if `userId` doesn't exist.
- `403 Forbidden` if the caller isn't an `Admin`.

## Suggested Project Structure

Additions on top of Paths 01–08:

- [ ] ASP.NET Core Identity wired to your existing EF Core setup (new migration for Identity's
      tables).
- [ ] `Admin` and `Contributor` roles, seeded at startup.
- [ ] A default seeded `Admin` account, gated to `Development` only (same environment-gating
      pattern as Path 02's diagnostics endpoint).
- [ ] JWT issuing/validation configuration, with the signing key read from configuration/User
      Secrets — never hardcoded.
- [ ] A custom authorization requirement + handler for the "`Admin` OR (`Contributor` +
      `trusted`)" author-deletion policy — this one doesn't fit a single `[Authorize(Roles=...)]`
      attribute cleanly.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Add ASP.NET Core Identity

- [ ] Wire up ASP.NET Core Identity against your existing EF Core setup, and apply the migration
      for its tables.
- [ ] Seed the `Admin` and `Contributor` roles at startup.
- [ ] Seed one default `Admin` user at startup, gated to `Development` only — you need at least
      one `Admin` account to bootstrap everything else in this path, and there's no self-service
      way to become one.

### M2 — Registration and Login Endpoints

- [ ] Implement `POST /api/auth/register` per the [contract](#post-apiauthregister): create the
      user via Identity, assign the `Contributor` role, validate the request with FluentValidation
      (Path 08) — reasonable password rules (length, complexity) and a valid email format.
- [ ] Implement `POST /api/auth/login` per the [contract](#post-apiauthlogin): verify credentials
      through Identity, and prepare (but don't necessarily issue yet, that's M3) whatever you need
      to build a token on success.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Register with a valid, unused email/password | `201`. |
| Register with an email that's already registered | `400`, `ProblemDetails` shape from Path 08. |
| Register with a weak password | `400`, naming the specific rule violated. |
| Login with a valid email, wrong password | `401`, generic message. |
| Login with an email that was never registered | `401`, the **same** generic message as above. |

### M3 — Issue and Validate JWTs

- [ ] Configure JWT generation on successful login (issuer, audience, expiry, and a signing key
      read from configuration) — finish wiring `POST /api/auth/login` to actually return a token.
- [ ] Configure JWT Bearer authentication so incoming requests with a valid token are recognized.
- [ ] Add one simple protected diagnostic endpoint first — e.g. `GET /api/auth/me` returning the
      caller's claims — and prove the whole pipeline (register → login → call protected endpoint
      with the token) works **before** touching any real book/author endpoint.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Call `GET /api/auth/me` with a valid token | `200`, claims reflect the logged-in user. |
| Call it with no token at all | `401`. |
| Call it with an expired or tampered token | `401`. |

### M4 — Lock Down Write Endpoints with Roles

- [ ] Require authentication plus `Contributor` **or** `Admin` on every book/author create and
      update endpoint, in **both** `v1` and `v2`.
- [ ] Require authentication plus `Admin` specifically on book deletion, in both versions.
- [ ] Confirm every `GET` endpoint remains fully anonymous — re-run a few from your Path 01–08
      regression scripts with no token at all and confirm they still succeed.

### M5 — Custom Claim + Policy for Author Deletion

- [ ] Implement `POST /api/auth/users/{userId}/trusted-contributor` per the
      [contract](#post-apiauthusersuseridtrusted-contributor), `Admin`-only, granting the custom
      `trusted` claim.
- [ ] Implement the author-deletion policy from the
      [Authorization Matrix](#authorization-matrix): `Admin`, **or** `Contributor` with the
      `trusted` claim. Since this is an OR across two different kinds of checks (a role, and a
      claim), it doesn't fit a single built-in `[Authorize(Roles=...)]` attribute — implement it as
      a genuine custom authorization requirement and handler.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| `Admin` deletes an author | Succeeds. |
| Plain `Contributor` (no `trusted` claim) tries to delete an author | `403`. |
| `Contributor` granted `trusted` deletes an author | Succeeds. |
| Non-`Admin` calls the grant-trusted endpoint | `403`. |

### M6 — Apply Consistently Across `v1` and `v2`

- [ ] Confirm every rule in the [Authorization Matrix](#authorization-matrix) behaves identically
      across `v1` and `v2` — trigger the same operation (e.g. a `Contributor` creating a book)
      through both versions and confirm both succeed the same way, and the same for a forbidden
      case.

### M7 — Get 401 vs. 403 Right, Everywhere

- [ ] Audit every protected endpoint: a request with **no token at all** (or an invalid/expired
      one) must return `401`; a request with a **valid token but insufficient role/claim** must
      return `403`. These mean genuinely different things to a client, and mixing them up anywhere
      is a real regression, not a nitpick.
- [ ] Confirm both status codes still come back in the Path 08 `ProblemDetails` shape.

### M8 — Don't Leak Secrets or Credentials

- [ ] Confirm your JWT signing key lives in configuration (User Secrets locally; an environment
      variable or secret store anywhere else), never hardcoded or committed.
- [ ] Confirm no API response, anywhere, ever includes a password or password hash — check this
      explicitly rather than assuming your DTOs (Path 05) already prevent it.

### M9 — Full Regression + New Auth Scenario Matrix

- [ ] Full regression across every Path 01–08 scenario for anonymous reads — completely
      unaffected by this path.
- [ ] Full pass across the new scenario matrix: anonymous write attempt (`401`), `Contributor`
      create (success), `Contributor` delete book (`403`), `Admin` delete book (success),
      untrusted `Contributor` delete author (`403`), trusted `Contributor` delete author
      (success), `Admin` delete author (success).

### M10 — Document the Security Scheme

- [ ] Add the JWT bearer security scheme to your OpenAPI spec for real (Path 06 left this as a
      stretch-goal placeholder) so the generated UI lets you authenticate and try protected
      endpoints interactively.
- [ ] Make sure each protected endpoint's documentation (Path 06) mentions which role(s) it
      requires.

## Manual Test Script

Run in order against a freshly seeded database (one default `Admin` account from M1).

```http
@baseUrl = https://localhost:5001

### 1. Register a new Contributor -> 201
POST {{baseUrl}}/api/auth/register
Content-Type: application/json

{ "email": "contributor1@example.com", "password": "SomeStr0ngP@ss!" }

### 2. Register the same email again -> 400
POST {{baseUrl}}/api/auth/register
Content-Type: application/json

{ "email": "contributor1@example.com", "password": "AnotherStr0ngP@ss!" }

### 3. Login with wrong password -> 401, generic message
POST {{baseUrl}}/api/auth/login
Content-Type: application/json

{ "email": "contributor1@example.com", "password": "WrongPassword!" }

### 4. Login with an email that was never registered -> 401, SAME generic message as #3
POST {{baseUrl}}/api/auth/login
Content-Type: application/json

{ "email": "nobody@example.com", "password": "WhoKnows123!" }

### 5. Login with correct credentials -> 200 + accessToken
POST {{baseUrl}}/api/auth/login
Content-Type: application/json

{ "email": "contributor1@example.com", "password": "SomeStr0ngP@ss!" }

### 6. Call the protected "me" endpoint with the token from #5 -> 200, shows Contributor role
GET {{baseUrl}}/api/auth/me
Authorization: Bearer {{contributorToken}}

### 7. Anonymous read still works with no token at all
GET {{baseUrl}}/api/v1/books

### 8. Anonymous create attempt -> 401
POST {{baseUrl}}/api/v1/books
Content-Type: application/json

{ "title": "Should Fail", "publishedYear": 2000, "genre": "Mystery", "authorId": 1 }

### 9. Contributor create -> 201
POST {{baseUrl}}/api/v1/books
Authorization: Bearer {{contributorToken}}
Content-Type: application/json

{ "title": "Contributor Book", "publishedYear": 2010, "genre": "Mystery", "authorId": 1 }

### 10. Contributor attempts delete -> 403
DELETE {{baseUrl}}/api/v1/books/1
Authorization: Bearer {{contributorToken}}

### 11. Login as the seeded Admin -> 200 + accessToken
POST {{baseUrl}}/api/auth/login
Content-Type: application/json

{ "email": "admin@example.com", "password": "SeedAdminP@ss!" }

### 12. Admin delete succeeds
DELETE {{baseUrl}}/api/v1/books/1
Authorization: Bearer {{adminToken}}

### 13. Contributor (untrusted) attempts to delete an author -> 403
DELETE {{baseUrl}}/api/v1/authors/1
Authorization: Bearer {{contributorToken}}

### 14. Admin grants the trusted claim to the contributor -> 204
POST {{baseUrl}}/api/auth/users/{{contributorUserId}}/trusted-contributor
Authorization: Bearer {{adminToken}}

### 15. Contributor (now trusted) deletes an author -> succeeds
DELETE {{baseUrl}}/api/v1/authors/1
Authorization: Bearer {{contributorToken}}

### 16. Non-admin attempts to grant trusted status -> 403
POST {{baseUrl}}/api/auth/users/{{contributorUserId}}/trusted-contributor
Authorization: Bearer {{contributorToken}}

### 17. Same scenario matrix repeated against v2 to confirm identical behavior
POST {{baseUrl}}/api/v2/books
Authorization: Bearer {{contributorToken}}
Content-Type: application/json

{ "title": "V2 Contributor Book", "publishedYear": 2010, "genres": ["Mystery"], "authorId": 1 }
```

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Add refresh tokens so a client doesn't have to re-login every time the short-lived access
      token expires — and think through where refresh tokens need to be stored more carefully than
      access tokens.
- [ ] Add rate limiting specifically on `POST /api/auth/login` (a preview of Path 10) to slow down
      credential-guessing attempts.
- [ ] Add an `Admin`-only endpoint to list all users and their roles/claims, so you don't have to
      inspect the database directly to see the current state.
- [ ] Add a policy requiring **two** custom claims combined (not just one), to practice composing
      more than one condition in a single custom requirement.

## Definition of Done

- [ ] M1–M10 all checked off, in order, each with manual test evidence.
- [ ] Every `GET` endpoint, in both versions, remains fully anonymous.
- [ ] Registration, login, and the protected `/api/auth/me` endpoint all work end-to-end.
- [ ] The [Authorization Matrix](#authorization-matrix) is enforced exactly, in both versions.
- [ ] `401` and `403` are never confused anywhere in the project.
- [ ] Login never reveals whether a given email is registered.
- [ ] No secret or password/password hash ever appears in a committed file or an API response.
- [ ] The OpenAPI spec documents the security scheme and which roles each endpoint requires.
- [ ] The full Path 01–08 regression pass still succeeds for every anonymous read.

## Self-Review Checklist

- [ ] You can explain, specifically, why login returns the same error for "wrong password" and
      "no such user" — and you actually verified this yourself instead of assuming it.
- [ ] You can point to the custom authorization requirement/handler for author deletion and
      explain why a simple `[Authorize(Roles = "...")]` attribute couldn't express it.
- [ ] You checked, concretely, that no response body anywhere contains a password hash — not just
      assumed the DTOs from Path 05 already prevent it.
- [ ] Every `401` vs. `403` case in your manual test script actually returned the code you
      expected — not the other one.
- [ ] Your JWT signing key is not sitting in a file that's tracked by git.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `feat(m4): require Contributor/Admin roles on book write endpoints`.
- [ ] Never commit an actual signing key or seeded `Admin` password, even a "fake" one that looks
      real — use an obviously placeholder value in anything that gets committed, and keep the real
      one in User Secrets/your environment only.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 10 hardens this
      same authentication layer further (rate limiting, CORS, HTTPS enforcement).

## Reference Docs (use only when stuck)

Identity and authentication:
- [Introduction to Identity on ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authentication/identity)
- [JWT Bearer authentication in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authentication/jwt-authn)

Authorization:
- [Role-based authorization](https://learn.microsoft.com/aspnet/core/security/authorization/roles)
- [Claims-based authorization](https://learn.microsoft.com/aspnet/core/security/authorization/claims)
- [Custom authorization policy-based requirements](https://learn.microsoft.com/aspnet/core/security/authorization/policies)

Secrets management:
- [Safe storage of app secrets in development](https://learn.microsoft.com/aspnet/core/security/app-secrets)
