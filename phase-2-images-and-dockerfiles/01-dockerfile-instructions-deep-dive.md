# Dockerfile Instructions: A Deep Dive

You already know Dockerfile syntax at a surface level — `FROM`, `RUN`, `COPY`, `CMD`. This chapter goes one level deeper: what each instruction actually produces in terms of image layers (Phase 1, Chapter 4), which instructions create layers and which don't, and the specific semantic differences (`CMD` vs `ENTRYPOINT`, `COPY` vs `ADD`, `ENV` vs `ARG`) that cause real production bugs when confused. The goal isn't memorizing the instruction list — it's understanding precisely what happens to the image at build time for each one, so nothing in a Dockerfile is "magic" anymore.

---

## Every Instruction, Categorized by What It Actually Does

Not every instruction creates a new filesystem layer. This distinction matters for reasoning about build cache invalidation and final image size.

| Instruction | Creates a new layer? | What it actually does |
|---|---|---|
| `FROM` | Sets the base layer(s) | Establishes the starting point for the union filesystem stack |
| `RUN` | Yes | Executes a command in a temporary container, commits the resulting filesystem diff as a layer |
| `COPY` | Yes | Copies files from build context into the image filesystem, as a layer |
| `ADD` | Yes | Like `COPY`, plus URL fetching and archive auto-extraction (mostly avoid — see below) |
| `WORKDIR` | Yes (metadata + directory creation) | Sets the working directory for subsequent instructions; creates the dir if absent |
| `ENV` | No (metadata only) | Sets an environment variable, persists into the final image and any derived containers |
| `ARG` | No (build-time only, no image layer) | Defines a build-time variable, not present in the final image or running container |
| `EXPOSE` | No (metadata only) | Documents which port(s) the container listens on — does **not** actually publish anything |
| `CMD` | No (metadata only) | Default command/args if none given at `docker run` |
| `ENTRYPOINT` | No (metadata only) | The fixed executable that becomes PID 1 |
| `USER` | No (metadata only) | Sets the user subsequent instructions and the final container run as |
| `LABEL` | No (metadata only) | Arbitrary key-value metadata attached to the image |
| `VOLUME` | No (metadata only) | Declares a mount point as intended for external data (Phase 5) |
| `HEALTHCHECK` | No (metadata only) | Defines a command Docker runs periodically to assess container health |

The practical implication: **every `RUN` and `COPY` is a cache-invalidation boundary and a size contributor.** Metadata-only instructions (`ENV`, `CMD`, `LABEL`, etc.) don't add filesystem bytes but do still invalidate downstream cache if changed — we'll cover cache mechanics precisely in the next chapter.

---

## `RUN`: Where Your Build Actually Executes

Each `RUN` instruction starts a temporary container from the current image state, executes the given command inside it, and then commits the resulting filesystem changes as a new read-only layer — mechanically identical to what `docker commit` does to a running container (Phase 1, Chapter 3), just automated as part of the build.

```dockerfile
RUN apt-get update && apt-get install -y curl
```

A critical, frequently-misunderstood detail: **`apt-get update` and `apt-get install` must happen in the same `RUN` instruction.** If split across two instructions:

```dockerfile
# WRONG — a classic Dockerfile bug:
RUN apt-get update
RUN apt-get install -y curl
```

If the build cache later reuses the cached layer for `apt-get update` (because nothing above it changed) but the `apt-get install` line is new or changed, you get a stale package index combined with a fresh install command — potentially installing an outdated or unavailable package version, or failing outright when the cached index no longer matches what's actually available in the registry. Combining them into one `RUN` makes them atomic from the cache's perspective: either both are cached together, or both re-execute together.

### Shell Form vs. Exec Form — and Why It Matters for Signals

`RUN`, `CMD`, and `ENTRYPOINT` all support two syntaxes with meaningfully different runtime behavior:

```dockerfile
# Shell form — runs via /bin/sh -c "..."
CMD java -jar app.jar

# Exec form — runs the binary directly, no shell wrapper
CMD ["java", "-jar", "app.jar"]
```

