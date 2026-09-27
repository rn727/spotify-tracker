# 0002: Choose a database

- **Date:** 2026-09-27
- **Status:** Accepted
- **Phase:** 0
- **Related:** ADR-0001 (Go); affects ADR-0005, ADR-0007, ADR-0008, ADR-0009, ADR-0021, ADR-0022

## Context

The database is the only durable state in the system. Plays that fall out of
Spotify's 50-play recently-played window exist nowhere else, so losing the
database means losing history that only a new data export can partly restore.

What the app needs from it:

- **Small data, low traffic.** One user. The poller writes at most 50 rows
  per run, every ~30 minutes. A multi-year export backfill is likely tens to
  hundreds of thousands of rows. The API serves one person.
- **Timestamps in UTC.** Every play has a timestamp, and top-tracks queries
  filter by time range (ADR-0006).
- **Duplicate prevention.** Ingestion must be idempotent. ADR-0005 chooses
  between checking in code and a unique constraint with `ON CONFLICT`; the
  database must support whichever wins.
- **Grouped queries.** "Count plays per track within a range, order by count,
  limit N."

What later phases need from it:

- **Phase 2 (Containers):** Compose runs the stack with persistent storage and
  a healthcheck. `GET /healthz` returns 200 only when the app reaches the
  database.
- **Phase 4 (CI):** Tests run against a real database on pull requests.
- **Phase 8 (Observability):** Something to monitor.
- **Phase 9 (Backups):** Nightly dump to offsite storage and a timed restore
  drill.
- **Phase 10 (Kubernetes):** ADR-0022 decides where the database lives in a
  cluster.

Constraints:

- Cost should be zero or close to it.
- The repo exists to learn DevOps. An option that removes operational work
  also removes the chance to learn it. An option that adds operational work
  must stay explainable.
- The owner must be able to explain the setup in an interview.
- The app is written in Go (ADR-0001). Every option needs a Go database driver,
  which is a separate choice made in Phase 1.

## Options considered

### Option 1: Self-hosted Postgres

- **How it works:** Postgres runs as its own server process, in a container
  next to the app (Compose in Phase 2, the server in Phase 3). The app
  connects over the network with a connection string in `DATABASE_URL`. Data
  lives in a Docker volume.
- **Pros:**
  - Industry standard. Most job postings and tutorials that mention a
    relational database mean Postgres or something like it.
  - Has a real timezone-aware timestamp type (`timestamptz`) and full
    date/time functions, which fit the UTC rule and ADR-0006.
  - Supports unique constraints and `INSERT ... ON CONFLICT`.
  - Exercises every DevOps skill the roadmap targets: a second container with
    a healthcheck and volume, a database service in CI, secrets for the
    password, a real dump-and-restore for Phase 9, and a stateful workload
    for Phase 10.
  - Official container images and first-class support as a GitHub Actions
    service container.
- **Cons:**
  - More to run: one more container, a password to manage, memory use on a
    small free-tier server, version upgrades.
  - You own backups, restore, and upgrades. If the volume is lost and there's
    no backup, data is gone.
  - Local development and tests need Docker (or a local install) running.
- **When a team would choose it:** Almost any server-side app that expects to
  grow, needs concurrent writers, or wants full control over its data and
  operations.

### Option 2: Supabase (hosted Postgres)

- **How it works:** Supabase runs a Postgres database for you on its
  platform. The app connects to it over the internet with a connection
  string. Supabase also offers extra features (auto-generated REST API, auth,
  storage) that this app would not use.
- **Pros:**
  - Same Postgres SQL, timestamp types, and `ON CONFLICT` as Option 1.
  - No database server to run, patch, or size. Nothing extra on the VM.
  - Web dashboard with a table viewer and SQL editor.
  - Moving to self-hosted Postgres later is a dump and restore, since it is
    Postgres underneath.
- **Cons:**
  - **Free plan limits:** 500 MB database, 2 active projects, **no automatic
    backups**, and projects **pause after 1 week of inactivity**. The poller
    writing every 30 minutes should count as activity, but if the poller
    breaks for a week, the database pauses too.
  - **Networking:** Direct connections use IPv6 unless you pay for the IPv4
    add-on. From an IPv4-only network (common on cloud VMs and some home
    ISPs), you use Supabase's connection pooler instead, which is one more
    thing to understand.
  - Adds an external service and a network hop between app and data. The app
    works only when both the server and Supabase are up.
  - Removes much of the operational learning: no database container, and
    backups are yours to build anyway on the free plan.
  - CI still needs a local Postgres for tests, so you end up running Postgres
    in containers regardless.
  - Your listening data lives with a third party.
