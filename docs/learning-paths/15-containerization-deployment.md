# 15. Containerization & Deployment — Book Catalog API

> Part of the [.NET API Development Roadmap](../dotnet-api-roadmap.md). Continues the Book Catalog
> API from [Path 01](01-aspnet-core-fundamentals.md) through
> [Path 14](14-background-processing.md). Build-it project — contracts, test requests, and
> checklists only. No C# solution code (and no literal Dockerfile/compose syntax either — the
> structure is described, you write the actual files).

## Project Brief

An API that only runs correctly on your machine, started the way you happen to start it, isn't
actually done. This path packages the Book Catalog API into a container, brings its dependencies
(Redis from Path 13, and your database) up alongside it with one command, and proves — not
assumes — that the same image behaves correctly purely through configuration, never a rebuild.
Along the way, Path 12's health checks and Path 14's graceful shutdown finally get a real consumer
(the container runtime itself, not just you curling an endpoint by hand), and a minimal CI
pipeline makes sure Path 11's test suite actually runs on every change, not just when you remember
to run it yourself.

## Prerequisites

- [ ] [Path 01](01-aspnet-core-fundamentals.md) through [Path 14](14-background-processing.md)
      are fully done. This path leans on nearly all of them: Path 02's configuration, Path 03's
      database choice, Path 09/10's secrets discipline, Path 11's test suite, Path 12's health
      endpoints, Path 13's Redis dependency, and Path 14's graceful shutdown.
- [ ] Docker is installed and working locally.

## Rules

- **No new business features.** This path packages and deploys what already exists; it doesn't
  change what the API does.
- **No secrets baked into the image, ever.** Verified in M4, not assumed.
- **The same image must behave correctly across environments purely through injected
  configuration** (environment variables / mounted config at container start) — never by
  rebuilding it per environment.
- Reuse Path 12's health endpoints and Path 14's graceful shutdown handling as-is. This path wires
  them into Docker; it doesn't reinvent either concept for containers specifically.
- If you used SQLite for Path 03, keep using it here — persistence becomes a volume question, not
  a "run a database server" question. If you used SQL Server LocalDB, decide explicitly for this
  path: switch to a container-friendly engine (e.g. SQL Server for Linux containers, or
  PostgreSQL), or use SQLite specifically for the containerized version. Either is fine — write
  down which, and why, the same way you've documented every other cross-path decision so far.

## Container Environment Configuration Contract

