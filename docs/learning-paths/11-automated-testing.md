# 11. Automated Testing — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 10](10-api-security-hardening.md). Build-it project — contracts, test requests, and
> checklists only. No C# solution code.

## Project Brief

For ten paths you've verified everything by hand: `.http` files, manually restarting the app,
manually checking response headers. That doesn't scale, and it's exactly why regressions creep in
quietly. This path builds a real automated test suite — fast, isolated **unit tests** for business
logic that shouldn't need a server or a database at all, and slower **integration tests** that
spin up the real HTTP pipeline against a real, disposable database. By the end, `dotnet test`
becomes your new first line of defense, and your `.http` files become a tool for quick manual
pokes instead of your only safety net.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 10](10-api-security-hardening.md)
      are fully done. This path doesn't add features — it writes tests against everything already
      built, so the more of it that already works correctly, the more useful this path is.

## Rules

- **No new business features.** Every test you write should describe behavior that already
  exists, not something new you're building.
- Every test must be **independent** — no test may depend on another test's leftover data or on
  a particular run order. If shuffling test order changes the outcome, that's a bug in the tests.
- Integration tests use a disposable test database created fresh for the test run — **never**
  your real development database.
- Reuse the Path 04 hand-written fake `IUnitOfWork` for at least one test before also trying a
  mocking library on the same scenario — keep the "try it by hand, then evaluate a library"
  pattern going one more time.

## Test Coverage Contract

The concrete deliverable for this path — behavior from earlier paths, and which kind of test now
proves it works:

| Behavior | From | Test type |
|---|---|---|
| Request validators reject missing/invalid fields, individually and combined | Path 08 | Unit |
| The `AuthorId`-exists validation rule, with its dependency faked/mocked | Path 03 / 08 | Unit |
| The custom author-deletion authorization handler, all four role/claim combinations | Path 09 | Unit |
| With-first-book atomicity (success and forced-failure), tested twice — once with the Path 04 hand-written fake, once with a mocking library | Path 04 | Unit |
| Full book CRUD through the real HTTP pipeline | Paths 01–05 | Integration |
| Combined pagination + filtering + sorting | Path 05 | Integration |
| Register → login → authenticated write succeeds → unauthenticated write fails (`401`) → wrong role fails (`403`) | Path 09 | Integration |
| The same book read through `v1` and `v2`, confirming the different wire shapes | Path 07 | Integration |
| A validation failure returns the exact `ProblemDetails` shape | Path 08 | Integration |
| An unhandled exception returns the safe `500` shape, never leaking internals | Path 08 | Integration |

## Unit vs. Integration: A Decision Framework

| Question | Lean toward |
|---|---|
| Does this only make sense with a real database/HTTP pipeline behind it? | Integration |
| Can you isolate the logic with a fake/mock and get an answer in milliseconds? | Unit |
| Are you testing "did the pieces wire together correctly end-to-end"? | Integration |
| Are you testing "is this one rule, calculation, or decision correct"? | Unit |

## Suggested Project Structure

- [ ] A dedicated test project (xUnit) for unit tests, referencing the API project.
- [ ] A dedicated (or clearly separated) set of integration tests using `WebApplicationFactory`,
      configured to use a disposable test database instead of your real one.
