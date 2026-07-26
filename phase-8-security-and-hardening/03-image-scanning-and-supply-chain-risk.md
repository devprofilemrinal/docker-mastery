# Image Scanning and Supply Chain Risk

Every image you build depends on a base image, and that base image depends on OS packages and libraries you didn't write and don't directly control. This chapter covers scanning images for known vulnerabilities (CVEs) in that dependency chain, and the broader supply-chain concerns — verifying image provenance, pinning digests — that scanning alone doesn't address.

---

## What a Vulnerability Scanner Actually Checks

An image scanner (Trivy, Grype, and Docker Scout are common choices) inspects an image's layers, identifies every installed OS package and language-level dependency (JAR files on the classpath, in your case) by name and version, and cross-references each against public vulnerability databases (the NVD, and language/ecosystem-specific advisory databases).

```bash
# Trivy — a widely used, straightforward open-source scanner:
trivy image demo-service:1.0
```

```text
demo-service:1.0 (debian 12.5)
=================================
Total: 14 (LOW: 8, MEDIUM: 4, HIGH: 2, CRITICAL: 0)

┌───────────┬────────────────┬──────────┬───────────────────┬───────────────┐
│  Library  │ Vulnerability  │ Severity │ Installed Version │ Fixed Version │
├───────────┼────────────────┼──────────┼───────────────────┼───────────────┤
│ libssl3   │ CVE-2024-XXXXX │ HIGH     │ 3.0.11-1          │ 3.0.13-1      │
└───────────┴────────────────┴──────────┴───────────────────┴───────────────┘
```

Every one of those findings traces to something a *base image maintainer* shipped — not your application code. This is the essential mental shift this chapter asks for: **most of your image's vulnerability surface comes from choices upstream of your Dockerfile**, which is exactly why base image choice (Phase 2, Chapter 4) is a security decision, not merely a size/convenience one.

---

## Why Smaller Base Images Reduce Scan Findings, Concretely

This directly connects back to Phase 2's multi-stage/distroless discussion: fewer installed packages means fewer things that can have a known CVE against them.

```bash
trivy image demo-service:v1-baseline   # JDK-based, single-stage (Phase 2 project baseline)
# Total: 47 vulnerabilities

trivy image demo-service:distroless
# Total: 3 vulnerabilities
```

This is not a coincidence — it's the direct, measurable security payoff of the exact image-size optimizations from Phase 2, now viewed through a different lens. A smaller image isn't just faster to pull; it's a genuinely smaller, more auditable attack surface.

---

## Scanning as a CI Gate

The real value of scanning comes from making it a **blocking step** in your build pipeline (built out fully in Phase 9), not a manual, occasional check:

```bash
# Fail the build on any HIGH or CRITICAL finding:
trivy image --exit-code 1 --severity HIGH,CRITICAL demo-service:1.0
echo $?
# 1 if HIGH/CRITICAL findings exist — a CI pipeline can use this
# exit code to block a merge or block pushing the image further.
```

---

## What Scanning Does *Not* Cover: Provenance and Tampering

A scanner tells you about **known** vulnerabilities in **what's actually in the image** — it says nothing about whether the image you're about to run is genuinely the one your CI pipeline built, versus a tampered or substituted image from a compromised registry step.

### Pin by Digest, Not Just Tag

Recall from Phase 1 Chapter 3: a tag like `myorg/order-service:1.4.2` can, in principle, be repointed to different underlying bytes (deliberately or via a compromised registry). A **digest** cannot:

```bash
# Tag: mutable, could theoretically point to different content over time
docker pull myorg/order-service:1.4.2

# Digest: immutable — this exact reference always resolves to the
# exact same bytes, or fails outright; nothing can silently substitute:
docker pull myorg/order-service@sha256:a1b2c3d4e5f6...
```

For anything genuinely security-sensitive (a production deployment pipeline, Phase 9), pinning by digest — not just a version tag — is the stronger guarantee, closing exactly the substitution gap a scanner alone doesn't address.

### Base Image Provenance

Prefer official, actively-maintained base images (`eclipse-temurin`, the official `postgres`, `nginx`) over unofficial or unmaintained third-party images with a similar name — supply-chain compromises have historically targeted popular but under-scrutinized community images specifically because they're an easy way to reach many downstream consumers at once.

---

## Common Misconceptions

- **"A clean scan result means the image is fully secure."** It means no *known, currently-catalogued* vulnerabilities were found in what the scanner checked — new vulnerabilities are discovered constantly, and scanning is a point-in-time snapshot, not a permanent guarantee; rescanning regularly (not just once at build time) matters for anything long-lived.
- **"Most vulnerabilities in a typical scan are from my own application code."** Overwhelmingly, they trace to base image OS packages and third-party dependencies — which is exactly why base image choice and dependency hygiene matter as much as your own code quality.
- **"Pinning a version tag is equivalent to pinning a digest."** A tag can be repointed; a digest is cryptographically tied to specific, immutable content — a meaningfully stronger guarantee for anything security-sensitive.

---

## What's Next

**Next:** [`04-secrets-management-in-containers.md`](./04-secrets-management-in-containers.md)