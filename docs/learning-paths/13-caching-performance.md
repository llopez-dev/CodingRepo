# 13. Caching & Performance — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 12](12-logging-observability.md). Build-it project — contracts, test requests, and
> checklists only. No C# solution code.

## Project Brief

"Faster" is a claim, not a feeling — this path is about proving it with numbers, not assuming it.
You'll measure the book list endpoint's real performance **before** touching anything, add
in-memory caching, get cache **keys** genuinely right for a parameterized endpoint (a mistake here
silently serves the wrong data to the wrong request), move to a distributed cache so this would
still work if the API ever ran as more than one process, add HTTP-level response caching without
ever caching anything personalized, audit your own code for async/await mistakes, and finish by
re-running the exact same measurement from the start to prove the difference is real.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 12](12-logging-observability.md)
      are fully done — you'll use Path 12's logging/tracing to actually observe what's slow, and
      Path 09's authenticated endpoints are exactly what you must be careful never to cache
      incorrectly.

## Rules

- **No new business features.** Everything here makes existing behavior faster or more scalable —
  it must never change what the *correct* answer is.
- **Correctness always wins over speed.** If a cache ever causes a wrong or stale-beyond-policy
  answer, that's a bug to fix, not an acceptable trade-off for performance.
- **Never cache a personalized or authenticated response** at the HTTP/response-cache level (e.g.
  Path 09's `/api/auth/me`). Anonymous, public data (the book/author catalog) is fair game.
- **Measure before and after every significant change.** "It feels faster" is not evidence.

## Worked Example

### A properly composed cache key

The paginated, filtered, sorted book list (Path 05) depends on **every** query parameter — the
cache key must reflect all of them, or two different requests will silently collide:

```
books:v1:page=2:pageSize=10:genre=fantasy:sortBy=title:sortDir=asc
```

Compare that to a cache key that looks reasonable but is actually a serious bug:

```
books:v1
```

This second key would serve **the same cached response** to a request for page 1 and a request
for page 2 — the cache doesn't know they're different requests, because you never told it.

### Response caching headers — public data vs. personalized data

The anonymous book list, safe to cache at the HTTP level:

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=60
Content-Type: application/json
```

`GET /api/auth/me` (Path 09) — must **never** look like this:

```http
HTTP/1.1 200 OK
Cache-Control: no-store
Content-Type: application/json
```

### A before/after measurement report — the actual shape M1 and M7 produce

| Metric | Before (M1 baseline) | After (M7 re-run) |
|---|---|---|
| Requests/sec | *(your real number)* | *(your real number)* |
| p50 latency | | |
| p95 latency | | |
| p99 latency | | |
| Error rate | | |

Fill this in with your own real numbers twice — once before any change in this path, once after
every milestone is done. A path that ends without this table filled in with real numbers hasn't
actually proven anything yet.

## Common Performance & Caching Pitfalls

Things worth checking on purpose, not just waiting to discover in production:

- A cache key missing even one query parameter that affects the response — the single most common
  way caching silently returns wrong data instead of failing loudly.
- Caching a personalized or authenticated response, even by accident (e.g. a shared response-cache
  policy applied too broadly and accidentally catching an authenticated route).
- No expiration at all — a cache entry that never invalidates will serve stale data forever, long
  after anyone remembers it's there.
- Caching error responses (a `404` or `500`) for as long as a normal success response — a
  transient failure shouldn't get "stuck" being served to everyone for your full cache duration.
- Blocking on async code (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`) anywhere in a request
  path — this can tie up thread-pool threads and hurt throughput under load in ways that don't
  show up until you actually measure concurrency.
- Wrapping already-request-scoped work in `Task.Run` just to "make it async" — this doesn't move
  the work off anything meaningful in ASP.NET Core and just spends an extra thread-pool thread.
- Optimizing anything before you have a real baseline measurement to compare against.
- **Cache stampede**: a popular cache entry expires, and many concurrent requests all miss at the
  same instant, hammering the database simultaneously — worth knowing about even if you don't
  fully solve it in this path (see stretch goals).

## Choosing What's Worth Caching

