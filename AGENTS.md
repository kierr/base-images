# kierr/base-images

## When to add an image

Add a new image when any of these apply:

- **The runtime requires non-trivial build logic** — Shopify's pshopify Ruby must be compiled from source; there is no pre-built binary or APK. This is the strongest case.
- **You want centralized security scanning and consistent rebuilds** — even when a runtime exists as a Wolfi APK (Go, Node, Bun), maintaining it here means Trivy CRITICAL/HIGH gating runs *before* any consumer pulls, not after. A bad Wolfi package gets caught here first.
- **Multiple consumers share the same runtime** — when two or more repos depend on the same base, a single image avoids each independently building from a different wolfi-base timestamp (different glibc, different openssl, different CVE surface).

Do NOT add an image when:

- **A repo uses a runtime exactly once, with no shared consumers** — `FROM wolfi-base:latest` + `apk add` in that repo's Dockerfile is simpler than maintaining an image nobody else references.
- **The runtime is only used in a build stage** — if the runtime never appears in the final shipped image (e.g., a multi-stage build where the compiler doesn't make it to runtime), the security-scan value is lower and the maintenance cost is the same. Prefer `apk add` in the build stage.

## Why centralized images instead of `apk add` everywhere

Each consumer that runs `FROM wolfi-base:latest` + `apk add go-1.27` independently gets:

- A different base-layer timestamp per build (different glibc, openssl, CVE surface per consumer)
- No vulnerability gating before the image is used — the consumer only discovers CVEs when *their* CI runs Trivy
- No signal when Wolfi ships a broken package — each consumer discovers it independently in their own CI

A centralized image solves all three: one rebuild scans once for all consumers, consistent base layers mean identical CVE surface across the org, and a broken Wolfi package fails here before it fails downstream.

The rebuild cost is negligible — GHA layer caching means unchanged images rebuild in seconds. The hourly polling cadence catches upstream Wolfi security patches with a max 1-hour vulnerability window, which `apk add` in a consumer's CI (which may run daily or on push only) cannot match.

## Image structure

Each image lives in `images/<name>/Dockerfile` with two targets:
- **runtime** — minimal, no compiler, no build tools
- **dev** — adds build toolchain for native extension / module compilation

Consumers should use `dev` for build stages and `runtime` for the final stage.

## Version selection

Pick current LTS, not old or deprecated. Include multiple major versions when both have active consumers. Floating tags (`latest`, `dev`) point to the newest version per runtime.

## CI cadence

Every hour. Wolfi has no push-based update feed, so polling is the only option. GHA layer caching makes unchanged builds essentially free, so the cost of frequent polling is negligible while the vulnerability window stays short.

### Broken Wolfi package response

When a scheduled build fails due to a broken Wolfi package, do not reduce cadence or skip the job. Instead, pin the specific APK version in the Dockerfile (e.g., `openssl-3.0.21-r0` instead of bare `openssl-3.0.21`) until Wolfi ships a fixed version. Remove the pin once the fix lands. This keeps builds green and cadence intact without suppressing the signal.
