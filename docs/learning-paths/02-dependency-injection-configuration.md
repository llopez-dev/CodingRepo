# 02. Dependency Injection & Configuration — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the same Book
> Catalog API you built in
> [Path 01](01-aspnet-core-fundamentals.md). Build-it project, not a reading assignment — contracts,
> test requests, and checklists only. No C# solution code.

## Project Brief

This path doesn't add a new book-related feature. It rewires *how the pieces of the API you
already built are put together*, and adds a way to change runtime behavior from a config file
instead of recompiling. By the end of this path:

- The book store will be resolved everywhere through an interface, never through its concrete
  type directly.
- One real, observable behavior — how many books `GET /api/books` returns by default — will be
  driven entirely by `appsettings.json`, including a different value per environment.
- You'll have **hard, observable proof** — not just a description you read somewhere — of what
  Transient, Scoped, and Singleton actually mean, using a small diagnostic endpoint you build and
  then delete.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md)'s Definition of Done is fully satisfied. If
      `GET`/`POST`/`PUT`/`DELETE /api/books` don't already behave correctly, fix that first — this
      path rewires an existing working API, it doesn't build one from scratch.
- [ ] Your Path 01 manual test script still passes, unmodified, before you touch anything below.

## Rules

- No new book-domain endpoints or fields. Path 05 owns API design/DTOs, Path 03 owns real
  persistence. This path only touches **how services are wired and configured**.
- No hardcoded `if (something == "Development")` branches anywhere in your own logic to fake
  configuration — use the real configuration system (`appsettings.json` + per-environment
  overrides + `IWebHostEnvironment`).
- No FluentValidation or other validation libraries. Keep the manual checks from Path 01 for the
  book fields; the *configuration* validation you add in this path uses the Options pattern's own
  validation features, not a separate library.
- The diagnostics endpoint you build in this path is scaffolding to prove a point about lifetimes.
  It is not a permanent feature of the Book Catalog API — see the clean-up milestone at the end.

## Configuration Contract

Your options type must bind to a `BookCatalog` configuration section with exactly these two
values:

| Key | `appsettings.json` (default) | `appsettings.Development.json` (override) | Rule |
|---|---|---|---|
| `BookCatalog:DefaultPageSize` | `10` | `5` | integer, required, `>= 1` and `<= MaxPageSize` |
| `BookCatalog:MaxPageSize` | `50` | `50` | integer, required, `>= 1` |

Example `appsettings.json` fragment:

```json
{
  "BookCatalog": {
    "DefaultPageSize": 10,
    "MaxPageSize": 50
  }
}
```

Example `appsettings.Development.json` fragment (only the override, not the whole file):

```json
{
  "BookCatalog": {
    "DefaultPageSize": 5
  }
}
```

## Lifetime Marker Contract

A small, temporary part of the API whose only job is to make lifetimes observable over HTTP.

| Type | Registered lifetime | Purpose |
|---|---|---|
| One concrete marker implementation (name it yourself) | — | Generates one unique id (e.g. a `Guid`) in its constructor and exposes it as a read-only property. The same implementation is registered three separate times below — it doesn't need three different classes. |
| `ITransientMarker` | Transient | A new instance — and therefore a new id — every single time it's resolved. |
| `IScopedMarker` | Scoped | The same instance (same id) for every resolution within one HTTP request; a different instance for the next request. |
| `ISingletonMarker` | Singleton | The same instance (same id) for the entire lifetime of the running process. |
| A reporter/helper service (name it yourself) | Scoped or Transient — your choice | Takes all three markers as constructor dependencies, purely so the diagnostics endpoint can compare a "resolved directly in the handler" reading against a "resolved indirectly through another service" reading. |

## Suggested Project Structure

Additions on top of what Path 01 already gave you:

