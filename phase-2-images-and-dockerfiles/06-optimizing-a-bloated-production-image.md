# Project: Optimizing a Bloated Production Image

Time to apply every lever from this phase at once, against a real before/after. We start from the Phase 1 project's Dockerfile (612MB, single-stage, no cache-aware ordering) and rebuild it applying: cache-aware layer ordering (Chapter 2), multi-stage builds (Chapter 3), a deliberately chosen base image (Chapter 4), and a BuildKit secret mount for a simulated private dependency (Chapter 5). We'll measure image size and rebuild speed at each step, not just assert improvement.

---

## Objective

- Start from the known-bad Phase 1 Dockerfile as a measured baseline
- Apply each optimization incrementally, measuring image size and rebuild time after each change
- End with a production-appropriate Dockerfile and a clear, numeric before/after comparison
- Demonstrate secret handling for a (simulated) private dependency, without leaking it into any layer

---

## Architecture

Same application as Phase 1 (the `demo-service` Spring Boot REST API) — this project is entirely about the Dockerfile, not new application code, so the improvements are cleanly attributable to Docker technique rather than app changes.

---

## Folder Structure

```text
spring-boot-docker-demo/
├── pom.xml
├── src/                          (unchanged from Phase 1)
├── .dockerignore                 (new)
├── Dockerfile.v1-baseline         (Phase 1's original, kept for comparison)
├── Dockerfile.v2-cache-ordered    (Chapter 2 fix)
├── Dockerfile.v3-multistage       (Chapter 3 fix)
├── Dockerfile.v4-optimized-base   (Chapter 4 fix)
└── Dockerfile                     (final production version — same as v4)
```

Keeping numbered intermediate Dockerfiles side-by-side (rather than only the final version) is deliberate for this project — it lets you rebuild any specific version and compare it directly, which is the entire pedagogical point.

---

## Step 0: Baseline — Measure What We Actually Start With

