# Database Containers in Practice

This chapter takes the volume and persistence concepts from Chapters 1–3 and applies them specifically to running a real database — Postgres — as a container, in a way that's realistic for actual local development and testing, not just a toy demo. We cover initialization behavior, credential handling, health checks, and the specific mistakes that cause "my database container looked like it started fine but the app couldn't connect" — a genuinely common early confusion.

---

## First-Run Initialization vs. Subsequent Starts

The official Postgres image has a specific, important behavior worth understanding precisely: its entrypoint script only runs first-time initialization logic (creating the specified database, running any init scripts) **if its data directory is empty** at container start.

```bash
docker volume create pg-data

docker run -d --name postgres \
  -v pg-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=orders \
  postgres:16

docker logs postgres
# PostgreSQL init process complete; ready for start up.
# ... database system is ready to accept connections
```

```bash
# Stop and restart — because pg-data ALREADY has data in it now,
# the init logic is correctly skipped this time:
docker restart postgres
docker logs postgres | tail -5
# database system was interrupted; last known up at ...
# database system is ready to accept connections
# (No "init process" messages — it recognized existing data and
#  skipped straight to normal startup.)
```

This "only initializes on an empty data directory" behavior is precisely why **changing `POSTGRES_PASSWORD` or `POSTGRES_DB` after the first run has no effect** — a genuinely common point of confusion. Those environment variables are only read during first-time initialization; once the volume has data, the database's actual credentials live in that data, not in whatever environment variables you happen to pass on a later `docker run`.

```bash
# This does NOT change the running database's password —
# the volume already has data, so init logic (which reads
# POSTGRES_PASSWORD) never runs again:
docker rm -f postgres
docker run -d --name postgres \
  -v pg-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=a-different-password \
  postgres:16
# The database still only accepts the ORIGINAL password —
# "a-different-password" was silently ignored.
```

To actually change credentials, you need either a real `ALTER USER` command run against the live database, or (for throwaway dev environments) deleting the volume entirely and letting initialization run fresh — never assume changing an env var retroactively reconfigures an already-initialized database.

---

## Init Scripts: Seeding Data on First Run

The official image runs any `.sql` or `.sh` files placed in `/docker-entrypoint-initdb.d/` — but again, **only on first initialization**, exactly like the credential behavior above.

```bash
mkdir -p ./init-scripts
cat > ./init-scripts/01-schema.sql << 'SQL'
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    sku VARCHAR(64) NOT NULL,
    quantity INT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
SQL

docker run -d --name postgres \
  -v pg-data:/var/lib/postgresql/data \
  -v "$(pwd)/init-scripts:/docker-entrypoint-initdb.d:ro" \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=orders \
  postgres:16

docker exec postgres psql -U postgres -d orders -c "\dt"
#           List of relations
# Schema |  Name  | Type  |  Owner
# --------+--------+-------+----------
# public | orders | table | postgres
```

Note the bind mount here (`:ro` — the init scripts directory has no reason to be writable) is exactly Chapter 2's local-development bind-mount pattern, applied to database seeding specifically — a genuinely useful pattern for spinning up a schema-ready database for local development or integration tests with zero manual setup steps.

---

## Health Checks: Why "Container Is Running" Doesn't Mean "Database Is Ready"

A Postgres container can report `Up` in `docker ps` well before the database is actually ready to accept connections — the process has started, but initialization or crash recovery may still be in progress. Applications that connect immediately on container start (a common mistake in local dev scripts and, more consequentially, in Compose's naive `depends_on` without a condition — covered fully in Phase 6) can fail intermittently with connection-refused errors that look like flaky infrastructure but are actually a simple ordering/timing problem.

```bash
docker run -d --name postgres \
  --health-cmd="pg_isready -U postgres" \
  --health-interval=5s \
  --health-timeout=3s \
  --health-retries=5 \
  -v pg-data:/var/lib/postgresql/data \
  -e POSTGRES_PASSWORD=secret \
  postgres:16

docker inspect postgres --format '{{.State.Health.Status}}'
# starting
# ... (a few seconds later) ...
docker inspect postgres --format '{{.State.Health.Status}}'
# healthy
```

`pg_isready` is Postgres's own bundled tool, purpose-built to answer exactly this question correctly (as opposed to, say, just checking whether the process exists, or attempting a raw TCP connect, either of which can give a false-positive "ready" before the database is actually accepting real connections). We build the same idea — real, meaningful health checks, not just "is the process running" — as a full topic in Phase 7, and Compose's `depends_on` with a `condition: service_healthy` clause (Phase 6) depends directly on this exact health check mechanism.

---

## Connecting From a Real Spring Boot Application

```properties
# application.properties — matches the postgres container's
# network-reachable name (Phase 4's DNS mechanism) and the
# credentials/database established during initialization above
spring.datasource.url=jdbc:postgresql://postgres:5432/orders
spring.datasource.username=postgres
spring.datasource.password=secret
spring.jpa.hibernate.ddl-auto=validate
```

`spring.jpa.hibernate.ddl-auto=validate` (rather than `update` or `create`) is a deliberate, production-realistic choice here: it makes Hibernate *verify* the schema matches your entity mappings on startup rather than silently mutating the schema — schema changes should come from explicit, reviewed migrations (Flyway or Liquibase, both of which work naturally against a containerized Postgres exactly the same way they would against any other Postgres instance) rather than an ORM's auto-DDL feature, which is convenient for quick local prototyping but risky for anything you intend to keep production-realistic.

---

## Common Misconceptions This Chapter Should Correct

- **"Changing `POSTGRES_PASSWORD` on a subsequent `docker run` updates the running database's credentials."** It doesn't, if the volume already has initialized data — init-time environment variables are only read during first-time initialization.
- **"A container showing `Up` in `docker ps` means its service is ready to accept requests."** It means the *process* started — readiness is a separate, meaningful concept, which is exactly why Docker health checks (and Kubernetes readiness probes, Phase 10) exist as a distinct signal from "process is running."
- **"Init scripts in `/docker-entrypoint-initdb.d/` run on every container start."** They run only once, during first-time initialization on an empty data directory — exactly like the credential behavior above, and for the same underlying reason.
- **"`ddl-auto=update` is a fine choice for anything beyond a quick local prototype."** It works, but it means your schema's source of truth is scattered across whatever your entity classes currently look like, with no reviewable migration history — a genuine risk once more than one person or environment depends on the same schema.

---

## What's Next

Time to put every piece from this phase together in one hands-on project: a real Spring Boot service backed by a real Postgres container, using a named volume for genuine persistence, with a health check gating readiness, verified end-to-end by killing and recreating the application container while confirming the data survives untouched.

**Next:** [`05-spring-boot-with-persistent-postgres-storage.md`](./05-spring-boot-with-persistent-postgres-storage.md)