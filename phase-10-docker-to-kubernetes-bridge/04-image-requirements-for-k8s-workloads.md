# Image Requirements for Kubernetes Workloads

Every hardening and correctness practice from Phases 2, 3, and 8 was framed as good practice generally — this chapter is about which of those become **functionally required**, not merely advisable, once Kubernetes is the orchestrator, because Kubernetes' own mechanisms depend on your image behaving correctly in specific ways.

---

## Correct Signal Handling Is No Longer Optional (Phase 3, Chapter 1)

Kubernetes sends `SIGTERM` to a Pod's containers before removing them during a rolling update, scale-down, or eviction — exactly like `docker stop`, but happening **far more frequently** in a Kubernetes environment (autoscaling, node maintenance, and rolling deploys all trigger this routinely, not just occasional manual intervention).

```yaml
spec:
  terminationGracePeriodSeconds: 30   # Kubernetes' equivalent of docker stop's --time flag
  containers:
    - name: order-api
      image: myorg/order-service:1.4.2
```

If your image's `ENTRYPOINT` is still shell-form, or lacks tini (Phase 3, Chapter 1's PID 1 fix), the *frequency* at which Kubernetes routinely restarts and reschedules Pods turns an occasional annoyance into an ongoing, continuous source of ungraceful terminations, dropped connections, and orphaned processes — this repository's PID 1 discipline stops being "good practice for the rare `docker stop`" and becomes load-bearing infrastructure correctness.

---

## Readiness Probes Require an Accurate, Cheap Endpoint

Kubernetes' `readinessProbe` (Chapter 3) is called **repeatedly, continuously**, for the entire lifetime of every Pod — not just once at startup like Compose's health-gated `depends_on`. This has two direct consequences for your image:

1. **The endpoint must be cheap.** A readiness check hitting your database on every single probe interval (commonly every 10 seconds, for every replica) adds continuous, avoidable load — Spring Boot Actuator's health groups (Phase 7, Chapter 4) let you scope exactly what a readiness check verifies, deliberately keeping it lightweight.
2. **The endpoint must be accurate.** A readiness probe that always returns healthy regardless of actual state means Kubernetes will happily route traffic to a Pod that can't actually serve it — silently defeating the entire purpose of having the probe.

---

## Resource Requests and Limits Are Now Load-Bearing for Scheduling

Phase 3 Chapter 3 covered `--memory`/`--cpus` as runtime enforcement. Kubernetes adds a **scheduling** dimension that Docker alone never had: the **request** value is what the scheduler uses to decide which node has room for this Pod *before* it ever starts — distinct from the **limit**, which is the same hard cgroup ceiling from Phase 3.

```yaml
resources:
  requests:
    memory: "400Mi"   # scheduling hint: "reserve at least this much on some node"
    cpu: "250m"
  limits:
    memory: "512Mi"   # hard ceiling — same cgroup mechanism as Phase 3, Chapter 3
    cpu: "500m"
```

Omitting `requests` entirely means the scheduler has no informed basis for placement, which can lead to a node being genuinely overcommitted despite every individual Pod's `limits` being reasonable in isolation — the measured JVM sizing methodology from Phase 3's capstone project is exactly what should inform both numbers here, not just the `limits` value.

---

## Non-Root and Read-Only Filesystem Have Dedicated, Enforceable Fields

Phase 8's runtime flags (`--cap-drop`, `--read-only`, non-root `USER`) map directly onto Kubernetes' `securityContext`, and — importantly — can be made **mandatory cluster-wide policy**, not just a per-Dockerfile discipline:

```yaml
spec:
  securityContext:
    runAsNonRoot: true          # Kubernetes will REFUSE to start a Pod
                                  # whose image doesn't specify a non-root USER
    fsGroup: 1001
  containers:
    - name: order-api
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop: ["ALL"]
```

`runAsNonRoot: true` is worth specifically calling out: with this set, **Kubernetes will refuse to even start the Pod** if the image's default user is root and no `runAsUser` override is provided — turning Phase 8 Chapter 2's non-root recommendation into a hard, cluster-enforced admission requirement rather than a convention that depends on every Dockerfile author remembering to apply it.

---

## Common Misconceptions

- **"Everything from Phases 2, 3, and 8 was 'nice to have' and remains equally optional under Kubernetes."** Several of these practices (correct PID 1/signal handling, accurate lightweight readiness endpoints, non-root execution) shift from advisable to functionally load-bearing, precisely because Kubernetes exercises them continuously and can enforce some of them as hard admission requirements.
- **"Kubernetes resource `limits` alone are sufficient; `requests` are optional extra detail."** Omitting `requests` removes the scheduler's ability to place Pods sensibly across the cluster — a real, observable operational problem, not a cosmetic omission.
- **"`readinessProbe` and Docker's `HEALTHCHECK` can share the identical endpoint with no reconsideration."** They can, but the *frequency and continuity* of Kubernetes' probing (versus Docker/Compose's more occasional checks) means cost and accuracy of that endpoint matter more under Kubernetes than they did before.

---

## What's Next

Time to take the entire Phase 6 Compose stack and migrate it, piece by piece, into a working set of Kubernetes manifests — applying every mapping and requirement from this phase concretely.

**Next:** [`05-migrating-the-compose-stack-to-kubernetes-manifests.md`](./05-migrating-the-compose-stack-to-kubernetes-manifests.md)