Not everything deserves a cache entry. Before adding one, ask:

| Question | If the answer is yes... |
|---|---|
| Is this read far more often than it changes? | Good caching candidate. |
| Is computing/fetching it genuinely expensive (a real query, not a trivial lookup)? | Good caching candidate — caching something already fast just adds complexity for no real gain. |
| Does it ever contain data specific to one user? | Do **not** cache it at the HTTP/response-cache level; a scoped, per-user cache key is a different, more advanced problem this path doesn't require. |
| Would slightly stale data actually cause a real problem for someone using it? | If yes, lean toward active invalidation (M4) over a long time-based expiration. |

## Suggested Project Structure

- [ ] A small caching abstraction wrapping whichever cache you're using at the time (`IMemoryCache`
      first, then `IDistributedCache`), so the endpoints calling into it don't change shape when
      the backing implementation changes underneath — the same lesson as Path 04's repositories,
      applied to caching.
- [ ] A documented, consistent cache key format (see the [Worked Example](#worked-example)) used
      everywhere you cache a parameterized result.
- [ ] Whatever load-testing tool you pick (a lightweight local tool is enough — look up a current,
      simple option), plus a place to record before/after results (e.g. `docs/performance.md`).
- [ ] Redis running locally via a single Docker command for M4 — not a full containerized setup
      of your own API; that's Path 15.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Establish a Baseline (Measure First)

- [ ] Pick a lightweight local load-testing tool (look up a current, simple option) and run it
      against a realistic, representative request — e.g. `GET /api/v1/books` with a typical
      filter/sort/page combination — **before making any change in this path**.
- [ ] Record real numbers at a chosen concurrency level: requests/sec, p50/p95/p99 latency, and
      error rate, using the [before/after table](#a-beforeafter-measurement-report--the-actual-shape-m1-and-m7-produce)
      shape.
- [ ] This baseline is what every later milestone gets compared against. Skipping it means M7 has
      nothing real to prove anything against.

### M2 — In-Memory Caching for a Real Hotspot

- [ ] Add `IMemoryCache` caching for the authors-with-book-count projection (Path 03 M9) — read
      often, doesn't change every second, and has no query parameters to get wrong yet.
- [ ] Choose and apply a reasonable time-based expiration, and write down why you picked that
      specific duration for this specific data.
- [ ] Measure the difference yourself: call the endpoint twice in a row and confirm the second
      (cached) call is noticeably faster than the first (cold) call.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| First call after the cache is empty (cold) | Normal (uncached) latency. |
| Second call within the expiration window | Noticeably faster. |
| A call made right after the entry expires | Back to cold latency once, then fast again. |

### M3 — Cache Key Discipline for a Parameterized Endpoint

- [ ] Extend caching to the paginated/filtered/sorted book list endpoint (Path 05) — a much
      trickier target than M2, because the correct response depends on **every** query parameter:
      `page`, `pageSize`, `genre`, `authorId`, `publishedYearMin`/`Max`, `sortBy`, `sortDir`.
- [ ] Build your cache key from **all** of those parameters, following the
      [worked example](#a-properly-composed-cache-key) — a key missing even one parameter is a
      real bug, not a simplification.
- [ ] Verify deliberately: request page 1 and page 2 back-to-back (same other filters) and confirm
      you get genuinely **different** results, not the same cached page served twice.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Same page/filter/sort requested twice | Second call served from cache, same data. |
| Same page, different `sortDir` | Treated as a different cache entry, correctly. |
| Different `genre` filter, same page number | Treated as a different cache entry, correctly. |

### M4 — Confront Cache Invalidation

- [ ] Decide, deliberately: is time-based expiration alone good enough here, or does
      creating/updating/deleting an author or book need to actively invalidate the relevant cache
      entries so users never see stale data longer than necessary?
- [ ] Implement whichever you decide, and write down your reasoning — there's a reason cache
      invalidation has a reputation as one of the genuinely hard problems in this field, and you
      just had to make a real decision about it, not just read about it.
- [ ] Verify: create a new author, and confirm the cached authors-with-book-count list reflects it
      either immediately (active invalidation) or within your documented staleness window
      (time-based expiration only) — not "eventually, whenever."

### M5 — Move to a Distributed Cache (Redis)

- [ ] Run Redis locally via a single Docker command (a full containerized setup of your *own* API
      is Path 15's job — this milestone only needs the cache itself running).
- [ ] Swap your caching code from `IMemoryCache` to `IDistributedCache` backed by Redis, without
      changing the public shape of the caching abstraction from M2/M3 — the callers shouldn't need
      to know or care which one is behind it.
- [ ] Write down, specifically for this API, why an in-memory cache stops being sufficient the
      moment this runs as more than one process: each instance would have its own, potentially
      inconsistent, cache — a distributed cache is shared across all of them.

### M6 — HTTP-Level Response Caching

- [ ] Add response caching (`Cache-Control` headers, ASP.NET Core's response caching middleware)
      to the anonymous, public book/author `GET` endpoints, per the
      [worked example](#response-caching-headers--public-data-vs-personalized-data).
- [ ] Explicitly confirm **nothing** that returns personalized or authenticated data (e.g.
      `/api/auth/me`) is ever response-cached — check this yourself by reading the actual response
      headers on that endpoint, don't just assume your policy is scoped correctly.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Anonymous `GET /api/v1/books` | `Cache-Control: public, max-age=...` present. |
| `GET /api/auth/me` with a valid token | `Cache-Control: no-store` (or equivalent), never cacheable. |
| A write endpoint (`POST`/`PUT`/`DELETE`) | Never cached, regardless of any general policy. |

### M7 — Async/Await Audit

- [ ] Audit your own code against the [Common Pitfalls](#common-performance--caching-pitfalls)
      list above, specifically looking for: blocking-on-async calls, unnecessary `Task.Run`
      wrapping around request-scoped work, and any fire-and-forget code that could silently
      swallow an exception.
- [ ] Fix everything you find, and write down what you found (even "nothing" is a valid, useful
      finding if you genuinely looked).

### M8 — Prove It With Numbers (Re-run the Baseline)

- [ ] Re-run the **exact same** load test from M1 — same endpoint, same request shape, same
      concurrency level — against the now-cached, now-audited API.
- [ ] Fill in the "After" column of your
      [before/after table](#a-beforeafter-measurement-report--the-actual-shape-m1-and-m7-produce)
      with real numbers, and confirm a genuine, measurable improvement — not a guess that it's
      probably faster now.

### M9 — Full Regression + Cache Correctness Pass

- [ ] Full functional regression across every Path 01–12 scenario — caching must never have
      changed a single correct answer, only how quickly it arrives.
- [ ] Specifically re-verify: no cached response is ever stale beyond your documented policy, no
      personalized/authenticated data ever appears in a shared cache, and every cache key from M3
      still correctly reflects every relevant query parameter.

## Manual Test Script

Part of this path is real HTTP requests, part of it is running an external load-testing tool and
reading its output — call those out explicitly.

```http
@baseUrl = https://localhost:5001

### 1. Cold call to the authors list (before caching, or right after cache expiry)
GET {{baseUrl}}/api/v1/authors

### 2. Immediately repeat - should be noticeably faster once caching is in place
GET {{baseUrl}}/api/v1/authors

### 3. Page 1 of the book list
GET {{baseUrl}}/api/v1/books?page=1&pageSize=5&genre=fantasy

### 4. Page 2, same filters - must return DIFFERENT data, not a cached copy of page 1
GET {{baseUrl}}/api/v1/books?page=2&pageSize=5&genre=fantasy

### 5. Confirm response caching headers on the anonymous list endpoint
GET {{baseUrl}}/api/v1/books

### 6. Confirm the personalized endpoint is never cacheable
GET {{baseUrl}}/api/auth/me
Authorization: Bearer {{contributorToken}}

### 7. Create a new author, then re-check the (possibly cached) authors list
POST {{baseUrl}}/api/v1/authors
Authorization: Bearer {{contributorToken}}
Content-Type: application/json

{ "name": "Cache Invalidation Test Author" }

### 8. Re-fetch authors and confirm your M4 decision (immediate or within-window) actually holds
GET {{baseUrl}}/api/v1/authors
```

Manual (non-HTTP) steps:

1. Run your chosen load-testing tool against `GET /api/v1/books` **before** starting this path's
   changes (M1) and record the numbers.
2. Re-run the identical load test after finishing M1–M7 (M8) and record the numbers again.
3. Compare the two runs side by side and confirm the difference is real, not noise — if it isn't
   clearly better, figure out why before declaring this path done.

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Address cache stampede directly (e.g. a "lock and recompute once" pattern, or staggered
      expiration) for your highest-traffic cached endpoint, and prove it with a load test that
      specifically targets the moment of expiration.
- [ ] Add `ETag`-based conditional requests (`If-None-Match` → `304 Not Modified`) for at least one
      endpoint, and measure how much bandwidth it actually saves for a repeat caller.
- [ ] Benchmark a specific hot piece of C# logic in isolation (not the whole HTTP pipeline) with a
      micro-benchmarking tool, to separate "the network/HTTP overhead" from "the actual code."
- [ ] Deliberately introduce one of the async/await pitfalls from this path's own checklist into a
      throwaway branch, load-test it, and see the actual measurable impact — then revert it. Seeing
      the damage a bad pattern causes, with numbers, is more convincing than reading that it's bad.
- [ ] Add a second, independent load-testing tool and confirm it reports roughly the same numbers
      as your first choice — a good habit before trusting any single tool's output too heavily.

## Definition of Done

- [ ] M1–M9 all checked off, in order, each with real, recorded evidence.
- [ ] The [before/after measurement table](#a-beforeafter-measurement-report--the-actual-shape-m1-and-m7-produce)
      is filled in with genuine numbers on both sides, showing a real improvement.
- [ ] Every parameterized cache key accounts for every query parameter that affects the response.
- [ ] A deliberate, written decision exists for cache invalidation (time-based vs. active), backed
      by reasoning specific to this API.
- [ ] The distributed cache (Redis) works correctly behind the same abstraction used for the
      in-memory cache.
- [ ] No personalized or authenticated response is ever response-cached — verified by reading
      actual response headers.
- [ ] The async/await audit found and fixed everything it was going to find (even if that's
      nothing) — and you can say what you checked.
- [ ] The full Path 01–12 regression pass still succeeds, with caching never changing a correct
      answer.

## Self-Review Checklist

- [ ] You can produce your own before/after numbers on request — not recall them from memory as
      "it got faster."
- [ ] You can explain exactly what's in one of your cache keys and why every piece of it is there.
- [ ] You checked the actual response headers on `/api/auth/me` yourself and confirmed it's
      genuinely never cacheable — not assumed your policy scoping was correct.
- [ ] You know, specifically, what your cache invalidation policy is for at least one cached
      resource, and why you chose it over the alternative.
- [ ] You searched your own code for blocking-on-async calls rather than assuming you never wrote
      any.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `perf(m3): cache key includes all book list query parameters`.
- [ ] Commit your M1 baseline numbers and your M8 after numbers as their own commits (e.g. in
      `docs/performance.md`), so the improvement is visible in history, not just asserted in a
      commit message.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 14 adds background
      processing on top of an API whose synchronous performance you now actually understand.

## Reference Docs (use only when stuck)

Caching:
- [Cache in-memory in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/caching/memory)
- [Distributed caching in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/caching/distributed)
- [Response caching middleware](https://learn.microsoft.com/aspnet/core/performance/caching/middleware)
- [Overview of caching in ASP.NET Core](https://learn.microsoft.com/aspnet/core/performance/caching/overview)

Async/await:
- [Async/await best practices in ASP.NET Core](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/async-scenarios)

Performance measurement:
- [ASP.NET Core performance best practices](https://learn.microsoft.com/aspnet/core/performance/performance-best-practices)
- [BenchmarkDotNet documentation (for stretch goals)](https://benchmarkdotnet.org/articles/overview.html)

Redis:
- [Redis quick start](https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/docker/)
