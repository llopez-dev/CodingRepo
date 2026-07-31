# 14. Background Processing — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 13](13-caching-performance.md). Build-it project — contracts, test requests, and
> checklists only. No C# solution code.

## Project Brief

Not everything a request triggers needs to finish before the response goes out, and not
everything your API does needs to be triggered by a request at all. This path adds two genuinely
different kinds of background work: a **periodic** job that runs on its own schedule and logs a
summary of books added recently, and a **queued** job — moving Path 13's cache-refresh work out of
the request path entirely, so creating a book doesn't have to wait for the cache to catch up. Along
the way you'll hit the two mistakes almost everyone makes the first time they write a hosted
service in ASP.NET Core: injecting a scoped dependency directly into it, and letting one bad
execution quietly take down the whole host.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 13](13-caching-performance.md) are
      fully done — this path reuses Path 02's Options pattern for configuration, Path 12's
      structured logging, and Path 13's cache-refresh logic directly.

## Rules

- **No new business features** beyond the periodic summary job and the queued cache refresh —
  this path is about *how* existing work gets scheduled and executed, not new things the API does.
- **Never inject a scoped service directly into a singleton-registered hosted service.** Always
  create a new scope per unit of work via `IServiceScopeFactory`.
- **A single failed background execution must never crash the whole host.** One bad run should be
  logged and survived, not fatal.
- **Document, explicitly, that the in-memory queue in this path is not durable.** If the process
  crashes with work still queued, that work is gone. Don't let this be discovered by accident.

## Worked Example

### Structured log lines for the periodic job's lifecycle

Startup:

```json
{ "Level": "Information", "MessageTemplate": "Book summary background service starting" }
```

One scheduled run:

```json
{
  "Level": "Information",
  "MessageTemplate": "Book summary: {Count} books added since {SinceUtc}",
  "Properties": { "Count": 3, "SinceUtc": "2026-07-31T12:00:00Z" }
}
```

Shutdown:

```json
{ "Level": "Information", "MessageTemplate": "Book summary background service stopping" }
```

### The captive-dependency error you should deliberately reproduce once (M3)

Something along these lines, from the built-in dependency injection container, when scope
validation is enabled (the default in `Development`):

```
System.InvalidOperationException: Cannot resolve scoped service 'IBookRepository' from root provider.
```

Seeing this once, on purpose, is the point — it's the exact same class of problem Path 02's
captive-dependency stretch goal warned about, now showing up somewhere it actually matters.

### A queued work item's lifecycle

```
Request arrives -> handler enqueues "refresh authors cache" -> handler returns response immediately
                                                                     |
                                                                     v
                                        Background service dequeues the item, some time later
                                                                     |
                                                                     v
                                              Cache is refreshed; the client never waited for this
```

### Durability: acceptable vs. not acceptable to lose

| Queued work | Acceptable to lose on a crash? | Why |
|---|---|---|
| Refresh the authors-with-book-count cache | Yes | The cache is just a performance optimization — a missed refresh self-heals on the next successful read or write. |
| "Charge this customer's card" (a hypothetical, **not** something this project does) | No | Losing it silently means a customer never gets billed, or worse, gets billed inconsistently — this needs a genuinely durable queue, which is out of scope for this path. |

## Periodic vs. Queued: A Decision Framework

| Question | Lean toward |
|---|---|
| Does the work need to happen on a schedule, regardless of whether anything triggered it? | Periodic `BackgroundService` (M2). |
| Does the work only make sense in reaction to something a request just did? | Queued work item (M6). |
| Is it fine if this runs slightly late, or even not at all this one time? | Either — this project's in-memory queue and simple timer both accept that. |
| Would losing this work silently, even occasionally, be a real problem? | Neither, as built in this path — you'd need a durable queue or scheduler, which is a stretch goal, not the core scope. |

## Common Background-Processing Pitfalls

