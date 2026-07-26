# Docker Mastery: A Deep Learning Repository for Backend Engineers

A complete Docker learning path built for an experienced Java Spring Boot backend engineer who already knows Java, Maven/Gradle, Linux basics, REST APIs, Git, microservices, and basic Kubernetes — and wants to understand Docker deeply enough to confidently build, debug, optimize, and deploy containerized production systems.

This is not a command reference. Every chapter explains **why** Docker behaves the way it does — the kernel mechanisms, the engineering trade-offs, the production consequences — before showing you how to use it. Diagrams, code, and hands-on projects are woven directly into each chapter; nothing requires jumping between files to follow along.

---

## How This Repository Is Organized

```text
docker-mastery/
├── phase-1-foundations/                    Kernel mechanics: namespaces, cgroups, layers, architecture
├── phase-2-images-and-dockerfiles/         Building efficient, correct, cache-friendly images
├── phase-3-runtime-and-processes/          PID 1, signals, cgroup limits, JVM behavior, logging
├── phase-4-networking/                     Bridge networking, DNS, packet-level mechanics
├── phase-5-storage-and-state/              Volumes, bind mounts, tmpfs, database containers
├── phase-6-docker-compose/                 Multi-container orchestration on one host
├── phase-7-debugging-and-observability/    Systematic diagnosis when things break
├── phase-8-security-and-hardening/         Isolation limits, non-root, scanning, secrets
├── phase-9-production-and-cicd/            Tagging, registries, multi-arch, CI pipelines
├── phase-10-docker-to-kubernetes-bridge/   Mapping everything onto Kubernetes
└── capstone/                                A full production-grade system, end to end
```

Each phase folder contains numbered concept chapters (one topic per file, self-contained — theory, diagrams, and code together) plus one or more clearly-named project files with fully implemented, runnable code. There are no separate `examples/` or `projects/` subfolders — everything relevant to a topic lives in that topic's own file.

---

## The Learning Path

### [Phase 1: Foundations](./phase-1-foundations/)
What a container actually *is*, mechanically. Namespaces, cgroups, the union filesystem, and the daemon/containerd/runc chain — the five ideas every later phase builds on. Ends with containerizing a real Spring Boot app and verifying every concept against it directly.

### [Phase 2: Images and Dockerfiles](./phase-2-images-and-dockerfiles/)
Dockerfile instructions explained by what they actually change about the resulting layers. Build caching, multi-stage builds, base image selection (JRE vs. JDK vs. distroless), and BuildKit internals. Ends with shrinking a 612MB image to 199MB, measured step by step.

### [Phase 3: Runtime and Processes](./phase-3-runtime-and-processes/)
Why your JVM becoming PID 1 matters, how graceful shutdown actually works, what cgroup memory/CPU limits enforce in practice, and how the JVM detects (and sometimes mis-sizes for) container constraints. Ends with reproducing and fixing a real OOM-kill using measured numbers.

### [Phase 4: Networking](./phase-4-networking/)
Bridge networks, the actual iptables/NAT rules behind port publishing, Docker's embedded DNS, and the difference between `EXPOSE` and `-p`. Ends with a segmented public/internal network lab with live isolation verification.

### [Phase 5: Storage and State](./phase-5-storage-and-state/)
Why the container filesystem is the wrong place for durable data, and the three mechanisms (volumes, bind mounts, tmpfs) that solve it correctly. Database container operation, backup/restore. Ends with a Postgres-backed service proving data survives container *and* database recreation.

### [Phase 6: Docker Compose](./phase-6-docker-compose/)
Compose as a convergence engine over primitives you already understand — not new technology. Health-check-gated startup ordering, secrets handling, profiles and overrides. Ends with a six-service stack combining synchronous and event-driven communication.

### [Phase 7: Debugging and Observability](./phase-7-debugging-and-observability/)
A systematic toolkit: inspection commands, a startup-failure checklist, remote JVM debugging, and the readiness-vs-liveness distinction. Ends with diagnosing four deliberately broken bugs in a real stack.

### [Phase 8: Security and Hardening](./phase-8-security-and-hardening/)
What container isolation does and doesn't protect against, non-root execution, image scanning, an honest look at secrets management, and the named categories of container escape. Ends with a fully hardened, verified production image.

### [Phase 9: Production and CI/CD](./phase-9-production-and-cicd/)
A consolidated production-readiness checklist, tagging and registry strategy, multi-architecture builds, and build-once/promote-everywhere discipline. Ends with a real, working CI pipeline enforcing all of it.

### [Phase 10: Docker to Kubernetes Bridge](./phase-10-docker-to-kubernetes-bridge/)
A direct concept map from everything you've learned onto Kubernetes — the CRI, why "Docker support" removal never changed the underlying runtime, and Compose-to-manifest translation. Ends with migrating the Phase 6 stack onto a real cluster.

### [Capstone: Production-Grade E-Commerce Backend](./capstone/)
Every phase, applied together: five services, synchronous and event-driven communication, hardened and scanned images, a real CI pipeline, and a Kubernetes deployment — built and debugged end to end.

---

## How to Use This Repository

- **Read in order.** Every phase assumes the previous ones. Concepts are cross-referenced explicitly (e.g., Phase 8's hardening reuses the exact image built in Phase 2) rather than repeated.
- **Run the code.** Every project chapter is fully implemented and reproducible — build commands, run commands, expected output, and a debugging walkthrough are all included. Type the commands yourself; don't just read them.
- **Treat diagrams as part of the explanation, not decoration.** Mermaid diagrams are used specifically where a sequence, hierarchy, or state transition is easier to internalize visually than in prose.
- **Chapter length varies deliberately.** Some topics are a page; others run several thousand words with multiple diagrams and code examples. Length reflects the topic's actual complexity, not a template.

## What You'll Be Able to Do When You're Done

- Design efficient, minimal, secure Docker images from first principles
- Debug container issues by reasoning from kernel mechanics, not guesswork
- Correctly size JVM memory and CPU settings against real cgroup constraints
- Reason precisely about container networking, storage, and security trade-offs
- Build and operate a real CI/CD pipeline for containerized services
- Translate any Docker/Compose setup onto Kubernetes with a clear mental model of what changes and what doesn't
- Explain every one of the above confidently in a senior backend engineering interview

---

## Prerequisites

Docker Engine (with Compose and BuildKit, both included by default in modern installations), a JDK 21 for local Maven builds outside containers if desired, and — for Phase 10 and the capstone — a local Kubernetes cluster (minikube or kind) or access to one.