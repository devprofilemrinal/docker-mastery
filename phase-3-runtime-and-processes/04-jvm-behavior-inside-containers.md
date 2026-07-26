# JVM Behavior Inside Containers

The previous chapter established that a cgroup memory limit is a budget for the *entire* container, not just JVM heap, and that the JVM tries to detect its cgroup-constrained CPU quota to size internal thread pools. This chapter makes both of those facts concrete and actionable: how the JVM actually detects container constraints, what its defaults do with that information, and a sizing approach you can apply directly rather than guessing.

---

## Container Awareness: What It Is, and Since When

Before Java 8u191 (backported) and reliably from Java 10 onward, the JVM had no idea it was running inside a cgroup-limited container — it queried the host's `/proc/meminfo` and CPU count directly, completely ignoring any cgroup constraints. This meant a JVM in a container limited to 512MB, running on a host with 64GB total RAM, would size its default heap (`-Xmx` unset) based on a fraction of **64GB**, not 512MB — reliably leading to OOM-kills, because the JVM believed it had far more room than it actually did.

Modern JVMs fix this via **container awareness** (enabled by default since JDK 10, and the current expected behavior on any JVM you'd deploy today): the JVM reads cgroup limit files directly (the same `memory.max` and `cpu.max` files from the previous chapter) and uses *those* numbers, not the host's, for its default ergonomics.

```bash
# Verify container awareness is active and what it's detecting,
# directly, rather than assuming it's working correctly:
docker run --rm --memory=512m eclipse-temurin:21-jre \
  java -XX:+PrintFlagsFinal -version 2>/dev/null | grep -i maxheapsize
# uintx MaxHeapSize = 134217728    <- 128MB: ~25% of the 512MB cgroup limit,
#                                      NOT 25% of the host's total RAM
```

That 25% figure is not a coincidence — it's the JVM's default `-XX:MaxRAMPercentage`, which we cover next.

---

## Default Heap Sizing: `-XX:MaxRAMPercentage`, Not a Fixed Number

Without an explicit `-Xmx`, the JVM computes its default max heap as a **percentage of detected available memory** (container-aware, per above), controlled by `-XX:MaxRAMPercentage` (default `25.0`):

```bash
# Explicit: heap can use up to 75% of whatever memory the container sees.
docker run -d --memory=1g \
  -e JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75.0" \
  demo-service:1.0
```

This percentage-based approach is genuinely more container-friendly than a hardcoded `-Xmx` value baked into an image, because the *same* image can run correctly across environments with different memory limits (a dev container at 256MB, a production container at 2GB) without needing environment-specific rebuilds — a direct payoff of the "same image everywhere" principle from Phase 1, now applied specifically to JVM memory configuration.

However, **25% (the default) is deliberately conservative** and, per the previous chapter's diagram, leaves a large amount of headroom for metaspace, thread stacks, and native memory — appropriate for a first pass, but usually worth tuning explicitly for a production service once you understand your application's actual non-heap memory footprint.

---

## A Concrete Sizing Approach

Rather than guessing, measure your specific application's actual memory breakdown and size deliberately:

```bash
# 1. Run your service with generous headroom and observe real usage
#    under representative load, using the JVM's own reporting:
docker run -d --name sizing-probe --memory=1g \
  -e JAVA_TOOL_OPTIONS="-Xlog:gc -XX:+PrintFlagsFinal" \
  demo-service:1.0

# 2. Send representative load, then inspect actual native memory
#    usage via NMT (Native Memory Tracking) if enabled, or via
#    the container's own cgroup accounting for a coarser view:
docker exec sizing-probe cat /sys/fs/cgroup/memory.current

# 3. Cross-reference against what the heap itself is actually using:
docker exec sizing-probe jcmd 1 GC.heap_info
```

A reasonable production starting allocation for a typical Spring Boot service, once you've measured rather than guessed:

| Component | Rough share of container memory limit |
|---|---|
| JVM Heap (`-Xmx`) | 50–65% |
| Metaspace | 5–10% (bounded explicitly with `-XX:MaxMetaspaceSize`) |
| Thread stacks | Depends on thread count × `-Xss` (default ~1MB/thread — a service with 200 threads is ~200MB before any heap at all) |
| Direct/native buffers, JIT code cache, GC overhead | Remaining 20–30% headroom |

```dockerfile
# Explicit, deliberately-sized JVM flags baked into the image's entrypoint,
# rather than relying solely on the default 25% MaxRAMPercentage:
ENTRYPOINT ["java", \
  "-XX:MaxRAMPercentage=60.0", \
  "-XX:MaxMetaspaceSize=128m", \
  "-XX:+ExitOnOutOfMemoryError", \
  "-jar", "/app/app.jar"]
```

`-XX:+ExitOnOutOfMemoryError` is worth calling out specifically: without it, a JVM that hits an `OutOfMemoryError` on the *heap* (a catchable Java exception, unlike the kernel's uncatchable cgroup OOM-kill) can sometimes limp along in a corrupted, partially-functional state rather than exiting — which is worse for a containerized service than a clean, fast exit that lets the orchestrator (Compose restart policy, Kubernetes) restart it promptly. This flag makes heap OOM behavior converge with container restart semantics deliberately, rather than leaving it to chance.

---

## Thread Pool Sizing and `ActiveProcessorCount`

Following on from the previous chapter's CPU quota discussion: Spring Boot's default thread pools (Tomcat's worker threads, the common `ForkJoinPool` used by parallel streams, `@Async` executors if unconfigured) are frequently sized based on `Runtime.getRuntime().availableProcessors()` — which, on a container-aware JVM, reflects the **cgroup CPU quota**, not the host's physical core count.

```java
// This code's behavior genuinely changes based on --cpus, not the host's
// actual core count — worth verifying explicitly, not assuming:
int cores = Runtime.getRuntime().availableProcessors();
System.out.println("JVM sees " + cores + " available processors");
```

```bash
docker run --rm --cpus=2 demo-service:1.0
# JVM sees 2 available processors

docker run --rm --cpus=0.5 demo-service:1.0
# JVM sees 1 available processors  (rounds up — verify exact behavior
# for your specific JVM version rather than assuming this rounding rule
# holds universally across versions)
```

A container with a very low `--cpus` quota (say `0.25`, common in dense multi-tenant clusters) can end up with a thread pool sized for essentially a single core, even if the application logic assumes more parallelism is available — a mismatch that shows up as unexpectedly serialized throughput, not an obvious error.

---

## Common Misconceptions This Chapter Should Correct

- **"Modern JVMs automatically get container memory sizing exactly right with zero configuration."** Container awareness (detecting the *limit*) has been solid since JDK 10, but the *default percentage* (25% of that limit) is a generic, conservative default — not a value tuned to your specific application's actual metaspace/thread/native footprint. Measure and set explicitly for production.
- **"Setting `-Xmx` directly is always better than `-XX:MaxRAMPercentage`."** A hardcoded `-Xmx` is *less* portable across environments with different memory limits — the percentage-based approach is usually preferable specifically because it travels correctly with the same image across differently-sized containers. Use an explicit `-Xmx` only when you have a specific, deliberate reason to fix an absolute number regardless of container size.
- **"`OutOfMemoryError` (JVM heap) and OOM-killed (cgroup, from the kernel) are the same event."** They are two entirely separate mechanisms with different triggers, different catchability, and different symptoms — conflating them leads to debugging the wrong layer.
- **"CPU/memory container-awareness works identically across all JVM distributions and versions."** The core mechanism (reading cgroup files) is standardized, but exact rounding behavior, default percentages, and edge-case handling have evolved across releases — always verify against your actual deployed JVM version rather than assuming behavior from a different version or vendor.

---

## What's Next

The next chapter shifts from memory/CPU to a different but equally practical runtime concern: how your application's logs should actually get out of the container and into whatever's collecting them — the stdout/stderr conventions that make container log aggregation work correctly (or, done wrong, silently lose or duplicate your logs).

**Next:** [`05-logging-and-stdout-stderr-conventions.md`](./05-logging-and-stdout-stderr-conventions.md)