- Injecting a scoped service (a repository, a `DbContext`) directly into a singleton-registered
  hosted service's constructor, instead of resolving it from a new scope per execution.
- Letting an unhandled exception inside a `BackgroundService`'s execution loop propagate all the
  way out — by default this can stop the **entire host**, not just that one background job.
- Ignoring the `CancellationToken` passed to your service, so shutdown either hangs waiting for it
  or kills it mid-operation instead of stopping cleanly.
- Treating an in-memory queue as if it were durable — assuming queued work always eventually runs,
  even across a crash or restart.
- An unbounded queue that can grow forever under sustained load, quietly consuming memory instead
  of applying backpressure or shedding load on purpose.
- Doing genuinely CPU-heavy work inside a hosted service without remembering it competes for
  resources with the threads handling real HTTP requests on the same process.

## Suggested Project Structure

- [ ] A `BackgroundService` for the periodic book summary job, configured via the Options pattern
      (Path 02) rather than a hardcoded interval.
- [ ] An `IBackgroundTaskQueue`-style abstraction (a simple in-memory channel underneath) plus a
      second `BackgroundService` that continuously dequeues and executes work items.
- [ ] Both hosted services registered as singletons, each using `IServiceScopeFactory` internally
      whenever they need a scoped dependency — never receiving one directly through their own
      constructor.
- [ ] A short note (e.g. in `docs/background-processing.md`) documenting the queue's non-durability
      and which use cases that is and isn't acceptable for.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Your First Hosted Service (Prove the Lifecycle First)

- [ ] Add a minimal `BackgroundService` that does nothing but log "starting" on startup and
      "stopping" on shutdown, using Path 12's structured logging.
- [ ] Start the app, confirm the "starting" log line appears; stop it, confirm "stopping" appears
      — in that order, actually tied to the app's real lifecycle, before this service does
      anything meaningful.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Normal `dotnet run` startup and a clean stop (Ctrl+C) | Both log lines appear, in order. |
| Restarting the app several times in a row | Both lines appear every time, never duplicated or missing. |

### M2 — The Periodic Summary Job