| Config value | Where it comes from in the container | Never comes from |
|---|---|---|
| Database connection string | Environment variable / mounted secret at container start | Baked into the image |
| JWT signing key (Path 09) | Environment variable / mounted secret at container start | Baked into the image |
| `ASPNETCORE_ENVIRONMENT` | Set explicitly for the deployment (typically `Production` for the containerized stack, even in local testing, unless you're deliberately exercising `Development` behavior) | Left to whatever the default happens to be |
| Redis connection (Path 13) | Environment variable pointing at the compose **service name** (e.g. `redis`) | A hardcoded `localhost` |

## Worked Example

### The shape of a multi-stage build (structure, not literal syntax — you write the real file)

```
Stage 1 ("build"): start from the full .NET SDK base image
  -> restore NuGet packages as their own step, before copying the rest of the source
     (so a source-code-only change doesn't force a slow re-restore every time)
  -> copy the remaining source
  -> publish a Release build to an output folder

Stage 2 ("final"): start from the much smaller ASP.NET Core *runtime* base image
  -> copy ONLY the published output from Stage 1 — the SDK, compilers, and NuGet caches
     never make it into this final image
  -> create and switch to a non-root user before the app runs
  -> declare the port the app listens on
  -> set the entrypoint to run the published app
```

Contrast this with a single-stage build that starts from the SDK image and never switches to a
runtime-only base — it works, but ships every compiler and build tool in your production image for
no reason, and stays root by default unless you explicitly change it.

### Compose topology (illustrative — you write the actual file)

| Service | Represents | Depends on |
|---|---|---|
| `api` | The Book Catalog API, built from your multi-stage Dockerfile | `redis`, your database |
| `redis` | Distributed cache (Path 13) | — |
| `database` (or a volume-backed SQLite file, per your M-prerequisite decision) | Persistent storage | — |

The `api` service reaches the other two by their **service names** (`redis`, `database`), never
by `localhost` — inside Docker's networking, `localhost` refers to the `api` container itself, not
its neighbors.

### What actually changes between local and containerized

| Aspect | Running locally (`dotnet run`, Paths 01–14) | Running containerized (this path) |
|---|---|---|
| Configuration source | `appsettings.json` + environment overrides | Environment variables / mounted config at container start — no rebuild per environment |
| Reaching Redis/the database | `localhost` | The other container's **service name** |
| Process lifecycle | You start/stop it directly | Docker sends a stop signal; your Path 14 shutdown handling must respond to it |
| Health checks | Something you `curl` by hand | Something Docker itself polls automatically |
| What could go wrong that you'd never see locally | Nothing containerization-specific | Secrets baked into a shippable image, `localhost` assumptions, missing volumes |

### A container health check, reusing what Path 12 already built

| Check | Points at |
|---|---|
| Container-level health check | Your existing `/health/live` and/or `/health/ready` endpoints — no new health logic, just a new consumer of what already exists. |

## Common Containerization Pitfalls

- Baking a connection string, signing key, or any other secret directly into the image — anyone
  who can pull the image can read it back out.
- A single-stage build that ships the full SDK (and every build tool) in your final image,
  inflating its size and attack surface for no benefit at runtime.
- Running as root inside the container by default, when a non-root user costs almost nothing to
  set up.
- Using `localhost` from inside one container to reach another — the classic first mistake with
  container networking; it needs the other container's **service name**.
- No `.dockerignore`, so your build context (and sometimes your image) ends up including `bin`,
  `obj`, `.git`, and anything else you never intended to ship.
- No volume for data that actually needs to survive a restart, silently losing everything on every
  `docker compose down`.
- A `HEALTHCHECK` that always reports healthy because it checks something trivial (e.g. "is the
  process running") instead of something meaningful (Path 12's actual readiness check) — the same
  liveness-vs-readiness distinction matters just as much to Docker as it does to a human.
- A CI pipeline that builds the image but never actually runs the test suite, giving false
  confidence that "the build passing" means "the code works."

## Suggested Project Structure

- [ ] A multi-stage `Dockerfile` for the API, following the
      [worked example](#the-shape-of-a-multi-stage-build-structure-not-literal-syntax--you-write-the-real-file)
      shape.
- [ ] A `.dockerignore` excluding `bin/`, `obj/`, `.git/`, and anything else that shouldn't be part
      of the build context.
- [ ] A `docker-compose.yml` (or equivalent) bringing up the `api`, `redis`, and database services
      together, per the [compose topology](#compose-topology-illustrative--you-write-the-actual-file).
- [ ] A named volume for whatever needs to persist across restarts.
- [ ] A minimal CI workflow file (e.g. under `.github/workflows/`) that builds the image and runs
      the Path 11 test suite.

## Milestones

Work top to bottom. Each one should run and be manually testable before you start the next.

### M1 — Write a Multi-Stage Dockerfile

- [ ] Write a multi-stage `Dockerfile` per the [worked example](#worked-example): an SDK-based
      build stage, and a slim runtime-based final stage that copies only the published output.
- [ ] Add a `.dockerignore` excluding `bin/`, `obj/`, `.git/`, and anything else irrelevant to the
      build.
- [ ] Build the image locally, and — just once, for comparison — build a naive single-stage
      version too. Compare the two images' sizes yourself and confirm the difference is real.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Multi-stage image vs. single-stage image, same source | Multi-stage is meaningfully smaller. |
| A source-only change, rebuilt | The NuGet restore layer is reused from cache if you ordered your build steps correctly, rather than re-running on every single change. |
| Running the final image without ever having the SDK installed on the host | Still works — the runtime image is genuinely self-contained. |

### M2 — Run as a Non-Root User

- [ ] Configure the final image to run as a non-root user rather than defaulting to root.
- [ ] Verify it yourself: confirm the process actually running inside the container is not root
      (there's a simple way to check this from outside the container, or by opening a shell into
      it).

### M3 — Get Configuration Right: Build Once, Configure Per Environment

- [ ] Confirm nothing environment-specific (connection strings, the JWT signing key, feature
      toggles) is baked into the image — everything must come from environment variables or
      mounted configuration supplied at container start.
- [ ] Prove it: run the **same** image twice with different environment variable values for at
      least one setting (e.g. the database connection string) and confirm it behaves differently
      each time, with no rebuild in between.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Same image, two different connection strings, two separate runs | Each run connects to its own configured database — proving configuration isn't baked in. |
| Missing a required configuration value entirely | Fails fast with a clear error at startup, not a confusing runtime failure later (same discipline as Path 02). |

### M4 — Audit: No Secrets in the Image

- [ ] Actually inspect your built image's layers/history for anything that shouldn't be there — a
      connection string, a signing key, a password — the same kind of concrete audit as Path 10's
      secrets sweep, now applied to a container image instead of source control.
- [ ] Fix anything you find, rebuild, and re-check.

### M5 — Docker Compose: The Whole Stack, One Command

- [ ] Write a compose file bringing up the API container plus Redis and your database, per the
      [compose topology](#compose-topology-illustrative--you-write-the-actual-file).
- [ ] Get the **entire stack** running with a single command.
- [ ] Confirm the API actually reaches Redis and the database using their compose **service
      names**, not `localhost` — if you used `localhost` anywhere, fix it now.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Fresh `docker compose up` from a clean state | All services start, and the API can serve a request that touches both Redis and the database. |
| Stop just the database service, leave the rest running | The API's `/health/ready` (Path 12) reflects the outage; `/health/live` still says it's alive. |

### M6 — Persist What Needs to Persist

- [ ] Confirm your database's actual data survives a full `docker compose down` followed by
      `docker compose up` (via a named volume), while data that's fine to lose (e.g. in-memory
      cache contents) doesn't need this treatment.
- [ ] Verify deliberately: create a book through the containerized API, tear the whole stack down,
      bring it back up, and confirm the book is still there.

### M7 — Wire Up Container Health Checks

- [ ] Add a container-level health check pointing at your Path 12 `/health/live` and/or
      `/health/ready` endpoints.
- [ ] Deliberately break the database connection while the stack is running (stop just that
      service, or point at a bad connection string) and confirm Docker itself now reports the
      `api` container as unhealthy — the same check you built for a human to read is now being
      read by your container tooling.

### M8 — Prove Graceful Shutdown Actually Works in a Container

- [ ] Stop the running `api` container (not a local Ctrl+C) and confirm your Path 14 hosted
      services still shut down cleanly, logging their "stopping" messages, within Docker's
      shutdown grace period.
- [ ] Understand and write down what happens if a shutdown doesn't finish before Docker's timeout
      (a forceful kill), and why that would matter for anything genuinely mid-operation.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| Stop the container while nothing is mid-operation | Clean shutdown, "stopping" log lines present. |
| Stop the container while the periodic job (Path 14) happens to be mid-run | Still shuts down cleanly within the grace period, or you've documented why it can't and what that risks. |

### M9 — A Minimal CI Pipeline

- [ ] Add a CI workflow that, on every push, builds the Docker image **and** runs the full
      Path 11 automated test suite — failing the build if either step fails.
- [ ] Confirm it actually catches something real: deliberately push a change that breaks a test,
      confirm CI reports it red, then fix it and confirm CI goes green again.

**Edge cases to verify:**

| Scenario | Expected result |
|---|---|
| A normal, passing change | CI goes green, image builds successfully. |
| A deliberately broken test | CI goes red, and it's clear from the output which test failed. |
| A change that breaks the Docker build itself (not a test) | CI still catches it, separately from test failures. |

### M10 — Full Regression + Container-Specific Verification

- [ ] Full functional regression running entirely inside the containerized stack — not your local
      `dotnet run`.
- [ ] New container-specific checks: the image contains no secrets (M4), it runs as non-root
      (M2), the same image behaves correctly across environments via configuration alone (M3),
      health checks work end-to-end (M7), data persists correctly (M6), graceful shutdown holds up
      (M8), and CI genuinely catches a deliberately broken test (M9).

## Manual Test Script

This path is mostly proven through terminal commands and container behavior, not just `.http`
requests — treat the steps below as one connected sequence.

```http
@baseUrl = http://localhost:8080

### 1. Regression check against the containerized API - core read
GET {{baseUrl}}/api/v1/books

### 2. Regression check - authenticated write still works
POST {{baseUrl}}/api/v1/books
Authorization: Bearer {{contributorToken}}
Content-Type: application/json

{ "title": "Containerized Test Book", "publishedYear": 2020, "genre": "Mystery", "authorId": 1 }

### 3. Health checks, now also consumed by Docker itself
GET {{baseUrl}}/health/live
GET {{baseUrl}}/health/ready
```

Manual (terminal/CLI) steps:

1. Build the multi-stage image and, once, a naive single-stage version — compare their sizes.
2. Run the image twice with two different environment variable configurations and confirm
   different behavior with no rebuild (M3).
3. Inspect the built image for secrets (M4) and fix anything found.
4. Bring up the whole stack with one compose command (M5), confirm service-name networking works,
   not `localhost`.
5. Create a book, run `docker compose down` then `docker compose up`, and confirm the book
   survived (M6).
6. Break the database connection deliberately and confirm Docker reports the container unhealthy
   (M7); restore it and confirm recovery.
7. Stop the running container and confirm graceful shutdown log lines appear within the grace
   period (M8).
8. Push a deliberately broken test to your CI pipeline, confirm it goes red, fix it, confirm it
   goes green (M9).

## Stretch Goals

Only after every milestone above is checked off:

- [ ] Push the built image to a container registry (e.g. GitHub Container Registry), then pull it
      down somewhere else entirely (a different machine, or after deleting your local images) to
      prove it's genuinely portable, not just "works on my machine."
- [ ] Add a lightweight reverse proxy in front of the API in compose, and move HTTPS termination
      there instead of the API handling it directly — a common real production topology.
- [ ] Extend the CI pipeline to also build and publish the image automatically on a successful
      run, stopping short of an actual deployment step.
- [ ] Try running the whole stack on a completely different machine (or a fresh VM/codespace)
      using only what's committed to the repo, to prove there's no hidden local-machine
      dependency you forgot about.
- [ ] Add a resource limit (CPU/memory) to the `api` service in compose, and deliberately set it
      too low once just to see how your app behaves when constrained, before setting it back to
      something reasonable.

## Definition of Done

- [ ] M1–M10 all checked off, in order, each with manual verification evidence.
- [ ] The final image contains no secrets, verified by direct inspection, not assumption.
- [ ] The container runs as a non-root user, verified directly.
- [ ] The same image behaves correctly across at least two different configurations without a
      rebuild.
- [ ] The whole stack (API, Redis, database) starts with one command and the services reach each
      other by name, not `localhost`.
- [ ] Data that should persist across a full stack restart does; data that shouldn't need to,
      doesn't need special handling.
- [ ] Docker's own health check correctly reflects a real database outage, using Path 12's
      existing endpoints.
- [ ] Graceful shutdown (Path 14) is proven to hold up under an actual container stop, not just a
      local Ctrl+C.
- [ ] CI builds the image, runs the full Path 11 test suite, and has been proven to catch a real,
      deliberately introduced failure.

## Self-Review Checklist

- [ ] You personally inspected your built image for secrets — not assumed your `.dockerignore` and
      environment-variable discipline were enough without checking.
- [ ] You ran the same image with two different configurations yourself and watched the behavior
      differ, rather than trusting the design on paper.
- [ ] You actually tore down and restarted the whole stack and confirmed which data survived and
      which didn't — not assumed your volume configuration was correct.
- [ ] You broke something on purpose (the database connection, a test) specifically to watch your
      health checks and CI pipeline catch it — not just seen them pass on the happy path.
- [ ] You can explain, specifically, why a multi-stage build is smaller and safer than a
      single-stage one, using your own two built images as evidence.

## Suggested Git Workflow

- [ ] Commit after each milestone, same convention as before, e.g.
      `feat(m5): docker-compose stack for api, redis, and database`.
- [ ] Keep the Dockerfile/compose commits separate from the CI workflow commit — they're related
      but distinct pieces of infrastructure.
- [ ] Note the commit where this path's Definition of Done is satisfied — Path 16 revisits the
      overall architecture of a project that now genuinely runs outside your machine.

## Reference Docs (use only when stuck)

Docker and .NET:
- [Containerize a .NET app with Docker](https://learn.microsoft.com/dotnet/core/docker/build-container)
- [.NET container images and best practices](https://learn.microsoft.com/dotnet/core/docker/introduction)

Docker Compose:
- [Docker Compose overview](https://docs.docker.com/compose/)

Health checks and shutdown in containers:
- [Health checks in ASP.NET Core](https://learn.microsoft.com/aspnet/core/host-and-deploy/health-checks)
- [.NET generic host shutdown](https://learn.microsoft.com/dotnet/core/extensions/generic-host#host-shutdown)

CI:
- [GitHub Actions for .NET](https://learn.microsoft.com/dotnet/devops/dotnet-secure-github-action)