This isn't a style choice. **Shell form wraps your command in a shell process, which becomes PID 1** — your actual application becomes a *child* of that shell, not PID 1 itself. This directly affects signal handling: when Docker sends `SIGTERM` to stop a container, it sends it to PID 1. If PID 1 is `/bin/sh`, many shells do not forward signals to child processes by default, meaning your JVM never receives the `SIGTERM` at all and only dies when the full `SIGKILL` grace period expires.

```mermaid
flowchart TB
    subgraph ShellForm["Shell form: CMD java -jar app.jar"]
        S1["PID 1: /bin/sh -c 'java -jar app.jar'"]
        S2["PID 8: java (child, does NOT get SIGTERM directly)"]
        S1 --> S2
    end
    subgraph ExecForm["Exec form: CMD [\"java\", \"-jar\", \"app.jar\"]"]
        E1["PID 1: java (receives SIGTERM directly)"]
    end
```

This is a real production bug pattern: containers that take the full `docker stop` grace period (default 10 seconds) to exit, every single time, instead of shutting down gracefully in under a second, because the shutdown signal never reached the JVM. **Always prefer exec form for `CMD`/`ENTRYPOINT`** unless you specifically need shell features (variable expansion, piping) in the startup command — and if you do, we cover the correct pattern (`exec` inside the shell script) in Phase 3, Chapter 2, which is dedicated entirely to signal handling.

---

## `CMD` vs. `ENTRYPOINT`: The Distinction That Actually Matters

Both define what runs when the container starts, but they compose differently:

- **`ENTRYPOINT`** defines the fixed, non-overridable (without `--entrypoint`) executable.
- **`CMD`** defines default *arguments* — easily overridden by anything passed after the image name in `docker run`.

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=default"]
```

```bash
# Uses the CMD default:
docker run myimage
# Runs: java -jar app.jar --spring.profiles.active=default

# Overrides CMD, keeps ENTRYPOINT fixed:
docker run myimage --spring.profiles.active=staging
# Runs: java -jar app.jar --spring.profiles.active=staging
```

This pattern — fixed `ENTRYPOINT`, overridable `CMD` — is the standard way to build an image that's "always runs this program" but allows callers to adjust arguments without needing to know or repeat the base command. If you only define `CMD` with no `ENTRYPOINT`, the entire thing (including the program itself) is trivially overridden by anyone running the image — sometimes desirable (a generic base image), rarely desirable for a specific service image.

---

## `COPY` vs. `ADD`: Prefer `COPY`, Almost Always

`ADD` does everything `COPY` does, plus: it can fetch a remote URL directly, and it auto-extracts local tar archives on copy. Both of these "extra" behaviors are footguns in a production Dockerfile:

```dockerfile
# ADD's URL-fetching hides a real dependency behind an obscure instruction,
# with no cache-busting guarantee if the remote content changes silently:
ADD https://example.com/some-tool.tar.gz /opt/

# Prefer being explicit — fetch via RUN with curl/wget, where the
# command's intent is visible and its caching behavior is unambiguous:
RUN curl -fsSL https://example.com/some-tool.tar.gz -o /tmp/tool.tar.gz \
    && tar -xzf /tmp/tool.tar.gz -C /opt/ \
    && rm /tmp/tool.tar.gz
```

The one legitimate, common use of `ADD` is local tar auto-extraction as an intentional, well-understood pattern (e.g., some official base images use it to lay down a root filesystem). Outside that specific case: **default to `COPY`.** It's explicit about what it does and nothing more.

---

## `ENV` vs. `ARG`: Build-Time vs. Run-Time, and Why Secrets Don't Belong in Either

```dockerfile
ARG BUILD_VERSION=dev
ENV APP_VERSION=${BUILD_VERSION}
```

- **`ARG`** exists only during the build. It is not present in the final image, not visible via `docker inspect`, not visible to the running container — unless you explicitly promote its value into an `ENV` (as shown above).
- **`ENV`** is baked into the image's metadata permanently. Every container run from this image will have `APP_VERSION` set, and — critically — **`ENV` values are visible via `docker inspect` and `docker history`, in plaintext, to anyone with access to the image.**

```bash
docker build --build-arg BUILD_VERSION=1.4.2 -t myimage .