- [ ] Turn the M1 skeleton into a real job that runs on a configurable interval (Options pattern,
      Path 02) and logs a structured summary of books created since its last run, per the
      [worked example](#structured-log-lines-for-the-periodic-jobs-lifecycle).
- [ ] Use a short interval for local testing (e.g. 30 seconds) while keeping the configuration
      shaped so a real "once a day" production value is just a config change, not a code change.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| No books created since the last run | A summary log entry still fires, with `Count: 0` — silence isn't the same as "confirmed nothing happened." |
| Several books created since the last run | `Count` matches exactly. |
| Interval changed via configuration | The job's actual cadence changes without a code change or rebuild. |

### M3 — Handling Scoped Dependencies from a Singleton Service

- [ ] First, **deliberately** try injecting a scoped repository directly into the hosted service's
      constructor, run the app in `Development`, and reproduce the
      [captive-dependency error](#the-captive-dependency-error-you-should-deliberately-reproduce-once-m3)
      yourself.
- [ ] Fix it properly: resolve the scoped dependency from a **new scope**, created per execution,
      via `IServiceScopeFactory` — not once at startup, and not directly through the constructor.
- [ ] Confirm the job still produces correct summaries after the fix.

### M4 — Don't Let One Bad Run Kill the Whole Host

- [ ] Research what happens, by default, if your periodic job's per-execution logic throws an
      unhandled exception — confirm for yourself whether it's scoped to just that job or can bring
      down the entire host.
- [ ] Wrap each execution so a failure is logged (Path 12, at `Error` level) and the service
      survives to run again on its next scheduled interval, rather than taking the application
      down.
- [ ] Verify by deliberately throwing on purpose inside one execution (a temporary, obvious fake
      failure) and confirming: the app is still running afterward, the failure was logged, and the
      job successfully runs again on its next scheduled interval.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| One execution throws | Logged at `Error`, host keeps running. |
| The very next scheduled execution after a failure | Runs normally, as if nothing happened. |
| Two failures in a row | Both logged independently; host still alive after both. |

### M5 — Introduce the Background Work Queue

- [ ] Build a simple in-memory work queue (a channel-based `IBackgroundTaskQueue`-style
      abstraction) plus a second hosted service that continuously dequeues and executes items.
- [ ] Get one trivial, harmless queued work item (just a log line) working end-to-end — enqueue it
      from an endpoint, confirm it executes shortly after, and confirm the endpoint's response
      wasn't delayed waiting for it — before wiring in anything that matters.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Enqueue one item, then immediately check the endpoint's response time | Response returns without waiting for the queued item to execute. |
| Enqueue several items in quick succession | All of them eventually execute, in the order you'd expect for a simple queue. |

### M6 — Queue Real Work: Asynchronous Cache Refresh

- [ ] Change the book/author create, update, and delete endpoints to **enqueue** a
      "refresh the authors-with-book-count cache" work item (Path 13) instead of performing that
      refresh inline during the request.
- [ ] Confirm the write endpoints now return **without** waiting for the cache refresh to
      complete.
- [ ] Explicitly compare this against Path 13's original invalidation decision: what did you gain
      (a faster write response), and what did you give up (a brief window where the cache can be
      stale between the write completing and the queued refresh actually running)? Write down
      whether that trade-off is worth it for this specific data.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Create a book, immediately re-check the authors list | May briefly show the pre-refresh count — document how briefly, don't just assume. |
| Create a book, wait past the queue's typical processing time, re-check | Shows the updated count. |
| Several writes queued in quick succession | All eventually get processed; none are silently dropped under normal operation. |

### M7 — Understand This Queue's Real Limitations

- [ ] Deliberately verify the queue's lack of durability: enqueue some work, then stop the process
      before it's processed, and confirm the item is simply gone on restart — not retried, not
      recovered.
- [ ] Write down, per the [durability comparison](#durability-acceptable-vs-not-acceptable-to-lose)
      above, why this is an acceptable trade-off for the cache-refresh use case specifically, and
      what kind of work it would **not** be acceptable for.

### M8 — Respect Graceful Shutdown

- [ ] Confirm both hosted services (the periodic job and the queue processor) respect the
      cancellation token they're given on shutdown, rather than hanging shutdown indefinitely or
      being killed mid-operation.
- [ ] Verify by shutting the app down while a job is mid-run (or work is still queued) and
      observing what actually happens to it.

### M9 — Full Regression + Background Work Verification

- [ ] Full functional regression across every Path 01–13 scenario — none of this path's changes
      should alter any existing correct behavior.
- [ ] New checks specific to this path: the periodic job runs and logs on schedule, scoped
      dependency resolution works correctly inside it (no captive-dependency errors), queued
      cache-refresh happens for creates/updates/deletes without blocking the response, a
      deliberate exception doesn't kill the host, and shutdown is graceful.

## Manual Test Script

Much of this path is verified by watching logs and timing, not just request/response bodies — the
non-HTTP steps below are just as important as the requests themselves.

```http
@baseUrl = https://localhost:5001

### 1. Create a book - note the response time, and that it doesn't wait for a cache refresh
POST {{baseUrl}}/api/v1/books
Authorization: Bearer {{contributorToken}}
Content-Type: application/json

{ "title": "Background Processing Test Book", "publishedYear": 2020, "genre": "Mystery", "authorId": 1 }

### 2. Immediately check the authors list - may still show the pre-refresh count
GET {{baseUrl}}/api/v1/authors

### 3. Check again after a short wait - should now reflect the update
GET {{baseUrl}}/api/v1/authors
```

Manual (non-HTTP) steps:

1. Watch your logs across at least one full interval of the periodic job (M2) and confirm a
   summary log entry appears on schedule, with an accurate `Count`.
2. Temporarily inject a scoped dependency directly into a hosted service's constructor, run in
   `Development`, and confirm you see the captive-dependency error yourself — then revert to the
   correct `IServiceScopeFactory`-based approach.
3. Temporarily throw inside one execution of the periodic job, confirm the app survives and logs
   the failure, then confirm the very next scheduled run completes normally — then remove the
   deliberate failure.
4. Stop the app with a queued work item not yet processed, restart it, and confirm the item is
   gone (proving the queue's non-durability) rather than picked back up.
5. Start a shutdown while a job is mid-execution and confirm it stops cleanly rather than hanging
   or being killed abruptly.

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Add a bounded queue capacity and decide, deliberately, what happens when it's full — the
      enqueue call waits, or the new item is rejected/dropped — and justify your choice.
- [ ] Add a metric (Path 12) counting queued items processed and another counting failures, so
      this background work is observable the same way your HTTP endpoints already are.
- [ ] Replace the in-memory queue with a durable alternative (e.g. a simple database-backed outbox
      table) for the cache-refresh use case specifically, and decide honestly whether the added
      complexity is actually worth it here — it's a legitimate answer to conclude it isn't, for
      this particular piece of work.
- [ ] Add a second periodic job with a genuinely different schedule and confirm both run correctly
      and independently, without one's failure affecting the other's.
- [ ] Add a startup delay/jitter to the periodic job so, if this API ever ran as multiple
      instances, they wouldn't all fire their periodic job at exactly the same moment.

## Definition of Done

- [ ] M1–M9 all checked off, in order, each with manual verification evidence.
- [ ] The periodic summary job runs on a configurable schedule and logs an accurate count every
      time, including a zero count.
- [ ] No hosted service ever has a scoped dependency injected directly into its constructor — all
      go through `IServiceScopeFactory`.
- [ ] A deliberately thrown exception inside a background execution is logged and survived, not
      fatal to the host.
- [ ] Book/author writes enqueue their cache refresh instead of performing it inline, and are
      measurably faster for not waiting on it.
- [ ] The queue's lack of durability has been proven (not just assumed) and documented, along with
      which use cases that is and isn't acceptable for.
- [ ] Both hosted services shut down gracefully, verified by actually triggering a shutdown
      mid-operation.
- [ ] The full Path 01–13 regression pass still succeeds.

## Self-Review Checklist

- [ ] You personally reproduced the captive-dependency error before fixing it — not just written
      the correct code from memory of having read about the problem.
- [ ] You personally watched the host survive a deliberate exception inside a background
      execution, rather than assuming your `try`/`catch` placement was correct.
- [ ] You can state, specifically, how much faster a book-creation write became after moving the
      cache refresh to the queue — not just "it feels faster."
- [ ] You actually lost a queued item on purpose (by stopping the process) and confirmed it, rather
      than assuming the queue works the way you configured it to.
- [ ] You triggered a real shutdown mid-operation at least once and watched what happened, instead
      of trusting the cancellation token is respected because you passed it in somewhere.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `feat(m5): in-memory background work queue and processor`.
- [ ] Keep the M3 "reproduce the captive dependency, then fix it" work as two visible steps in
      history (even a commit that's later reverted/amended is fine) rather than only committing
      the already-correct final version.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 15 containerizes
      an API that now includes long-running background services, not just request handlers.

## Reference Docs (use only when stuck)

Hosted services:
- [Background tasks with hosted services in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services)

Queued background work:
- [Queued background tasks](https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services#queued-background-tasks)
- [System.Threading.Channels](https://learn.microsoft.com/dotnet/core/extensions/channels)

Dependency injection scopes in background services:
- [Dependency injection scope validation](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines#scope-validation)

Host shutdown behavior:
- [.NET generic host shutdown](https://learn.microsoft.com/dotnet/core/extensions/generic-host#host-shutdown)
