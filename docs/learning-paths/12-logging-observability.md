# 12. Logging & Observability — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 11](11-automated-testing.md). Build-it project — contracts, test requests, and checklists
> only. No C# solution code.

## Project Brief

Everything so far has been verified while you're sitting right there, watching the console. That
stops working the moment this API runs somewhere you can't just look at it directly. This path
adds structured logging for the events that actually matter, `/health` endpoints that answer two
genuinely different questions (is the process alive vs. can it actually serve traffic), and a
first real look at distributed tracing — and ties all three together around the **one** `traceId`
Path 08 already put in every error response, so a single identifier can lead you from "a client
saw this error" to "here are the exact logs and the exact trace for that request."

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 11](11-automated-testing.md) are
      fully done — in particular, Path 08's `traceId` on every error response and Path 11's
      automated test suite, both of which this path builds directly on top of.

## Rules

- **No new business features.** This path instruments what already exists; it doesn't change what
  the API does.
- **Never log a secret.** Passwords, attempted passwords, full JWTs, and connection strings must
  never appear in a log line, at any level, in any environment. This is exactly the kind of thing
  that's easy to do by accident (e.g. logging "login failed for {Email}/{Password}" during
  debugging and forgetting to remove it) — audit for it deliberately, don't just assume you didn't.
- Health check responses visible to an anonymous caller must not leak internal infrastructure
  details (hostnames, connection strings, internal exception messages) — this is the same
  Security Misconfiguration category (API8) you audited in Path 10, applied to a brand-new
  endpoint you're only now adding.
- Use the **same** `traceId` concept from Path 08 as your correlation backbone. Don't invent a
  second, unrelated identifier that means almost the same thing.

## Logging Contract

| Event | Level | Structured properties to include (illustrative) |
|---|---|---|
| Book created | Information | `BookId`, `AuthorId`, `UserId` |
| Book updated | Information | `BookId`, `UserId` |
| Book deleted | Information | `BookId`, `UserId` |
| Author created | Information | `AuthorId`, `UserId` |
| Login succeeded | Information | `UserId` — **never** the email or password |
| Login failed | Warning | Enough to investigate abuse (e.g. a hashed/partial identifier) — **never** the attempted password |
| Rate limit triggered (Path 10) | Warning | Policy name, a client identifier |
| Unhandled exception (Path 08) | Error | Exception type, the same `traceId` already in the client response |

These must be **structured** log entries (named properties you could later filter/query on), not
a single interpolated string that happens to contain the same information as unstructured text.

## Health Check Contract

| Endpoint | What it checks | Anonymous response |
|---|---|---|
| `/health/live` | The process is running — nothing else. No dependency checks at all. | `200` + a minimal body (e.g. just a status string). |
| `/health/ready` | Can the app actually serve traffic — at minimum, can it reach the database. | `200` if healthy, `503` if not; still just a minimal body — no connection strings, hostnames, or raw exception detail, even when something is failing. |

Liveness and readiness are genuinely different questions. A process can be alive (the `.NET`
process is running, `/health/live` says `200`) while not being ready (the database is temporarily
unreachable, `/health/ready` says `503`). Confusing the two is a real operational bug: a system
that restarts a process because it's not "ready" (when the process itself was actually fine) or,
worse, keeps routing traffic to a process that's alive but can't do anything useful.

## Trace and Log Correlation Contract

One `traceId`, three views of the exact same request:

1. **Client-visible**: the `traceId` field already present in every Path 08 `ProblemDetails`
   response.
2. **Logs**: every structured log line written while handling that request carries the same id as
   a property.
3. **Traces**: the OpenTelemetry trace/span tree for that request is discoverable using the same
   id.

If you can't go from a `traceId` a client reports to you, to both the matching log lines and the
matching trace, this path isn't done yet — regardless of how much logging or tracing code exists.

## Worked Example

### A structured log line — book created

```json
{
  "Timestamp": "2026-07-31T14:22:05.123Z",
  "Level": "Information",
  "MessageTemplate": "Book {BookId} created by user {UserId}",
  "Properties": {
    "BookId": 42,
    "AuthorId": 2,
    "UserId": "3f2504e0-4f89-11d3-9a0c-0305e82c3301",
    "TraceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-01"
  }
}
```

### A structured log line — login failed. Notice what's deliberately absent

