# 0001: Choose a language and framework for the app

- **Date:** 2026-09-27
- **Status:** Accepted
- **Phase:** 0
- **Related:** None

## Context

The app is a small Spotify listening tracker. It needs:

- One program with several subcommands: `serve` (read-only HTTP API),
  `poll` (fetch once, insert, exit), `import` (load a Spotify data export),
  `authorize` (one-time OAuth), and `migrate` (apply schema migrations).
  Later phases package and deploy this program.
- A database that supports timestamp values.
- A small JSON API (top tracks by time range, `GET /healthz`) plus one plain
  HTML page.
- Tests that run against a real database, locally and in CI.

The app is intentionally small. The main goal of the repo is learning DevOps:
containers, CI, infrastructure as code, deployment, observability, and backups.
The language affects later work: how the app is packaged and deployed, how
large the deployed artifact is, how many dependencies need security updates,
and how long builds take.

The owner has some experience with Python and JavaScript and none with Go. The
owner must be able to explain every line in an interview, so readability and
good documentation matter more than brevity.

## Options considered

### Option 1: Python with FastAPI

- **How it works:** The app runs on the Python interpreter. FastAPI
  defines the HTTP routes and needs a separate server process (an ASGI server)
  to serve them. The standard library can parse subcommands. Database access
  uses a separate driver package.
- **Pros:** Readable code and a large tutorial base. Request validation comes
  from type hints. API docs are generated automatically at `/docs`. The owner
  already knows some Python, so Phase 1 would go fastest.
- **Cons:** The interpreter must ship with the app, so the deployed artifact
  is larger and has more components to keep patched. Dependency management
  adds tooling (virtual environment, pip or uv, a lockfile). FastAPI pulls in
  several packages and hides work behind decorators and dependency injection,
  which is more to explain.
- **When a team would choose it:** Data-heavy apps, fast iteration, or a team
  that already works in Python.

### Option 2: Node.js with Express

- **How it works:** The app runs on the Node.js runtime. Express defines
  the HTTP routes. Subcommands come from the process arguments. Database
  access uses a separate driver package.
- **Pros:** Same language as the HTML page's JavaScript. Express itself is
  small. npm lockfiles are simple.
- **Cons:** Async code and the event loop take more explanation. Input
  validation needs an extra library. Deep npm dependency trees mean the most
  packages to keep updated and patched of the three options. Sharing a language
  with one plain HTML page is a small benefit.
- **When a team would choose it:** A JavaScript-first team, or one that
  shares code between backend and frontend.

### Option 3: Go with the standard library

- **How it works:** The program compiles to one static binary. Subcommands
  and HTTP routing use the standard library. Since Go 1.22 the standard router
  matches methods and path patterns, so no web framework is needed. Database
  access uses a separate driver package. Testing is built into the toolchain.
- **Pros:** One binary with no separate runtime, so the deployed artifact is
  small, has few components to patch, and starts fast. Very few third-party
  dependencies. Code is explicit, with little hidden behavior. Docker,
  Kubernetes, and Terraform are written in Go, so the language is common in
  DevOps work.
- **Cons:** Steepest learning curve, since the owner hasn't used Go. More code
  for the same feature: explicit error checks after most calls and manual
  mapping of query results into structs. Fewer beginner tutorials for this
  exact stack. Learning the language will slow Phase 1.
- **When a team would choose it:** Infrastructure tools and small services
  where artifact size, startup time, and deployment simplicity matter.

## Decision

<!--
OWNER WRITES THIS SECTION.
"We chose X because..." in your own words, 2–4 sentences.
Name the one or two factors that decided it.
-->

I chose Go because I want to expand my horizons with the languages that I know. If I am learning a lot of new concepts with the development of this project, it would not hurt learning one more new thing that is also decently popular in the field that I am choosing.

## Consequences

<!-- OWNER WRITES THIS SECTION. -->

- **What gets easier:** Performance and less dependencies. 
- **What gets harder, or what we're accepting:** Building the application as well as getting the data structured and validated in the right way.
- **What would make us revisit this:** Time spent on building the application compared to learning and devloping proficiency on DevOps tooling.

## References

- [Go blog: Routing enhancements for Go 1.22](https://go.dev/blog/routing-enhancements)
- [FastAPI documentation](https://fastapi.tiangolo.com/)
- [Express documentation](https://expressjs.com/)