- **When a team would choose it:** Small teams that want Postgres without
  running it, especially apps that also use Supabase's auth or auto-generated
  APIs.

### Option 3: SQLite

- **How it works:** The database is a single file on disk. There is no server
  process; the app opens the file through a driver library. In Compose, the
  file sits on a volume shared by the containers that need it.
- **Pros:**
  - Simplest to run: no second container, no password, no network. Tests can
    use a fresh temporary file per run, which still counts as a real database.
  - Plenty for this workload. SQLite's own docs recommend it for single-server,
    low-to-medium traffic sites.
  - Supports unique constraints and `ON CONFLICT` upserts.
  - Backup is copying one file safely (SQLite has a built-in online backup).
- **Cons:**
  - **No native date/time type.** Timestamps are stored as ISO 8601 text or
    Unix integers, and the app must use one format consistently. This is
    manageable but is something to get right, since ADR-0006 and range queries
    depend on it.
  - **One writer at a time.** Fine for one poller, but `poll` and `import`
    running at once will queue, and a long import can briefly block polls.
  - **Must be on local disk.** SQLite's docs warn against network
    filesystems. This limits where the file can live on the server and in
    Kubernetes (ADR-0022).
  - **Go driver trade-off.** Go SQLite drivers either call the C library
    (needs a C compiler at build time, which complicates a static binary and
    the base-image choice in ADR-0008) or use a pure-Go translation of it
    (no C compiler, somewhat slower). The driver is picked in Phase 1.
  - Less DevOps practice: no database service in Compose or CI, no connection
    healthcheck worth the name, a simpler backup story.
  - Less common in server job postings.
- **When a team would choose it:** Single-server apps, embedded or desktop
  apps, prototypes, and small personal tools where simplicity matters more
  than concurrent writes.

## Decision

<!--
OWNER WRITES THIS SECTION.
"We chose X because..." in your own words, 2–4 sentences.
Name the one or two factors that decided it.
-->

I chose a self-hosted Postgres server because it is industry standard tooling that a lot of companies look for. Additionally, it works well and is documented with Github Actions and Docker, which I currently plan to use towards the later phases of this project.

## Consequences

<!-- OWNER WRITES THIS SECTION. -->

- **What gets easier:** Industry standard tooling is immediately learned; no need to try and pivot from an easier solution not intended for enterprise applications.
- **What gets harder, or what we're accepting:** I have to manage my own backups, restores, and updates. It will be harder to upkeep than other simpler/more abstracted solutions and take more time and focus to implement correctly.
- **What would make us revisit this:** Timing if phase 1 drags on super long, especially when I am already trying to learn Go at the same time.

## References

- [PostgreSQL documentation: Date/Time types](https://www.postgresql.org/docs/current/datatype-datetime.html)
- [PostgreSQL documentation: INSERT ... ON CONFLICT](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)
- [PostgreSQL documentation: Backup and restore](https://www.postgresql.org/docs/current/backup.html)
- [Postgres official Docker image](https://hub.docker.com/_/postgres)
- [GitHub Docs: Creating PostgreSQL service containers](https://docs.github.com/en/actions/use-cases-and-examples/using-containerized-services/creating-postgresql-service-containers)
- [Supabase pricing](https://supabase.com/pricing)
- [Supabase Docs: Project pausing](https://supabase.com/docs/guides/platform/free-project-pausing)
- [Supabase Docs: Connecting to Postgres (IPv4/IPv6, pooler)](https://supabase.com/docs/guides/database/connecting-to-postgres)
- [SQLite: Appropriate uses for SQLite](https://www.sqlite.org/whentouse.html)
- [SQLite: Datatypes (dates and times)](https://www.sqlite.org/datatype3.html)
- [SQLite: UPSERT](https://www.sqlite.org/lang_upsert.html)
- [SQLite: Online backup API](https://www.sqlite.org/backup.html)