```json
{
  "Timestamp": "2026-07-31T14:23:10.456Z",
  "Level": "Warning",
  "MessageTemplate": "Login failed for the request identified by {TraceId}",
  "Properties": {
    "TraceId": "00-9e1c3a9e2f3b4a2d8a6e1c9f0a2b3c4d-01"
  }
}
```

No email, no password, no attempted password — anywhere. If either shows up in your own version
of this log line, that's a bug to fix now, not a detail to clean up later.

### A trace, shown as a tree

```
Trace 00-4bf92f3577b34da6a3ce929d0e0e4736-01
└─ HTTP POST /api/v1/books                     [42ms]
   └─ EF Core: INSERT INTO Books ...           [8ms]
```

Even this simple two-level tree is the real shape of "a trace": one parent span for the whole
request, at least one child span for the work underneath it. A bigger system's trace could have
dozens of spans across several services — the shape doesn't change, only the depth.

### Minimal, safe health check responses

`GET /health/live`, always, regardless of anything else going on:

```json
{ "status": "Healthy" }
```

`GET /health/ready`, database reachable:

```json
{ "status": "Healthy" }
```

`GET /health/ready`, database unreachable — still minimal, no connection string, hostname, or
exception detail leaked to the caller:

```json
{ "status": "Unhealthy" }
```

## Suggested Project Structure

- [ ] Serilog configured in your composition root, replacing or wrapping the default logging
      provider, with at least a console sink and a rolling file sink.
