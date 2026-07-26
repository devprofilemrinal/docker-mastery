# Project: Hardening a Production Container

This project takes the v5 image from Phase 2's optimization project and applies every hardening technique from this phase — non-root execution, dropped capabilities, a read-only filesystem, image scanning as a gate, and secrets fetched at runtime rather than baked in — producing a before/after comparison with a real scan report.

---

## Objective

- Start from Phase 2's optimized `demo-service:v5-final` (199MB, JRE-only, tini-based)
- Apply non-root execution, capability dropping, and a read-only root filesystem
- Scan both the "before" and "after" images and compare results
- Verify each hardening measure actually holds with a concrete test, not just configuration review

---

## The Hardened Dockerfile

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml .
COPY .mvn/ .mvn/
COPY mvnw .
COPY src ./src
RUN --mount=type=cache,target=/root/.m2 \
    ./mvnw clean package -DskipTests

FROM eclipse-temurin:21-jre
RUN apt-get update && apt-get install -y --no-install-recommends tini curl \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd --gid 1001 appgroup \
    && useradd --uid 1001 --gid appgroup --shell /usr/sbin/nologin --no-create-home appuser

WORKDIR /app
COPY --from=build /app/target/demo-0.0.1-SNAPSHOT.jar /app/app.jar
RUN chown -R appuser:appgroup /app

USER appuser
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --start-period=20s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health/readiness || exit 1

ENTRYPOINT ["tini", "--", "java", "-jar", "/app/app.jar"]
```

```bash
docker build -t demo-service:hardened .
```

---

## Applying Runtime Hardening Flags

```bash
docker run -d --name hardened-service \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  -p 8080:8080 \
  demo-service:hardened
```

- `--read-only` + `--tmpfs /tmp`: root filesystem is immutable; only the explicitly-provided tmpfs is writable (Chapter 2, Phase 5 Chapter 2)
- `--cap-drop=ALL`: no Linux capabilities beyond the bare minimum the kernel always requires (Chapter 1)
- `--security-opt=no-new-privileges`: prevents any process in the container from gaining additional privileges via `setuid` binaries or similar, even if one happened to be present

---

## Verifying Each Mitigation Actually Holds

### 1. Confirm non-root execution

```bash
docker exec hardened-service whoami
# appuser

docker exec hardened-service id
# uid=1001(appuser) gid=1001(appgroup)
```

### 2. Confirm the read-only filesystem is genuinely enforced

```bash
docker exec hardened-service sh -c "echo test > /app/malicious-write.txt"
# sh: can't create /app/malicious-write.txt: Read-only file system

# But /tmp, explicitly provided as tmpfs, remains writable:
docker exec hardened-service sh -c "echo test > /tmp/scratch.txt && cat /tmp/scratch.txt"
# test
```

### 3. Confirm capabilities are genuinely dropped

```bash
docker exec hardened-service cat /proc/1/status | grep Cap
# CapEff: 0000000000000000   <- zero effective capabilities
```

### 4. Confirm the application still works end to end

```bash
curl http://localhost:8080/api/hello
# {"message":"Hello from inside a container", ...}   <- unaffected by any hardening step
```

This last check matters as much as the security checks themselves — hardening that breaks the application isn't a usable outcome; every mitigation here was chosen specifically because it doesn't interfere with this service's actual runtime needs.

---

## Before/After Scan Comparison

```bash
trivy image demo-service:v5-final       # Phase 2's optimized, but not yet hardened, image
# Total: 6 (LOW: 4, MEDIUM: 2, HIGH: 0, CRITICAL: 0)

trivy image demo-service:hardened
# Total: 6 (LOW: 4, MEDIUM: 2, HIGH: 0, CRITICAL: 0)
```

**Worth being honest about what this comparison shows**: scan results are essentially unchanged, because non-root execution, capability dropping, and read-only filesystems are **runtime hardening**, not changes to installed package versions — a vulnerability scanner (Chapter 3) checks *what's installed*, not *what privileges the running container has*. These are complementary, non-overlapping layers of defense: scanning addresses "are there known-vulnerable packages," runtime hardening addresses "if something goes wrong anyway, how much can it actually do." Neither substitutes for the other.

---

## Debugging Note: Distroless Would Go Further, With a Trade-off

Following Phase 2 Chapter 4 and Phase 7 Chapter 3: swapping the final stage to a distroless base would further shrink both the scan surface and remove the shell entirely (closing off `docker exec ... sh` as an attacker's post-compromise tool, alongside your own convenient debugging path) — a deliberate additional step worth considering for a genuinely high-value production service, understanding the debugging trade-off it carries.

---

## Production Considerations

- `no-new-privileges` and `--cap-drop=ALL` should be defaults for essentially every application container in this repository's context — they cost nothing in ordinary operation and meaningfully shrink worst-case impact.
- These runtime flags (`--read-only`, `--cap-drop`, etc.) need to be applied wherever the container actually runs — Compose (`cap_drop:`, `read_only:` keys) and, eventually, Kubernetes (`securityContext`, Phase 10) have their own equivalent fields; the Dockerfile-level hardening (non-root `USER`) travels with the image everywhere, but the runtime flags do not and must be set at every deployment layer.

---

## Cleanup

```bash
docker rm -f hardened-service
docker rmi demo-service:hardened
```

---

## What's Next

Phase 8 is complete: the isolation-vs-VM mental model, non-root execution, image scanning, secrets handling, the named attack-surface categories, and a fully hardened, verified production image. Phase 9 moves to the CI/CD pipeline that builds, scans, and promotes exactly this kind of image through real environments.

**Next:** [`../phase-9-production-and-cicd/01-production-ready-dockerfile-checklist.md`](../phase-9-production-and-cicd/01-production-ready-dockerfile-checklist.md)