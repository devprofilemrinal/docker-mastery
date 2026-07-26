# Translating Compose Concepts to Kubernetes

This chapter is a direct, side-by-side translation of the Phase 6 Compose stack's specific YAML constructs into their Kubernetes equivalents — not abstract mapping, but literal before/after snippets, so the transition from "I know Compose" to "I can read a Kubernetes manifest" is concrete rather than conceptual.

---

## Services → Deployments + Services

A Compose `service` bundles "what to run" and "how it's reachable" into one block. Kubernetes splits these into two separate resources: a **Deployment** (what to run, how many replicas, how to roll out updates) and a **Service** (how it's reachable by name — the DNS mechanism from Phase 4, Chapter 3, now cluster-wide).

```yaml
# Compose (Phase 6)
services:
  order-api:
    build: ./order-api
    ports:
      - "8080:8080"
```

```yaml
# Kubernetes Deployment — "what to run"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-api
  template:
    metadata:
      labels:
        app: order-api
    spec:
      containers:
        - name: order-api
          image: myorg/order-service:1.4.2   # built by Phase 9's pipeline, unchanged
          ports:
            - containerPort: 8080
---
# Kubernetes Service — "how it's reachable by name" (Phase 4, Ch.3's DNS, cluster-wide)
apiVersion: v1
kind: Service
metadata:
  name: order-api
spec:
  selector:
    app: order-api
  ports:
    - port: 8080
      targetPort: 8080
```

Other Pods in the cluster reach this via `http://order-api:8080` — **the identical DNS-by-name pattern from Phase 4 Chapter 3**, just resolved by Kubernetes' cluster DNS (CoreDNS) instead of Docker's embedded resolver, and working across every node in the cluster rather than one host's bridge network.

---

## `depends_on` + Healthcheck → Init Containers + Probes

Phase 6 Chapter 2 established that `condition: service_healthy` gates Compose's startup order. Kubernetes splits this into two genuinely distinct mechanisms:

```yaml
# Compose
order-api:
  depends_on:
    postgres:
      condition: service_healthy
```

```yaml
# Kubernetes: an init container BLOCKS the main container from starting
# until postgres is reachable — the direct equivalent of the startup gate:
spec:
  initContainers:
    - name: wait-for-postgres
      image: busybox
      command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
  containers:
    - name: order-api
      image: myorg/order-service:1.4.2
      readinessProbe:            # <- the Phase 7, Ch.4 distinction, now a real, separate field
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 10
      livenessProbe:             # <- genuinely separate from readiness, unlike Docker's one healthcheck
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 30
        periodSeconds: 15
```

This is the **exact** readiness/liveness distinction argued for in Phase 7 Chapter 4 — Kubernetes gives you two genuinely independent probes rather than Docker's single `HEALTHCHECK`, resolving the "conflating the two causes unwanted restarts" problem directly at the platform level.

---

## Named Volumes → PersistentVolumeClaims

```yaml
# Compose
volumes:
  - pg-data:/var/lib/postgresql/data
```

```yaml
# Kubernetes
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pg-data
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 10Gi
---
# referenced in the Postgres Pod's spec:
      volumeMounts:
        - name: pg-storage
          mountPath: /var/lib/postgresql/data
  volumes:
    - name: pg-storage
      persistentVolumeClaim:
        claimName: pg-data
```

The conceptual goal — data outliving any specific container instance (Phase 5, Chapter 2) — is identical; Kubernetes additionally decouples the data from any specific *node*, addressing the "default `local` volume driver ties data to one host" limitation flagged in Phase 5 Chapter 3.

---

## `.env`/Secrets → ConfigMaps and Secrets

```yaml
# Compose
environment:
  SPRING_PROFILES_ACTIVE: docker
secrets:
  - pg_password
```

```yaml
# Kubernetes ConfigMap for non-sensitive config:
apiVersion: v1
kind: ConfigMap
metadata:
  name: order-api-config
data:
  SPRING_PROFILES_ACTIVE: docker
---
# Kubernetes Secret for sensitive values (still base64-encoded, NOT
# encrypted at rest by default — the Phase 8, Ch.4 honest caveat about
# Docker's secrets: applies almost identically here; a real secrets
# manager integration remains the correct answer for genuinely
# sensitive production credentials):
apiVersion: v1
kind: Secret
metadata:
  name: pg-credentials
type: Opaque
data:
  password: c2VjcmV0   # base64, NOT strong encryption
```

---

## Common Misconceptions

- **"Kubernetes manifests are a totally different mental model from Compose."** Every construct above has a direct, traceable Compose equivalent — Kubernetes is more verbose and more explicit (separating concerns Compose bundles together), not conceptually foreign.
- **"Kubernetes Secrets are meaningfully more secure than Compose's `secrets:`."** Base64 is encoding, not encryption — the same honest caveat from Phase 8 Chapter 4 applies; genuine secret security still requires encryption-at-rest configuration and/or an external secrets manager integration.
- **"An init container and a readiness probe solve the same problem."** Init containers gate *initial Pod startup*; readiness probes govern *ongoing traffic routing* even after startup — genuinely different scopes, mirroring the readiness/liveness distinction itself.

---

## What's Next

**Next:** [`04-image-requirements-for-k8s-workloads.md`](./04-image-requirements-for-k8s-workloads.md)