# Capstone: Build, Run, Debug, and Productionize

The final chapter: running the full system locally via Compose, exercising Phase 7's debugging techniques against it, then walking the Phase 9 production checklist and the Phase 10 Kubernetes deployment path.

---

## Build and Run (Local, via Compose)

```bash
cd capstone-ecommerce
cp .env.example .env   # set a real POSTGRES_PASSWORD

docker compose up -d --build
docker compose ps
# All services should reach "Up (healthy)" in dependency order,
# per the health-gated startup chain from Phase 6, Chapter 2.
```

## Expected Output: Full Request Flow

```bash
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{"sku":"WIDGET-001","quantity":3}'
# {"id":1,"sku":"WIDGET-001","quantity":3,"createdAt":"..."}

docker compose logs notification-service --tail 3
# notification-service-1 | Sending notification for: OrderPlaced:WIDGET-001
```

Both the synchronous path (gateway → order-service → inventory-service, visible in the immediate response) and the asynchronous path (order-service → kafka → notification-service, visible moments later in logs) are confirmed working — exactly the dual-pattern architecture from the capstone's opening chapter.

---

## Debugging Walkthrough: Applying Phase 7 to a Real Multi-Service Failure

**Deliberately introduce a failure**: stop `inventory-service` mid-session.

```bash
docker compose stop inventory-service

curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" -d '{"sku":"WIDGET-001","quantity":1}'
# 500 Internal Server Error (or a gateway timeout, depending on gateway config)
```

**Diagnose it, following Phase 7 Chapter 2's checklist:**

```bash
docker compose logs order-service --tail 10
# Caused by: org.springframework.web.client.ResourceAccessException:
#   I/O error on GET request for "http://inventory-service:8080/...":
#   Connection refused

docker compose ps inventory-service
# STATUS: Exited (0)   <- confirms it's stopped, not crashed unexpectedly
```

**Root cause confirmed**: `order-service`'s synchronous call to `inventory-service` fails cleanly with a connection-refused error — exactly the expected behavior for the synchronous pattern from Phase 4 Chapter 5, correctly surfacing the dependency's unavailability rather than hanging indefinitely (thanks to the explicit timeouts configured on `InventoryClient`).

```bash
docker compose start inventory-service
# Confirm recovery:
curl -X POST http://localhost:8080/api/orders -H "Content-Type: application/json" -d '{"sku":"WIDGET-001","quantity":1}'
# 200 OK — recovered correctly once the dependency returned.
```

---

## Production Readiness Review (Phase 9, Chapter 1's Checklist, Applied)

| Checklist item | Status |
|---|---|
| Pinned base image tags | ✓ `eclipse-temurin:21-jdk`/`21-jre`, no `:latest` anywhere |
| Multi-stage builds | ✓ every service's Dockerfile |
| JRE-only final stage | ✓ |
| Cache-efficient layer ordering | ✓ dependency files copied before source |
| `.dockerignore` present | ✓ excludes `target/`, `.git` |
| Exec-form ENTRYPOINT + tini | ✓ uniformly applied |
| Non-root `USER` | ✓ |
| Readiness-based `HEALTHCHECK` | ✓ |
| No baked-in secrets | ✓ `POSTGRES_PASSWORD` via `.env`, gitignored |
| Scanned, no unresolved HIGH/CRITICAL | Run `trivy image` against each service before considering this checked |

```bash
for service in api-gateway order-service inventory-service notification-service; do
  docker build -t capstone/$service:1.0 ./$service
  trivy image --exit-code 1 --severity HIGH,CRITICAL capstone/$service:1.0
done
```

---

## Productionizing: The Kubernetes Path (Phase 10, Applied)

```bash
kubectl create namespace ecommerce
kubectl apply -n ecommerce -f k8s/data/          # postgres, redis, kafka
kubectl apply -n ecommerce -f k8s/order-service/
kubectl apply -n ecommerce -f k8s/inventory-service/
kubectl apply -n ecommerce -f k8s/notification-service/
kubectl apply -n ecommerce -f k8s/api-gateway/

kubectl get pods -n ecommerce
```

Every Deployment here carries the `securityContext` hardening, resource `requests`/`limits`, and separate readiness/liveness probes from Phase 10 Chapter 4 — the exact same translation pattern demonstrated in Phase 10's project, applied across all four Spring Boot services rather than just one.

```bash
kubectl port-forward svc/api-gateway -n ecommerce 8080:8080
curl -X POST http://localhost:8080/api/orders -H "Content-Type: application/json" -d '{"sku":"WIDGET-001","quantity":2}'
```

---

## What You've Built

Across ten phases and this capstone, you've gone from "what problem does Docker solve" to operating a multi-service, event-driven backend — hardened, scanned, CI-piped, and deployed to Kubernetes — with a working understanding of *why* every layer behaves the way it does, from the kernel namespace/cgroup mechanics up through cluster-level orchestration. This is the complete arc: not memorized commands, but a mental model precise enough to debug, optimize, and extend a real system confidently.

---

## Closing Note on Continuing From Here

Real-world extensions worth exploring beyond this repository's scope: distributed tracing across the synchronous/asynchronous boundary (correlating a request through `order-service` → Kafka → `notification-service`), a service mesh for more sophisticated traffic management than plain Kubernetes Services provide, and GitOps-style deployment (Argo CD or Flux) building directly on Phase 9's promotion-by-digest discipline. Every one of these builds on the foundation this repository established — none of them require unlearning anything covered here.