**`Dockerfile.v1-baseline`** (identical to Phase 1's project):

```dockerfile
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY . .
RUN ./mvnw clean package -DskipTests
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "target/demo-0.0.1-SNAPSHOT.jar"]
```

```bash
docker build -f Dockerfile.v1-baseline -t demo-service:v1 .
```

**Measure baseline size:**

```bash
docker images demo-service:v1 --format "{{.Size}}"
# 612MB
```

**Measure baseline rebuild time after a trivial source change** (edit a comment in `HelloController.java`, then):

```bash
time docker build -f Dockerfile.v1-baseline -t demo-service:v1 .
# real    0m52.891s     <- full Maven dependency re-download, every time
```

This is our baseline: **612MB, ~53 seconds for a one-line code change.**

---

## Step 1: Add `.dockerignore`

**`.dockerignore`:**

```text
.git
target/
*.log
.idea/
.vscode/
Dockerfile*
.dockerignore
README.md
```

```bash
docker build -f Dockerfile.v1-baseline -t demo-service:v1 .
# => Sending build context to Docker daemon  3.891MB
# (previously: likely 50-400MB+ depending on your local .git history and any stray target/ dir)
```

No image size change yet (this Dockerfile still copies everything relevant), but the *context transfer* step is now fast and won't silently balloon as your `.git` history grows. This is foundational hygiene, not yet the main optimization.

---

## Step 2: Cache-Aware Layer Ordering

**`Dockerfile.v2-cache-ordered`:**

```dockerfile
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY pom.xml .mvn mvnw ./
RUN ./mvnw dependency:go-offline -B
COPY src ./src
RUN ./mvnw clean package -DskipTests -o
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "target/demo-0.0.1-SNAPSHOT.jar"]
```

```bash
docker build -f Dockerfile.v2-cache-ordered -t demo-service:v2 .
docker images demo-service:v2 --format "{{.Size}}"
# 618MB   <- essentially unchanged; we haven't removed anything yet, just reordered
```

**Rebuild after the same trivial source edit:**

```bash
time docker build -f Dockerfile.v2-cache-ordered -t demo-service:v2 .
# => COPY pom.xml .mvn mvnw ./          CACHED
# => RUN ./mvnw dependency:go-offline   CACHED   <- the expensive step, skipped!
# => COPY src ./src                     (fast)
# => RUN ./mvnw clean package -o        ... 7.4s
# real    0m9.203s
```

**Size unchanged, but rebuild time dropped from ~53s to ~9s** — roughly 5.7x faster iteration, from reordering alone. This validates Chapter 2's core claim directly, with real numbers.

---

## Step 3: Multi-Stage Build

**`Dockerfile.v3-multistage`:**

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml .mvn mvnw ./
RUN ./mvnw dependency:go-offline -B
COPY src ./src
RUN ./mvnw clean package -DskipTests -o

FROM eclipse-temurin:21-jre AS runtime
WORKDIR /app
COPY --from=build /app/target/demo-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build -f Dockerfile.v3-multistage -t demo-service:v3 .
docker images demo-service:v3 --format "{{.Size}}"
# 241MB   <- down from 618MB, purely from dropping JDK + Maven cache + source tree
```

Rebuild time after a trivial source edit remains fast (~9s) since we preserved the Chapter 2 ordering inside the `build` stage — **this step is a pure size win with no speed regression.**

---

## Step 4: Choose a Deliberate Base Image (Alpine, After Verification)

We verified (per Chapter 4's guidance) that this application has no native library dependencies — pure Spring Boot Web + Actuator, no JNI, no native crypto libraries — so Alpine is a safe choice here.

**`Dockerfile.v4-optimized-base`:**

```dockerfile
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml .mvn mvnw ./
RUN ./mvnw dependency:go-offline -B
COPY src ./src
RUN ./mvnw clean package -DskipTests -o

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=build /app/target/demo-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build -f Dockerfile.v4-optimized-base -t demo-service:v4 .
docker images demo-service:v4 --format "{{.Size}}"
# 128MB   <- down from 241MB
```

**Functional verification (not just "did it build"):**

```bash
docker run -d --name v4-test -p 8080:8080 demo-service:v4
curl http://localhost:8080/api/hello
# {"message":"Hello from inside a container","timestamp":"...","pid":1}

curl http://localhost:8080/actuator/health
# {"status":"UP"}

docker rm -f v4-test
```

Both endpoints work correctly — confirming the base image swap didn't silently break anything, per Chapter 4's explicit warning about verifying, not just assuming.

---

## Step 5: Simulated Private Dependency with a Build Secret

To demonstrate Chapter 5's secret-mounting pattern concretely, we'll simulate authenticating to a private Maven repository during the dependency download step (using a placeholder token file — in a real project this would be a genuine private repo credential):

```bash
mkdir -p ~/.secrets
echo "simulated-private-repo-token-abc123" > ~/.secrets/maven_token.txt
```

**Add to the top of `Dockerfile.v4-optimized-base`** (or the final `Dockerfile`):

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml .mvn mvnw ./

RUN --mount=type=secret,id=maven_token \
    if [ -f /run/secrets/maven_token ]; then \
      echo "Authenticated with private repo token (not shown)"; \
    fi && \
    ./mvnw dependency:go-offline -B

COPY src ./src
RUN ./mvnw clean package -DskipTests -o

FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app
COPY --from=build /app/target/demo-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
docker build \
  --secret id=maven_token,src=$HOME/.secrets/maven_token.txt \
  -t demo-service:final .
```

**Verify the secret never landed in a layer:**

```bash
docker history --no-trunc demo-service:final | grep -i "abc123"
# (no output — the token never appears anywhere in the image's layer history)
```

---

## Final Results Table

| Version | Change applied | Image size | Rebuild time (1-line src change) |
|---|---|---|---|
| v1 (baseline) | None — Phase 1 original | 612MB | ~53s |
| v2 | Cache-aware layer ordering | 618MB* | ~9s |
| v3 | Multi-stage build | 241MB | ~9s |
| v4 | Alpine JRE base (verified safe) | 128MB | ~9s |
| final | + BuildKit secret mount (no size/speed change, security fix) | 128MB | ~9s |

*v2's size is marginally higher than v1 due to slightly different layer boundaries — expected and irrelevant, since v2's entire purpose is rebuild speed, not size; size only improves starting at v3.

**Net result: 612MB → 128MB (79% smaller), and ~53s → ~9s per iterative rebuild (83% faster).**

---

## Debugging Walkthrough: Confirming Nothing Broke Along the Way

At each step, the same verification applies — don't just trust the size number:

```bash
# Run whichever version you just built:
docker run -d --name verify-step -p 8080:8080 demo-service:v4

# Confirm the API works:
curl -f http://localhost:8080/api/hello || echo "FAILED"

# Confirm actuator health works (we'll lean on this heavily in Phase 7):
curl -f http://localhost:8080/actuator/health || echo "FAILED"

# Confirm the JVM is genuinely PID 1 (Phase 1, Chapter 2 concept, still holds):
docker exec verify-step ps aux
# PID 1 should be the java process, not a shell wrapper

docker rm -f verify-step
```

---

## Common Mistakes

- **Declaring victory on image size alone, without functionally testing the running container** — a smaller image that silently fails at runtime (e.g., due to an Alpine/musl incompatibility you didn't check for) is strictly worse than a larger, working one.
- **Forgetting `--secret` on the build command** after adding `--mount=type=secret` to the Dockerfile — the build will simply proceed without the secret available (which may or may not fail loudly, depending on how your `RUN` command handles a missing file), so verify the secret mount is actually working, not just present in the Dockerfile text.
- **Leaving old `Dockerfile.v1`-style single-stage files as the one actually referenced by CI/CD**, having "fixed" it only in a differently-named file that nothing actually builds from.

---

## Production Considerations

- This final image is still running as root by default (Alpine JRE images don't set a non-root user out of the box) — Phase 8 fixes this explicitly.
- No `HEALTHCHECK` instruction is defined yet — Phase 3 and Phase 7 cover this, including the actuator endpoint we just verified manually.
- Multi-arch support (building this same optimized image for both `amd64` and `arm64`) is not yet addressed — that's Phase 9.

---

## What's Next

Phase 2 is complete. We've taken a functionally-correct-but-bloated image and turned it into something genuinely production-appropriate, with real, measured numbers at every step — not just "trust me, this is better." Phase 3 shifts focus from *building* the image to *running* it: what actually happens to your process once it's alive inside a container, how PID 1 signal handling really works, how cgroup memory limits interact with JVM heap sizing in practice, and how to reason about logging correctly inside a container.

**Next:** [`../phase-3-runtime-and-processes/01-container-lifecycle-and-pid1.md`](../phase-3-runtime-and-processes/01-container-lifecycle-and-pid1.md)