- [ ] An options class file matching the [Configuration Contract](#configuration-contract).
- [ ] The three marker interfaces plus their one shared concrete implementation.
- [ ] The reporter/helper service described above.
- [ ] A diagnostics-only endpoint (or route group) kept visibly separate from the book endpoints,
      so it's obvious it isn't part of the book domain.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Confirm the Store Is Behind an Interface

- [ ] If Path 01 didn't already do this, extract an interface (e.g. `IBookStore`) for the book
      store, and register the in-memory implementation against that interface, not its concrete
      type: the DI container should be told "when something asks for the interface, give it this
      implementation" rather than registering the concrete class directly.
- [ ] Confirm every place that uses the store (your endpoint handlers, any helper service) depends
      only on the interface. Search your own code for the concrete class name outside of the one
      line that registers it — it shouldn't appear anywhere else.
- [ ] Re-run the Path 01 manual test script to confirm this refactor changed nothing observable.

### M2 — Add Strongly-Typed Configuration

- [ ] Add the `BookCatalog` section to `appsettings.json` and the override to
      `appsettings.Development.json`, matching the [Configuration Contract](#configuration-contract)
      exactly.
- [ ] Create an options class matching that contract, and bind it using the Options pattern (look
      up how to bind a configuration section to a typed options class) rather than reading
      `IConfiguration["BookCatalog:DefaultPageSize"]` by hand wherever you need the value.
- [ ] Add startup validation so the app **fails fast** if the configuration is invalid (e.g.
      `DefaultPageSize` greater than `MaxPageSize`, or either value less than `1`). Look up how the
      Options pattern supports validation with a clear failure at startup rather than a confusing
      bug later at request time.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Valid configuration as specified in the contract | App starts normally. |
| `DefaultPageSize` set higher than `MaxPageSize` in `appsettings.json` | App **fails to start**, with an error that actually names the problem. |
| `DefaultPageSize` set to `0` or a negative number | App fails to start. |
| The `BookCatalog` section is missing entirely | Decide what should happen (sensible built-in defaults vs. fail fast) and be deliberate about it — don't leave it to accident. |

### M3 — Use the Configured Value in `GET /api/books`

- [ ] Add an optional `?take=` query parameter to `GET /api/books`.
- [ ] When `take` is not supplied, return `DefaultPageSize` books from the options value.
- [ ] When `take` is supplied and is between `1` and `MaxPageSize` inclusive, return that many
      (still never more than actually exist).
- [ ] When `take` is supplied and exceeds `MaxPageSize`, **clamp** it down to `MaxPageSize` rather
      than erroring — decide this and document it, this is your rule, not a framework default.
- [ ] When `take` is `0` or negative, this is invalid client input — return `400 Bad Request`.

**Edge cases to verify:**

| Request | Expected result |
|---|---|
| `GET /api/books` (no `take`) | Count equals the current `DefaultPageSize` (or fewer, if fewer books exist). |
| `GET /api/books?take=3` | Exactly 3 books (or fewer, if fewer exist). |
| `GET /api/books?take=1000` | Clamped to `MaxPageSize` — but still capped by however many books actually exist; clamping caps the *request*, it doesn't conjure extra books into existence. |
| `GET /api/books?take=0` | `400`. |
| `GET /api/books?take=-5` | `400`. |
| `GET /api/books?take=abc` (non-numeric) | Verify and document what actually happens — don't assume; this is a good moment to notice the difference between "parameter missing" and "parameter present but unparsable." |

### M4 — Seed Enough Data to Make This Observable

- [ ] Make sure the store has at least 12 books before you test M3/M5, so the default/clamp
      behavior is actually visible (12 is comfortably above `DefaultPageSize` in both
      environments, and comfortably below `MaxPageSize`).
- [ ] Either (a) seed this data automatically at startup — a good excuse to try gating it behind
      `IWebHostEnvironment.IsDevelopment()` so it never runs in a "production" run — or (b) just
      `POST` 12 books manually as the first step of your test script. Either is fine; pick one and
      be consistent, and note which one you picked.

### M5 — Prove the Environment Switch Works

- [ ] Run the app the normal way (`dotnet run`), which uses the `Development` launch profile by
      default, and confirm `GET /api/books` (no `take`) returns a count matching the
      **Development** default.
- [ ] Now force the **Production** configuration and confirm the count changes without touching a
      single line of code:
      - In PowerShell: `$env:ASPNETCORE_ENVIRONMENT = "Production"`, then run the app again in
        that same terminal, **or**
      - Publish/build and run the compiled output directly (bypassing `dotnet run` and
        `launchSettings.json` entirely), which defaults to the `Production` environment when no
        environment variable is set.
- [ ] Confirm the count reverts to the `Development` default once you unset the environment
      variable (or go back to `dotnet run`) and restart again.

### M6 — Introduce the Lifetime Markers

- [ ] Implement the marker type(s) and the reporter/helper service from the
      [Lifetime Marker Contract](#lifetime-marker-contract).
- [ ] Register the same concrete marker implementation three separate times, once per lifetime, so
      you end up with three independently resolvable services (`ITransientMarker`,
      `IScopedMarker`, `ISingletonMarker`).

### M7 — Build the Lifetime Diagnostics Endpoint

- [ ] Add `GET /api/diagnostics/lifetimes` that:
      - Resolves all three markers **directly** (e.g. as handler parameters).
      - Also calls the reporter/helper service, which resolves all three markers **indirectly**.
      - Returns a body reporting both readings for each lifetime, so you can compare them.

Example response shape (your field names may differ, but report both readings per lifetime):

```json
{
  "transient": { "fromHandler": "b2f1...", "fromReportedService": "9ac4..." },
  "scoped": { "fromHandler": "7e21...", "fromReportedService": "7e21..." },
  "singleton": { "fromHandler": "1d3f...", "fromReportedService": "1d3f..." }
}
```

### M8 — Prove Each Lifetime With Real HTTP Calls

- [ ] Call `GET /api/diagnostics/lifetimes` once and confirm, **within that single call**:
      - `transient.fromHandler` differs from `transient.fromReportedService`.
      - `scoped.fromHandler` **equals** `scoped.fromReportedService`.
      - `singleton.fromHandler` equals `singleton.fromReportedService` (trivially true, but
        confirm it anyway).
- [ ] Call it again as a **separate** request and compare against the first call's results:
      - Every transient value differs from every previous transient value.
      - The scoped value from this call differs from the scoped value in the previous call, even
        though both of this call's own readings match each other.
      - The singleton value is **identical** to the first call's singleton value.
- [ ] Restart the app and call it once more: the singleton value should now be different from
      before (new process, new singleton instance) — but stable across every request within this
      new run.

**Summary table to confirm you actually observed:**

| Lifetime | Same within one request? | Same across separate requests? | Same after a restart? |
|---|---|---|---|
| Transient | No | No | No |
| Scoped | Yes | No | No |
| Singleton | Yes | Yes | No |

### M9 — Full Regression Pass + Clean-up

- [ ] Re-run the entire Path 01 manual test script against the current code — confirm nothing
      about the CRUD behavior broke while you were rewiring DI and configuration.
- [ ] Re-run every new scenario from this path (default `take`, clamp, invalid `take`, environment
      switch, both lifetime calls).
- [ ] Gate or remove the diagnostics endpoint so it does **not** exist when the app is not running
      in `Development` (look up how to conditionally map an endpoint based on
      `IWebHostEnvironment`). Confirm this by running under `Production` and verifying the
      endpoint now returns a plain routing `404`.

## Manual Test Script

Run in order, against a freshly started instance. Steps that require restarting the process or
changing an environment variable are called out in plain text since they can't be expressed as a
single HTTP request.

```http
@baseUrl = https://localhost:5001

### 1. (Prerequisite) Confirm at least 12 books exist — seed at startup, or POST them here first.
GET {{baseUrl}}/api/books?take=100

### 2. Default take, Development environment -> count == Development DefaultPageSize (5)
GET {{baseUrl}}/api/books

### 3. Explicit valid take -> exactly 3 results
GET {{baseUrl}}/api/books?take=3

### 4. Take far above MaxPageSize -> clamped to MaxPageSize (50), still limited by actual data
GET {{baseUrl}}/api/books?take=1000

### 5. Take = 0 -> 400
GET {{baseUrl}}/api/books?take=0

### 6. Negative take -> 400
GET {{baseUrl}}/api/books?take=-5

### 7. Non-numeric take -> observe and document actual behavior
GET {{baseUrl}}/api/books?take=abc

### 8. First lifetime diagnostics call -> record all six values
GET {{baseUrl}}/api/diagnostics/lifetimes

### 9. Second lifetime diagnostics call (same run) -> compare against step 8
GET {{baseUrl}}/api/diagnostics/lifetimes
```

Manual (non-HTTP) steps to run alongside the script above:

1. Before step 2: confirm you're running via `dotnet run` (Development profile).
2. After step 9: stop the app, set `ASPNETCORE_ENVIRONMENT=Production` (PowerShell:
   `$env:ASPNETCORE_ENVIRONMENT = "Production"`), restart, and repeat request #2 — the count
   should now match the **Production** default (10), not the Development default (5).
3. Repeat request #8 once more after the Production restart — the singleton value must differ
   from step 8's singleton value (new process), while still matching itself if you call it twice
   in this new run.
4. Unset the environment variable (or open a fresh terminal) and confirm `dotnet run` goes back to
   the Development default.

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Swap your options injection from `IOptions<T>` to `IOptionsSnapshot<T>` for one reading, edit
      `appsettings.json` **while the app is still running**, and confirm the snapshot picks up the
      new value on the next request without a restart — while a value captured through
      `IOptions<T>` at startup does not change.
- [ ] Try `IOptionsMonitor<T>` and its change-notification callback; log a line every time the
      configuration changes on disk.
- [ ] Deliberately create a **captive dependency**: make the Singleton marker's implementation take
      the Scoped marker as a constructor dependency. Run the app in `Development` and observe the
      validation error ASP.NET Core gives you at startup. Explain *why* this is dangerous (what
      would happen if the container let it through silently), then remove the bad dependency.
- [ ] Replace the DataAnnotations-based config validation with a custom validator (look up the
      interface the Options pattern expects for hand-written validation logic).

## Definition of Done

- [ ] M1–M9 all checked off, in order, each with manual test evidence.
- [ ] The store is referenced only through its interface outside of the single DI registration
      line.
- [ ] An invalid `BookCatalog` configuration reliably prevents the app from starting.
- [ ] `GET /api/books` correctly applies default, explicit, and clamped `take` values, and rejects
      invalid ones with `400`.
- [ ] The Development vs. Production default page size difference is proven with an actual restart
      and environment-variable change, not just asserted.
- [ ] All three lifetimes are proven with real HTTP calls matching the
      [summary table](#m8--prove-each-lifetime-with-real-http-calls), across both a single request
      and multiple requests.
- [ ] The diagnostics endpoint no longer exists outside of `Development`.
- [ ] The full Path 01 regression pass still succeeds.

## Self-Review Checklist

- [ ] You can point to the exact registration line for the store and explain why depending on the
      interface (rather than the concrete type) matters here.
- [ ] You can explain, in terms of *this specific app*, why a Singleton depending on a Scoped
      service is a real bug and not just a style preference.
- [ ] Nothing reads a configuration value by string key (`IConfiguration["..."]`) anywhere except
      the one place that binds the options class.
- [ ] The clamping and validation rules for `take` live in one place, not duplicated.
- [ ] You did not leave the diagnostics endpoint reachable in a non-Development run.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as Path 01, e.g.
      `feat(m3): configurable default page size for GET /api/books`.
- [ ] Keep the "prove the lifetimes" commits (M6–M8) separate from the "use configuration for
      `take`" commits (M2–M3) — they're two different concepts sharing one path, and a reviewer
      (including future you) will thank you for not tangling them together.
- [ ] Note the commit where this path's Definition of Done is satisfied, the same way you did for
      Path 01 — Path 03 will build directly on top of it.

## Reference Docs (use only when stuck)

Dependency injection basics:
- [Dependency injection in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/dependency-injection)
- [Service lifetimes](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection#service-lifetimes)

Configuration & environments:
- [Configuration in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/configuration/)
- [Use multiple environments in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/environments)

The Options pattern:
- [Options pattern in ASP.NET Core](https://learn.microsoft.com/dotnet/core/extensions/options)
- [Options validation](https://learn.microsoft.com/dotnet/core/extensions/options-validation)

For the stretch goals:
- [IOptionsMonitor and change notifications](https://learn.microsoft.com/dotnet/core/extensions/options#reload-configuration-data-with-ioptionsmonitor)