- [ ] A small, explicit set of structured log calls at the points listed in the
      [Logging Contract](#logging-contract) — not a blanket "log everything" approach.
- [ ] Two health check endpoints matching the [Health Check Contract](#health-check-contract),
      including a custom check for database connectivity.
- [ ] OpenTelemetry configured for tracing (ASP.NET Core + EF Core instrumentation) exporting to a
      simple local exporter (e.g. console) for now — a full tracing backend is a stretch goal.
- [ ] One real counter metric, wired to an actual business event.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Add Serilog and Get a Baseline

- [ ] Replace the default logging provider with Serilog, console sink first.
- [ ] Log at least one event using **structured** properties (named values), and confirm your
      console output actually shows the individual properties, not just one flattened string that
      happens to contain the same words.

### M2 — Log Key Business Events, Not Just Framework Noise

- [ ] Add explicit, structured log entries for every row of the
      [Logging Contract](#logging-contract): book created/updated/deleted, author created, login
      succeeded/failed, rate limit triggered.
- [ ] Apply the log **level** specified for each row — don't log everything at `Information`
      just because it's the default you're used to seeing.
- [ ] Audit every new log call you just added for anything sensitive slipping in (a password, an
      attempted password, a full token) and fix any you find before moving on.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| A successful login | One `Information`-level structured log entry, `UserId` present, no credentials anywhere in it. |
| A failed login (wrong password) | One `Warning`-level entry, with **no** attempted password logged anywhere. |
| A rate limit trigger (Path 10) | One `Warning`-level entry identifying which policy was hit. |
| The database becomes briefly unreachable | `/health/ready`'s own internal check logs the failure at `Warning` or above — you don't have to go digging through an unrelated exception log to find out readiness flipped. |

### M3 — Add a File Sink and Environment-Appropriate Levels

- [ ] Add a rolling file sink alongside the console sink.
- [ ] Configure the minimum log level per environment — more verbose in `Development`, quieter
      elsewhere — the same environment-gating discipline as Path 02.
- [ ] Trigger the same event under `Development` and under a non-`Development` run, and confirm
      the verbosity difference is real and observable, not just configured and never actually
      exercised.

### M4 — Correlate Logs to a Single Request

- [ ] Confirm every log line written while handling one HTTP request carries the **same**
      correlation id as the `traceId` your Path 08 `ProblemDetails` responses already expose.
- [ ] Prove it end-to-end: trigger a request that fails, take the `traceId` from the response
      body, and find **every** log line for that exact request using only that id — nothing else.

**Edge case to verify:** trigger two different failing requests back-to-back and confirm their
`traceId`s (and therefore their log lines) are distinct — a correlation id that's accidentally
shared across requests is worse than having none at all.

### M5 — Health Checks: Liveness vs. Readiness

- [ ] Implement `/health/live` and `/health/ready` per the
      [Health Check Contract](#health-check-contract), with `/health/ready` performing a real
      database connectivity check.
- [ ] Write down, in your own words, a concrete scenario where conflating the two would cause a
      real operational problem — not a generic textbook answer, one that makes sense for this
      specific API.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Database reachable | `/health/live` → `200`; `/health/ready` → `200`. |
| Database temporarily unreachable (stop it, or point at a wrong connection string briefly) | `/health/live` → still `200` (the process itself is fine); `/health/ready` → `503`. |
| Restore the database | `/health/ready` → back to `200`, without restarting the app. |

### M6 — Don't Leak Infrastructure Details Through Health Checks

- [ ] Decide, deliberately, exactly what an anonymous caller sees in the body of both health
      endpoints — and make sure it's the minimal, safe version, not whatever your health check
      library defaults to (some default detailed-output formats include dependency names,
      durations, and even exception messages).
- [ ] Verify by actually calling both endpoints with no authentication and reading the raw
      response body yourself.

### M7 — A First Look at Distributed Tracing

- [ ] Add the OpenTelemetry SDK, instrumenting ASP.NET Core's request pipeline and EF Core
      automatically via their instrumentation packages, exporting to a simple local exporter
      (console is fine for this milestone).
- [ ] Trigger one request that involves a database query, and look at the resulting trace.
      Identify the parent span (the HTTP request itself) and at least one child span nested under
      it (an EF Core query) — this is the actual shape a trace takes, not just an abstract idea.

### M8 — Unify the Identifiers

- [ ] Confirm the trace id OpenTelemetry generates for a request, the correlation id in your
      structured logs (M4), and the `traceId` in the Path 08 `ProblemDetails` response are either
      the literal same value, or are explicitly and provably mapped to each other (e.g. one is
      derivable from the other in a documented way).
- [ ] This is the actual point of this whole path: one id, three views of the same request. If
      you have three unrelated ids floating around instead, that's not done yet.

### M9 — A First Metric

- [ ] Add exactly one counter metric using OpenTelemetry's metrics API — e.g. total books created
      — and confirm it increments when you exercise the corresponding endpoint.
- [ ] Resist adding a dozen metrics right now. One real metric you fully understand, wired to a
      real business event, is worth more here than ten copied from an example you didn't verify.

**Edge case to verify:** create three books in a row and confirm the counter increased by exactly
three, not by some other number — an off-by-one in a metric is easy to miss because nothing else
about the API looks wrong when it happens.

### M10 — Full Regression + Observability Sanity Pass

- [ ] Full functional regression across every Path 01–11 scenario — none of this path's changes
      should alter any existing behavior, only what's observable about it.
- [ ] New observability-specific pass: log correlation (M4) still works, both health endpoints
      (M5/M6) behave correctly including the "database down" case, a trace with a nested span is
      visible (M7), the identifiers unify (M8), and the metric increments (M9).

## Manual Test Script

This path mixes real HTTP requests with manual inspection of logs, health responses, and traces —
call those parts out explicitly rather than expecting a single automated script to prove them.

```http
@baseUrl = https://localhost:5001

### 1. Trigger a book creation - note the response and remember the request time
POST {{baseUrl}}/api/v1/books
Authorization: Bearer {{contributorToken}}
Content-Type: application/json

{ "title": "Observability Test Book", "publishedYear": 2020, "genre": "Mystery", "authorId": 1 }

### 2. Trigger a deliberate validation failure - grab the traceId from the response body
POST {{baseUrl}}/api/v1/books
Authorization: Bearer {{contributorToken}}
Content-Type: application/json

{ "publishedYear": 2020, "genre": "Mystery", "authorId": 1 }

### 3. Trigger a failed login - confirm no credentials end up logged
POST {{baseUrl}}/api/auth/login
Content-Type: application/json

{ "email": "contributor1@example.com", "password": "DefinitelyWrongPassword!" }

### 4. Anonymous health checks
GET {{baseUrl}}/health/live
GET {{baseUrl}}/health/ready
```

Manual (non-HTTP) inspection steps:

1. Using the `traceId` from request #2, find every log line for that exact request in your
   console/file sink — confirm they all share that same id.
2. Confirm the log entry from request #3 exists, is at `Warning` level, and contains **no**
   attempted password anywhere in it.
3. Open whatever you're using to view exported traces (even a console exporter's raw output) and
   find the trace for request #1 — identify the HTTP span and the EF Core query span nested
   under it.
4. Confirm your new book-created counter metric incremented by exactly one after request #1.
5. Temporarily make the database unreachable (stop it, or point at a bad connection string) and
   confirm `/health/live` still returns `200` while `/health/ready` returns `503` — then restore
   it and confirm `/health/ready` recovers without an app restart.

## Common Observability Pitfalls

Things worth checking on purpose, not just waiting to notice by accident:

- Logging the whole request or response body "just in case" — this is exactly how a password or
  a full JWT ends up sitting in a log file for days before anyone notices.
- A health check that returns `200` no matter what, because the check itself swallowed an
  exception instead of reporting the failure as unhealthy.
- Every log entry sitting at `Information` — the structured-logging equivalent of not having log
  levels at all, since nobody can filter signal from noise later.
- A trace exporter configured only for `Development`, silently producing no traces at all once
  you're running anywhere else.
- A correlation id that changes partway through a single request (e.g. something generates a new
  one instead of reusing what's already on the request) — this quietly breaks the entire point of
  M4 and M8.
- Health check endpoints that require authentication to call — liveness/readiness probes are
  normally called by infrastructure that can't, and shouldn't need to, log in first.

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Export traces to a real local tracing backend (e.g. Jaeger, Zipkin, or the .NET Aspire
      dashboard) instead of only a console exporter, once you're comfortable spinning up the extra
      container this requires (a light preview of Path 15).
- [ ] Add a couple more metrics (e.g. request duration, a gauge for in-flight requests), each tied
      to something you'd actually want to look at if this API were misbehaving.
- [ ] Add log redaction/sampling so large request or response bodies are never fully written to
      your logs verbatim.
- [ ] Write down (on paper — no need to actually wire up alerting yet) what alert you'd want fired
      if `/health/ready` started failing repeatedly, and who or what should see it.
- [ ] Add a structured log entry specifically for health check state **transitions** (healthy →
      unhealthy, and back) rather than only logging the current state on every poll — this is
      usually far more useful than a flood of "still healthy" entries.

## Definition of Done

- [ ] M1–M10 all checked off, in order, each with manual verification evidence.
- [ ] Every event in the [Logging Contract](#logging-contract) produces a structured log entry at
      the specified level, with no sensitive data anywhere in any of them.
- [ ] `/health/live` and `/health/ready` both exist, behave differently when the database is
      unreachable, and leak no infrastructure detail to an anonymous caller.
- [ ] A single `traceId` can be used to find the matching client response, the matching log
      lines, and the matching trace/span tree for one request.
- [ ] At least one real metric increments correctly when its corresponding action happens.
- [ ] The full Path 01–11 regression pass still succeeds, unchanged.

## Self-Review Checklist

- [ ] You personally searched your logs for a password or token and found none — not assumed your
      log calls were safe because you were careful while writing them.
- [ ] You can explain, in your own words and specific to this API, a real scenario where liveness
      and readiness genuinely need to be different checks.
- [ ] You verified the anonymous health check response bodies yourself, rather than trusting your
      library's defaults.
- [ ] You actually followed one `traceId` from a failed response, to logs, to a trace — not just
      configured the pieces and assumed they'd line up.
- [ ] Your one new metric is something you'd genuinely want to look at later, not something added
      only to satisfy this milestone.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `feat(m5): liveness and readiness health check endpoints`.
- [ ] Keep the logging (M1–M4), health check (M5–M6), and tracing/metrics (M7–M9) commits in
      separate groups — they're three related but distinct concerns sharing one path.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 13 optimizes
      performance on an API you can now actually observe while doing it.

## Reference Docs (use only when stuck)

Structured logging:
- [Serilog documentation](https://serilog.net/)
- [Logging in .NET](https://learn.microsoft.com/dotnet/core/extensions/logging)

Health checks:
- [Health checks in ASP.NET Core](https://learn.microsoft.com/aspnet/core/host-and-deploy/health-checks)

Tracing and metrics:
- [OpenTelemetry .NET documentation](https://opentelemetry.io/docs/languages/net/)
- [.NET observability with OpenTelemetry](https://learn.microsoft.com/dotnet/core/diagnostics/observability-with-otel)
