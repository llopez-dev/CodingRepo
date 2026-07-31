# 10. API Security Hardening — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 09](09-authentication-authorization.md). Build-it project — contracts, test requests, and
> checklists only. No C# solution code.

## Project Brief

Authentication and authorization (Path 09) answer "who is this, and what are they allowed to do."
This path answers a different question: what happens when someone tries to abuse the API itself —
hammer the login endpoint, call it from a browser origin it was never meant to serve, send a
gigantic payload, or simply poke at whatever was accidentally left misconfigured. You'll add CORS,
rate limiting, HTTPS/HSTS enforcement, a body-size limit, do a full secrets audit across every
earlier path, and finish with a genuine, filled-in self-review against the OWASP API Security
Top 10 — including being honest about which items don't apply here, and why.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through
      [Path 09](09-authentication-authorization.md) are fully done — authentication, authorization,
      validation, and standardized error responses are all in place. This path hardens what already
      exists; it doesn't add new business features.

## Rules

- **No new business features.** Everything here is hardening or auditing what Paths 01–09 already
  built.
- Every finding from the [OWASP audit](#owasp-api-security-top-10-audit) that isn't genuinely
  "not applicable" must end this path as either **Audited-OK** (verified, not assumed) or
  **Fixed** — nothing left unresolved.
- Don't weaken anything from Path 08 (error detail hiding) or Path 09 (auth) while hardening
  something else — re-run their regressions, not just this path's new scenarios.
- CORS, rate limiting, and body-size limits apply identically across `v1` and `v2` — same
  discipline as every other cross-cutting concern so far.

## CORS Policy Contract

| Setting | Value |
|---|---|
| Allowed origin(s) | A specific, named origin representing a real future consumer (e.g. `https://localhost:3000` for a local frontend) — **not** a wildcard `*`. |
| Allowed methods | `GET`, `POST`, `PUT`, `DELETE` — whatever your API actually uses, nothing extra. |
| Allowed headers | Must include whatever header carries your JWT (`Authorization`), plus `Content-Type`. |
| Credentials | Decide deliberately whether the policy allows credentials, and understand that this can't be combined with a wildcard origin even if you wanted it to be. |

## Rate Limit Policy Contract

| Policy | Applies to | Limit (illustrative — pick concrete numbers and document them) | On exceeding |
|---|---|---|---|
| `auth` (strict) | `POST /api/auth/login`, `POST /api/auth/register` | e.g. 5 requests / minute per client | `429 Too Many Requests`, with a `Retry-After` header |
| `general` (relaxed) | Every other endpoint | e.g. 100 requests / minute per client | `429 Too Many Requests`, with a `Retry-After` header |

## Worked Example: Reading the Evidence

This path is mostly proven by response headers and status codes, not response bodies — get
comfortable reading raw HTTP responses, not just JSON payloads.

A successful CORS preflight response from an **allowed** origin:

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://localhost:3000
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
```

A request from a **disallowed** origin — notice the `Access-Control-Allow-Origin` header simply
isn't present, which is what causes the browser itself to block the response from reaching your
script:

```http
HTTP/1.1 200 OK
Content-Type: application/json

[ ... the body may still come back, but without CORS headers a browser will not expose it to the calling page ... ]
```

A rate-limited response after exceeding the strict `auth` policy:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 42
Content-Type: application/problem+json

{
  "type": "https://tools.ietf.org/html/rfc6585#section-4",
  "title": "Too many requests.",
  "status": 429,
  "traceId": "00-7c9e6679712c4396a9db0348dad9c1e5-01"
}
```

A response outside `Development`, showing HSTS present:

```http
HTTP/1.1 200 OK
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

The same endpoint in `Development` — no `Strict-Transport-Security` header at all:

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

## OWASP API Security Top 10 Audit

Fill this in honestly against **this project specifically** — "not applicable" is a legitimate,
valuable answer when you can justify it; it is not the same as skipping the row.

| # | Category | Check performed on the Book Catalog API | Result |
|---|---|---|---|
| API1 | Broken Object Level Authorization | Confirm no endpoint lets one user reach another user's private data via a guessable id | Books/authors are shared catalog data, not per-user owned — document why this mostly doesn't apply, then double-check the `trusted-contributor` grant endpoint (Path 09) really is `Admin`-only regardless of which `userId` is targeted |
| API2 | Broken Authentication | Re-verify Path 09's password policy, token expiry, and generic login failure message | Audited-OK, or fix what's missing |
| API3 | Broken Object Property Level Authorization | Confirm no DTO lets a client set `id`, a role, or a claim via mass assignment on create/update | Audited-OK, or fix what's missing |
| API4 | Unrestricted Resource Consumption | Rate limiting (M2), the Path 05 max-page-size clamp, and this path's request body size limit (M4) | Implemented |
| API5 | Broken Function Level Authorization | Re-audit Path 09's Authorization Matrix across **every** endpoint, in **both** versions — this is exactly the kind of check where "we did it for `v1`" quietly stops being true for `v2` | Audited-OK, or fix what's missing |
| API6 | Unrestricted Access to Sensitive Business Flows | Rate limit registration/login specifically (M2), since these are the flows most attractive to automate against | Implemented |
| API7 | Server-Side Request Forgery | Confirm the API never makes an outbound request built from a client-supplied URL | Not applicable today — document why, and note it would need revisiting if that ever changes |
| API8 | Security Misconfiguration | CORS (M1), HSTS (M3), dev exception pages gated to `Development` (Path 08), Swagger UI exposure decision (M7) | Implemented / Audited-OK |
| API9 | Improper Inventory Management | Cross-check the Path 06 OpenAPI spec genuinely lists every reachable endpoint in both versions, with nothing forgotten or undocumented | Audited-OK, or fix what's missing |
| API10 | Unsafe Consumption of APIs | Confirm the API doesn't consume any third-party API at all today | Not applicable today — document why |

## Suggested Project Structure

Additions on top of Paths 01–09:

- [ ] A named CORS policy registered and applied in your composition root.
- [ ] Rate limiter configuration with (at least) the two policies from the
      [Rate Limit Policy Contract](#rate-limit-policy-contract).
- [ ] HSTS configuration, gated to run outside `Development`.
- [ ] A request body size limit applied consistently (globally, or per-endpoint if your tooling
      makes that easier).
- [ ] The filled-in [OWASP audit table](#owasp-api-security-top-10-audit) saved somewhere in your
      project (e.g. `docs/security-review.md`), not just in your head.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — CORS: From Nothing to a Real Policy

- [ ] Audit your current CORS configuration — if you have none, or a permissive default, that's
      your starting point.
- [ ] Define and apply a real policy per the [CORS Policy Contract](#cors-policy-contract):
      specific origin(s), specific methods, specific headers (including whatever carries your JWT).
- [ ] Verify a request carrying an **allowed** origin succeeds, including the preflight
      (`OPTIONS`) request if you're testing from an actual browser-based tool.
- [ ] Verify a request from a **disallowed** origin is rejected by the policy.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Preflight (`OPTIONS`) request from the allowed origin, for a method/header you actually support | `204`, with matching `Access-Control-Allow-*` headers. |
| Preflight request asking for a method you don't support (e.g. `PATCH` if you never implemented it here) | The disallowed method is not reflected back in `Access-Control-Allow-Methods`. |
| Actual request (not preflight) from a disallowed origin | Server may still process it, but the response lacks CORS headers, so a browser blocks the calling script from reading it — confirm you understand this is enforced by the browser, not your server, for simple/actual requests. |

### M2 — Rate Limiting

- [ ] Add rate limiting middleware with the two policies from the
      [Rate Limit Policy Contract](#rate-limit-policy-contract): a strict one on
      `POST /api/auth/login` and `POST /api/auth/register`, and a more relaxed general one on
      everything else.
- [ ] Confirm exceeding the strict limit returns `429 Too Many Requests` with a `Retry-After`
      header, and that normal usage well under the limit is completely unaffected.
- [ ] Confirm the two policies are independent — hammering login shouldn't lock you out of reading
      the book catalog.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Exactly at the configured limit | Still succeeds. |
| One request past the limit | `429`, with `Retry-After`. |
| Waiting out the `Retry-After` window, then retrying | Succeeds again. |
| Heavy traffic against `general` endpoints while `auth` is also being hammered | Each policy's counter is independent of the other. |

### M3 — HTTPS and HSTS

- [ ] Confirm HTTPS redirection (present since the Path 01 template default) is actually doing
      something — send a plain `http://` request and confirm it's redirected or rejected, not
      silently served.
- [ ] Add HSTS, and confirm — deliberately, by checking response headers yourself — that it's
      present outside `Development` and **absent** in `Development` (HSTS on `localhost` during
      local development is actively unhelpful, which is why ASP.NET Core's own defaults exclude
      it there).

### M4 — Request Body Size Limits

- [ ] Add a reasonable maximum request body size limit, applied consistently across the API.
- [ ] Send a request whose body deliberately exceeds the limit and confirm it's rejected cleanly
      (a `413 Payload Too Large`-family response) rather than hanging, crashing, or consuming
      unbounded memory while it's read.

### M5 — Secrets Audit

- [ ] Search your **entire** project history and current files for anything that shouldn't be
      there: hardcoded connection strings, the JWT signing key, the seeded `Admin` password
      (Path 09), any API key. Paths 02, 03, and 09 are the most likely places something slipped
      through.
- [ ] Move anything you find into User Secrets (locally) or environment variables/a secret
      store (anywhere else), and confirm `.gitignore` actually excludes the files that hold them
      locally.
- [ ] Confirm this by trying to find the secret again purely from what's tracked by git — you
      should come up empty.

### M6 — OWASP API Security Top 10 Self-Review

- [ ] Work through every row of the [OWASP audit table](#owasp-api-security-top-10-audit) against
      your actual project, filling in a real result for each — not a copy of the illustrative
      wording above.
- [ ] For anything you mark "not applicable," write one sentence justifying why, specific to this
      project — "not applicable" without a reason is indistinguishable from having skipped it.
- [ ] For anything you find that's actually broken, fix it before moving on, and note what you
      fixed.

### M7 — Misconfiguration Sweep (a closer look at API8)

- [ ] Confirm detailed/developer exception pages are impossible to trigger outside `Development`
      (re-verify Path 08's exception-handling milestone still holds).
- [ ] Decide, deliberately, what happens to the Swagger/OpenAPI UI (Path 06) outside
      `Development` — publicly reachable, gated behind authentication, or disabled entirely — and
      implement whichever you choose. "It just happens to still be open" is not a decision.
- [ ] Confirm the seeded `Admin` account (Path 09) is unambiguously gated to `Development` and
      could not exist in any other environment as currently configured.

### M8 — Full Regression + Security Regression Matrix

- [ ] Full functional regression across every Path 01–09 scenario.
- [ ] New security-specific regression pass:

| Check | Expected result |
|---|---|
| Request from an allowed CORS origin | Succeeds. |
| Request from a disallowed CORS origin | Rejected by the CORS policy. |
| Repeated rapid login attempts | `429` once the limit is exceeded, with `Retry-After`. |
| Plain `http://` request | Redirected to HTTPS or rejected, never silently served. |
| Response headers outside `Development` | HSTS present. |
| Response headers in `Development` | HSTS absent. |
| Oversized request body | `413`-family response, not a hang or crash. |
| [OWASP audit table](#owasp-api-security-top-10-audit) | Every row is `Implemented`, `Audited-OK`, or a justified `Not applicable` — zero unresolved findings. |

## Common Misconfigurations to Actively Look For

Things worth checking on purpose, not just waiting to stumble onto:

- A CORS policy that technically "works" only because it still allows every origin (`*`) — that's
  not a policy, that's the absence of one.
- Rate limiting configured but never actually wired into the request pipeline (a policy that
  exists in code but isn't applied to any endpoint protects nothing).
- HSTS enabled everywhere, including `Development` — harmless most of the time, but a sign the
  environment-gating pattern from Path 02 wasn't actually applied here.
- A body-size limit set so high it might as well not exist, or so low it rejects legitimate
  requests (e.g. a book with a long but reasonable title) — verify against your actual Path 01
  field-length rules, not a guess.
- Any secret that's only "not hardcoded" in the newest code, while an older commit from Path 02,
  03, or 09 still has it in history — a secrets audit that only checks the current working tree
  misses this.
- A Swagger/OpenAPI UI (Path 06) left reachable outside `Development` by default, simply because
  nobody revisited it after Path 06 shipped.

## Manual Test Script

Some of this (CORS preflight, HSTS headers) is easiest to verify with your HTTP client showing raw
response headers, or a browser's network tab, rather than a plain `.http` file — call those out
explicitly.

```http
@baseUrl = https://localhost:5001

### 1. Plain HTTP request -> should redirect/reject (try the http:// scheme, not https://)
GET http://localhost:5000/api/v1/books

### 2. Repeated login attempts to trigger the strict rate limit (run this request rapidly, several times in a row)
POST {{baseUrl}}/api/auth/login
Content-Type: application/json

{ "email": "nobody@example.com", "password": "WrongPassword!" }

### 3. Oversized body -> 413 (pad "title" with a very large string, well beyond your configured limit)
POST {{baseUrl}}/api/v1/books
Authorization: Bearer {{contributorToken}}
Content-Type: application/json

{ "title": "REPLACE_WITH_A_STRING_LARGER_THAN_YOUR_CONFIGURED_LIMIT", "publishedYear": 2000, "genre": "Mystery", "authorId": 1 }

### 4. Regression - anonymous read still works
GET {{baseUrl}}/api/v1/books

### 5. Regression - authenticated write still works under the relaxed rate limit
POST {{baseUrl}}/api/v1/books
Authorization: Bearer {{contributorToken}}
Content-Type: application/json

{ "title": "Still Works", "publishedYear": 2010, "genre": "Mystery", "authorId": 1 }
```

Manual (non-HTTP / browser-based) steps:

1. From a browser console (or a tool that lets you set an `Origin` header) on your **allowed**
   origin, call an endpoint and confirm it succeeds, including the preflight `OPTIONS` request.
2. Repeat from a **disallowed** origin and confirm the browser blocks the response (or the server
   rejects the preflight, depending on how you test it).
3. Inspect response headers outside `Development` and confirm HSTS is present; inspect them in
   `Development` and confirm it's absent.
4. Open the Swagger/OpenAPI UI outside `Development` and confirm it behaves exactly the way you
   decided in M7 (reachable-with-auth, or fully disabled).

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Make your rate limit keyed per authenticated user where possible, falling back to per-IP for
      anonymous requests, instead of a single global counter.
- [ ] Add security response headers beyond HSTS (e.g. `X-Content-Type-Options`,
      `Referrer-Policy`) and justify each one you add in terms of what it actually protects
      against for an API (some browser-security headers matter far more for HTML-serving apps than
      JSON APIs — don't add one you can't explain).
- [ ] Run a dependency vulnerability scan against your project's NuGet packages and resolve
      anything it flags.
- [ ] Write your `docs/security-review.md` audit as something you'd actually be comfortable
      handing to a real reviewer, and ask yourself whether every "Audited-OK" row would survive
      someone else trying to poke a hole in it.
- [ ] Search your full git history (not just current files) for the string patterns a secret
      would match (a connection string, a key-looking Base64 blob) to double-check M5 didn't miss
      anything committed and later "fixed" without ever being removed from history.
- [ ] Add a lightweight security-focused automated check (a preview of Path 11) that fails your
      build if a wildcard CORS origin or an obviously hardcoded secret pattern ever reappears.

## Definition of Done

- [ ] M1–M8 all checked off, in order, each with manual test evidence.
- [ ] A real, named-origin CORS policy is enforced — no wildcard origin anywhere.
- [ ] Rate limiting is active on both the strict `auth` policy and the general policy, verified
      with real `429` responses.
- [ ] HSTS is present outside `Development` and absent inside it, verified by actually reading
      response headers.
- [ ] An oversized request body is rejected cleanly, not left to hang or crash the process.
- [ ] A full secrets audit found and fixed everything, verified by searching what's actually
      tracked by git — not by memory.
- [ ] The [OWASP audit table](#owasp-api-security-top-10-audit) is completely filled in, with
      zero unresolved findings, saved in the project.
- [ ] The full Path 01–09 regression pass still succeeds.

## Self-Review Checklist

- [ ] You tested the CORS policy from both an allowed and a disallowed origin yourself — not just
      read the configuration and assumed it was right.
- [ ] You actually triggered the rate limit and saw a real `429`, rather than trusting the
      configuration values alone.
- [ ] You can name, specifically, what you found (if anything) during the M5 secrets audit — not
      just "everything was already fine."
- [ ] Every "Not applicable" row in your OWASP table has a one-sentence justification you'd stand
      behind if someone challenged it.
- [ ] You made a deliberate, stated decision about Swagger UI exposure outside `Development` —
      not an accidental default.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `feat(m2): rate limiting on auth endpoints and general API traffic`.
- [ ] Commit the M5 secrets-audit fixes separately, with a message describing what was moved
      (not what the secret's value was).
- [ ] Commit the filled-in OWASP audit document as its own commit, e.g.
      `docs(m6): OWASP API Security Top 10 self-review`.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 11 starts writing
      automated tests against an API that's now both correct and hardened.

## Reference Docs (use only when stuck)

CORS:
- [Enable CORS in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/cors)

Rate limiting:
- [Rate limiting middleware in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/rate-limit)

HTTPS/HSTS:
- [Enforce HTTPS in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/enforcing-ssl)

Secrets management (from Path 09, revisited here):
- [Safe storage of app secrets in development](https://learn.microsoft.com/aspnet/core/security/app-secrets)

OWASP:
- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