- [ ] A mocking library (Moq or NSubstitute — pick one) referenced by the unit test project.
- [ ] Test names/organization that make it obvious, at a glance, which row of the
      [Test Coverage Contract](#test-coverage-contract) each test proves.

## Test Naming Convention

Pick one naming convention and apply it to every test, so anyone (including future you) can tell
what's being proven without opening the test body:

- A consistent shape like `MethodOrEndpoint_Scenario_ExpectedResult` — a failing test called
  something like *"create book, missing title, returns validation error"* tells you what broke
  straight from your terminal's failure summary, before you open anything else.
- Group tests by the same areas as the [Test Coverage Contract](#test-coverage-contract) — one
  test class/fixture per row's general area, not one giant catch-all class holding everything.
- A failing test's **name alone** should tell you which row of the coverage contract broke.

## Test Data Setup Strategy

Decide, deliberately, how each integration test gets the data it needs — don't let this happen by
accident:

| Approach | Trade-off |
|---|---|
| Each test class seeds its own known data and cleans up (or uses a fresh database) afterward | Most isolated, but more setup code per test class. |
| One shared baseline seeded once per test run, with each test only asserting on data it created itself | Less duplicated setup, but a test that accidentally reads *someone else's* data can pass for the wrong reason. |
| A brand-new database per test class, not just per test run | Maximum isolation, slowest to run. |

Whichever you pick, the check from M5 (your real development database is never touched by any
test run) is the one non-negotiable part — everything else here is a trade-off between isolation
and speed.

## What This Path Deliberately Does NOT Cover

Being deliberate about scope is itself a real testing skill — not everything needs, or benefits
from, an automated test:

- **CORS and rate limiting (Path 10)** are timing- and configuration-sensitive in ways that make
  them awkward and often flaky to assert reliably in an automated suite. They stay verified
  manually, exactly as you already proved them in Path 10 — a deliberate choice, not a gap you
  forgot about.
- **Exhaustive combinatorial validation testing** (every possible combination of every field being
  valid or invalid at once) isn't the goal — the
  [Test Coverage Contract](#test-coverage-contract) asks for representative coverage of each rule,
  not a combinatorial explosion of near-duplicate test cases.
- **The OWASP audit table from Path 10** is a human judgment exercise, not something to convert
  into assertions — some of its rows (its "not applicable" justifications, especially) aren't the
  kind of thing an automated test meaningfully proves.
- **Manual `.http` files stay useful** for a quick, ad-hoc poke at the running API. Automating the
  important scenarios doesn't make hand-testing worthless — it just stops being your *only* safety
  net.

## Milestones

Work top to bottom. Each one should run and be manually verified (by actually running the tests)
before you start the next.

### M1 — Set Up the Test Project

- [ ] Create a new xUnit test project referencing your API project.
- [ ] Write one trivial test that always passes, and confirm you can run it via your test runner
      (`dotnet test` or your IDE's test explorer) before writing anything meaningful — prove the
      plumbing works first.

### M2 — Unit Test Your Validators

- [ ] Write unit tests for at least one FluentValidation validator (Path 08), covering: valid
      input passes with no errors, each individual rule violation fails with the message you
      expect, and multiple simultaneous violations all appear together.
- [ ] For the `AuthorId`-exists rule (which needs a dependency to check against), use your chosen
      mocking library to fake that dependency instead of hitting a real database — this is your
      first taste of mocking in this path.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| All fields valid | No validation errors. |
| Title missing, everything else valid | Exactly one error, naming `Title`. |
| Title missing **and** year out of range | Two errors, one per field. |
| `AuthorId` mocked as "doesn't exist" | Validation fails, naming `AuthorId` — without a real database anywhere in this test. |

### M3 — Unit Test the Custom Authorization Handler

- [ ] Write unit tests directly against the custom author-deletion authorization requirement and
      handler from Path 09, covering all four combinations:

| Caller | Expected result |
|---|---|
| `Admin` | Succeeds. |
| Plain `Contributor` (no `trusted` claim) | Fails. |
| `Contributor` with the `trusted` claim | Succeeds. |
| Anonymous / no recognized role at all | Fails. |

- [ ] Notice how fast and simple this is compared to re-testing the same logic by hand through
      real HTTP calls each time (Path 09's manual test script) — this is the concrete case for why
      unit tests exist, not just something to take on faith.

### M4 — Unit Test the Atomic With-First-Book Logic, Two Ways

- [ ] Reuse the Path 04 hand-written fake `IUnitOfWork` to write a real xUnit test (not just
      manual verification) for the with-first-book handler: the success case, and the
      forced-failure case that must leave no partial data behind.
- [ ] Reimplement the **same** test using your mocking library instead of the hand-written fake.
- [ ] Compare the two versions honestly: which was less code? Which reads more clearly to someone
      who didn't write it? Which would be easier to update if the interface changed? There's no
      required answer — just an honest comparison, the same evaluative habit as Paths 04/05/06/07.

### M5 — Set Up `WebApplicationFactory` for Integration Tests

- [ ] Add integration test infrastructure using `WebApplicationFactory`, configured to point at a
      fresh, disposable test database (a new SQLite file or equivalent) instead of your real
      development database.
- [ ] Get **one** trivial integration test passing first — e.g. `GET /api/v1/books` returns `200`
      — before building out the rest of the suite.

**Edge case to verify:** run the full test suite twice in a row and confirm your real development
database (if you still use one locally) is completely untouched by either run.

### M6 — Integration Test the Full Book CRUD Flow

- [ ] Automate the core Path 01–05 scenarios end-to-end through the real HTTP pipeline: create →
      get → list with a combined filter/sort/page query → update → delete, asserting real status
      codes and response shapes at each step.
- [ ] This is the automated replacement for the `.http` files you've been running by hand since
      Path 01 — once it's green, you can trust it more than your memory of "I tested this
      manually a few paths ago."

**Edge cases worth asserting automatically, not just once by hand:**

| Scenario | What the test proves |
|---|---|
| Create a book, then immediately read it back | The full round-trip through DTO mapping and EF Core actually persists what you think it does. |
| Update a book, then read it again | Changes are genuinely visible afterward, not just accepted with a `200`. |
| Delete a book, then attempt to read it again | It's actually gone, not just reported as deleted. |

### M7 — Integration Test the Auth Flow

- [ ] Automate: register a new user → login → an authenticated create succeeds → the same create
      **without** a token fails with `401` → a delete attempted with an insufficient role fails
      with `403` → the same delete with the correct role succeeds.
- [ ] Confirm test isolation here specifically: each test that needs a logged-in user registers
      (or otherwise sets up) its **own** user, rather than depending on a user created by another
      test.

**Edge cases worth asserting automatically:**

| Scenario | What the test proves |
|---|---|
| Two different tests each register their own uniquely-generated user | Tests don't collide with each other over unique email constraints. |
| A token obtained in one test isn't reused or valid in a different, unrelated test's assertions | Isolation genuinely holds for auth state, not just for book/author data. |

### M8 — Integration Test Versioning and Error Shapes

- [ ] Automate reading the **same** underlying book through both `v1` and `v2`, asserting the
      `genre` vs. `genres` shape difference from Path 07 in one test.
- [ ] Automate a validation failure and assert the **exact** `ProblemDetails` shape from Path 08,
      including the specific contents of the `errors` dictionary — not just "it returned 400."
- [ ] Automate triggering the safe `500` path (Path 08 M6) and assert no exception detail leaks
      into the response, the same way you verified it by hand there.

### M9 — Test Isolation and Speed

- [ ] Run the full suite twice in a row and confirm identical results both times — a test whose
      outcome depends on what ran before it is a bug in the test, not a quirk to work around.
- [ ] Time the unit tests as a group versus the integration tests as a group, and notice the
      difference — this is exactly why you don't want everything to be an integration test, and
      exactly why a pure unit test (M2/M3/M4) is worth having even when an integration test could
      theoretically cover similar ground.

**Edge cases worth checking specifically:**

| Scenario | Expected result |
|---|---|
| Run only the unit tests, in isolation from the integration tests | Completes in a small fraction of the time the full suite takes. |
| Run the full suite with tests executed in a different order (most runners support this) | Same pass/fail outcome as the default order. |
| Delete and recreate the test database between two full runs | No difference in outcome — nothing depends on leftover state from a previous run. |

### M10 — Full Regression, Now Automated

- [ ] Run the complete automated suite and confirm it independently reproduces the outcomes of
      your core manual test scripts from Paths 01–10 (not necessarily every single edge case, but
      every behavior listed in the [Test Coverage Contract](#test-coverage-contract)).
- [ ] From this point on, treat `dotnet test` passing as a required step before considering any
      future path's milestones done — your `.http` files remain useful for quick manual spot
      checks, but they're no longer your primary safety net.

## Manual Test Script

This path's "manual testing" is running the automated suite itself and checking its behavior, not
new `.http` requests.

```
dotnet test
```

Checklist to walk through after running it:

- [ ] Every test in the [Test Coverage Contract](#test-coverage-contract) exists and passes.
- [ ] The unit test group completes in a small fraction of a second per test — if any "unit" test
      is slow, it's probably secretly doing integration-test work (hitting a real database or
      network) and is miscategorized.
- [ ] Running the whole suite twice in a row produces identical pass/fail results both times.
- [ ] Deleting the test database file (if file-based) between runs doesn't break anything — the
      integration test setup should recreate whatever it needs.
- [ ] Deliberately break one thing on purpose (e.g., comment out the `AuthorId` existence check)
      and confirm the corresponding test **fails** — a test suite that can't fail when the code is
      actually wrong isn't proving anything.

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Add code coverage tooling and look at what's genuinely covered versus not — then decide
      deliberately whether any gap is worth closing, rather than chasing a coverage percentage for
      its own sake.
- [ ] Automate the Path 03 N+1 check (counting actual SQL queries) inside an integration test
      using an EF Core interceptor, instead of only checking it by hand via logs.
- [ ] Try a mutation-testing tool (e.g. Stryker.NET) to check whether your tests would actually
      catch a deliberately introduced bug, not just whether they currently pass.
- [ ] Add running the test suite as a required local git pre-commit step, foreshadowing the real
      CI pipeline in Path 15.
- [ ] Add a small set of tests for the Path 05 pagination edge cases specifically (page beyond the
      last page still returns `200` with an empty list; an invalid `page`/`pageSize` returns
      `400`) — these are easy to get subtly wrong again during some future refactor without anyone
      noticing by hand.
- [ ] Add a test that deliberately registers a user, then asserts the stored data never contains a
      plaintext password anywhere reachable through your own repository layer — turning one of
      Path 09's manual security checks into a permanent, automated guard.

## Definition of Done

- [ ] M1–M10 all checked off, in order, each verified by actually running the tests.
- [ ] Every row of the [Test Coverage Contract](#test-coverage-contract) has a real, passing test.
- [ ] The full suite is independent (order doesn't matter) and passes twice in a row identically.
- [ ] At least one scenario (M4) is tested both with the Path 04 hand-written fake and with a
      mocking library, with an honest comparison written down.
- [ ] No integration test ever touches your real development database.
- [ ] You deliberately broke something and watched the corresponding test fail, proving the suite
      actually detects regressions.

## Self-Review Checklist

- [ ] You can explain, for at least one test, specifically why it's a unit test and not an
      integration test (or vice versa) — using the
      [decision framework](#unit-vs-integration-a-decision-framework), not a guess.
- [ ] You ran the full suite at least twice in a row yourself and confirmed identical results.
- [ ] You can point to the exact test that would fail if someone reintroduced a hand-rolled `if`
      validation check that Path 08 was supposed to have fully replaced.
- [ ] Your M4 comparison between the hand-written fake and the mocking library is based on what
      you actually experienced writing both, not a general opinion.
- [ ] No test in your suite depends on secrets, tokens, or data left over from a previous test run.
- [ ] You know which of your tests are unit tests and which are integration tests just by their
      project/folder location, without needing to read each one to find out.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `test(m6): integration tests for full book CRUD flow`.
- [ ] Keep the M4 hand-written-fake test and the mocking-library test as separate commits, so the
      comparison is visible in history, not just in a paragraph you wrote once.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 12 adds logging and
      observability on top of an API you can now change with confidence.

## Reference Docs (use only when stuck)

Unit testing:
- [Unit testing C# in .NET with xUnit](https://learn.microsoft.com/dotnet/core/testing/unit-testing-with-dotnet-test)
- [Moq documentation](https://github.com/devlooped/moq)
- [NSubstitute documentation](https://nsubstitute.github.io/)

Integration testing:
- [Integration tests in ASP.NET Core](https://learn.microsoft.com/aspnet/core/test/integration-tests)

Testing authorization and validation in isolation:
- [Unit testing authorization handlers](https://learn.microsoft.com/aspnet/core/security/authorization/policies#unit-testing-code-that-contains-authorization)

Test data and mutation testing (for the stretch goals):
- [Stryker.NET mutation testing](https://stryker-mutator.io/docs/stryker-net/introduction/)
- [.NET test data builders and object mothers (general pattern discussion)](https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices)
