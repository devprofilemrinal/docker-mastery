# Running as Non-Root

Chapter 1 established why root inside a container matters: without user namespace remapping, it's genuinely the host's UID 0. This chapter is the practical fix — running your Spring Boot container as an unprivileged user, applied to the exact image built across Phase 2.

---

## The Default You're Fighting

Unless a Dockerfile explicitly sets a `USER`, the container's main process runs as **root** — this is the base image's default, inherited silently unless overridden.

```bash
docker run --rm eclipse-temurin:21-jre whoami
# root
```

## Creating and Switching to a Dedicated User

```dockerfile
FROM eclipse-temurin:21-jre

# Create a dedicated, unprivileged user and group — never reuse an
# existing system account, and never just use UID 1000 blindly
# without confirming it doesn't collide with something meaningful
# on your specific base image:
RUN groupadd --gid 1001 appgroup \
    && useradd --uid 1001 --gid appgroup --shell /usr/sbin/nologin --no-create-home appuser

WORKDIR /app
COPY --from=build /app/target/demo-0.0.1-SNAPSHOT.jar /app/app.jar

# Ensure the app's own files are actually readable by the new user —
# a common oversight: switching USER without fixing file ownership
# just trades "runs as root" for "can't read its own JAR":
RUN chown -R appuser:appgroup /app

USER appuser

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

```bash
docker build -t demo-service:nonroot .
docker run --rm demo-service:nonroot whoami
# appuser

docker run --rm demo-service:nonroot id
# uid=1001(appuser) gid=1001(appgroup) groups=1001(appgroup)
```

---

## The Privileged-Port Trap

A common issue immediately after switching off root: **binding to ports below 1024 requires root privileges on Linux**, by long-standing kernel convention (independent of containers entirely).

```bash
docker run --rm demo-service:nonroot java -jar /app/app.jar --server.port=80
# Caused by: java.net.BindException: Permission denied
```

The fix is **not** to run as root again — it's to run your application on an unprivileged port (Spring Boot's default 8080 is already a sensible, unprivileged choice) and let the *published port mapping* (Phase 4, Chapter 4) handle any external-facing low-port requirement:

```bash
# App listens on unprivileged 8080 internally; externally it can still
# appear to be on port 80 via ordinary port publishing/mapping:
docker run -d -p 80:8080 demo-service:nonroot
```

If you specifically need the container itself to bind a low port without root (rare for an app behind a reverse proxy, but occasionally relevant), the `NET_BIND_SERVICE` capability from Chapter 1 is the correct, narrow fix — not reverting to a full-root user.

---

## Verifying Kernel-Level Enforcement, Not Just Convention

```bash
# Confirm the restriction is a real kernel-enforced permission check,
# not merely something Docker is politely respecting:
docker run --rm --user 1001:1001 alpine touch /root/test-file
# touch: /root/test-file: Permission denied
```

This is genuine, standard Unix file-permission enforcement, operating exactly as it would on a bare Linux host — containers don't add a new permission system, they run ordinary Linux permission checks inside an isolated namespace view (Phase 1, Chapter 2).

---

## Read-Only Root Filesystem: An Additional, Complementary Hardening Step

Beyond non-root execution, marking the container's root filesystem read-only closes off another entire class of risk — a compromised process attempting to write a malicious file to disk simply cannot, regardless of what user it's running as:

```bash
docker run -d --read-only \
  --tmpfs /tmp \
  demo-service:nonroot
```

`--tmpfs /tmp` is necessary here because many applications (including the JVM itself, for certain temp file operations) expect *some* writable path to exist — Chapter 2 of Phase 5's `tmpfs` mechanism is the right tool for providing exactly that narrow, RAM-only writable exception without compromising the read-only guarantee everywhere else.

---

## Common Misconceptions

- **"Switching `USER` in a Dockerfile is a purely cosmetic change."** It's a genuine, kernel-enforced permission change — verified directly above via a real permission-denied error, not a Docker-level convention.
- **"You need root to bind to any port a container publishes externally."** The container's *internal* listening port can be any unprivileged port; the externally-visible port number is a separate mapping concern (Phase 4, Chapter 4), entirely decoupled from what the process inside actually binds to.
- **"Non-root execution alone is sufficient hardening."** It's one layer among several (capabilities, read-only filesystem, image scanning in the next chapter) — defense in depth, not a single silver-bullet fix.

---

## What's Next

**Next:** [`03-image-scanning-and-supply-chain-risk.md`](./03-image-scanning-and-supply-chain-risk.md)