docker inspect myimage --format '{{.Config.Env}}'
# [APP_VERSION=1.4.2 PATH=/usr/local/sbin:...]
```

This is precisely why **secrets (API keys, database passwords, private registry credentials) must never be passed via `ARG` or baked into `ENV`.** An `ARG` used mid-build (e.g., to authenticate a private dependency download) still leaves a trace in the image's build history layers unless handled via BuildKit's dedicated secret-mounting mechanism — which we cover properly in Chapter 5 (BuildKit internals), since it requires understanding BuildKit's execution model to use correctly. For now, the rule: **if it's sensitive, it doesn't go in a Dockerfile instruction at all — it gets injected at runtime** (environment variables passed to `docker run`/Compose/Kubernetes, or fetched from a secrets manager at startup), never at build time.

---

## `EXPOSE` Doesn't Do What Its Name Implies

```dockerfile
EXPOSE 8080
```

This is **pure documentation** — it doesn't open a port, doesn't publish anything to the host, and has zero effect on whether the containerized app can actually receive traffic on that port. The application inside the container still has to actually bind to `0.0.0.0:8080` (not `127.0.0.1:8080` — a genuinely common misconfiguration that makes an app unreachable from outside its own network namespace even with correct port publishing). Actual port publishing to the host happens at `docker run -p` time, entirely independent of whether `EXPOSE` was declared — we cover the full mechanics of this in Phase 4.

`EXPOSE`'s only functional effect: if you run `docker run -P` (capital P, publish *all* exposed ports to random host ports), Docker uses the `EXPOSE` list to know which ports qualify.

---

## A Complete, Annotated Example

Pulling this chapter together against the Phase 1 project's Dockerfile:

```dockerfile
FROM eclipse-temurin:21-jre                  # Base layer: JRE only, not JDK (why, in Ch. 4)
ARG APP_VERSION=dev                          # Build-time only, not in final image
LABEL org.opencontainers.image.version=$APP_VERSION
WORKDIR /app
COPY target/demo-0.0.1-SNAPSHOT.jar app.jar  # Explicit, single-purpose copy — no ADD
EXPOSE 8080                                  # Documentation only
USER 1000:1000                               # Non-root (Phase 8 covers why this matters)
ENTRYPOINT ["java", "-jar", "app.jar"]        # Exec form — becomes real PID 1, gets real signals
```

Every line here is a deliberate choice traceable to a concept in this chapter — nothing is a stylistic default.

---

## Common Misconceptions This Chapter Should Correct

- **"`EXPOSE` opens a port."** It documents intent; publishing happens at `docker run` time via `-p`.
- **"Shell form and exec form are interchangeable."** They differ in whether your process becomes real PID 1 and receives signals directly — this has real consequences for graceful shutdown, covered fully in Phase 3.
- **"`ARG` values are hidden/secure since they're 'build-time only.'"** They can still leak into layer history depending on how they're used; never use `ARG` for actual secrets without BuildKit's dedicated secret mount (Chapter 5).
- **"`ADD` is just a more powerful `COPY`, so it's always fine to use."** Its extra powers (URL fetch, auto-extract) reduce clarity and introduce non-obvious caching and security behavior; default to `COPY`.

---

## What's Next

We've established what each instruction does individually. Next is understanding precisely how Docker's build cache reasons about a *sequence* of these instructions — why instruction order is a genuine performance lever, what invalidates the cache, and how to structure a Dockerfile so that the expensive steps (dependency downloads) are cached correctly across builds.

**Next:** [`02-build-context-and-caching.md`](./02-build-context-and-caching.md)