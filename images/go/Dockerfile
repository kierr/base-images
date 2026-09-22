#syntax=docker/dockerfile:1

# Go base image built on Wolfi OS.
#
# While Go is available as a Wolfi APK, maintaining a centralized base image
# provides: security rebuilds with Trivy CRITICAL/HIGH gating every 4 hours,
# consistent base layers across all consumers, and a single place to catch
# broken upstream packages before they reach production.
#
# The GO_VERSION ARG carries the full version (e.g., 1.27.1) for Renovate
# tracking, but the Wolfi APK uses major.minor only (go-1.27). The ARG is
# split into MAJOR.MINOR at install time.
#
# Build with: docker buildx build --build-arg GO_VERSION=1.27.1 ...

ARG WOLFI_BASE=cgr.dev/chainguard/wolfi-base:latest

# ---------------------------------------------------------------------------
# Stage 1: Dev image — Go toolchain + build tools
# ---------------------------------------------------------------------------
FROM ${WOLFI_BASE} AS dev

# renovate: datasource=github-releases depName=golang/go extractVersion=^go(?<version>.+)$
ARG GO_VERSION=1.26.8

RUN GO_MINOR="${GO_VERSION%.*}" && \
    apk add --no-cache \
    go-${GO_MINOR} \
    build-base gcc git curl bash ca-certificates

ENV GOROOT="/usr/lib/go" \
    GOPATH="/go" \
    PATH="/usr/lib/go/bin:/go/bin:$PATH"

RUN addgroup -g 1000 app && \
    adduser -u 1000 -G app -s /bin/bash -D -h /home/app app && \
    mkdir -p /go /work && \
    chown app:app /go /work /home/app

WORKDIR /work
USER app

ENTRYPOINT ["/bin/bash"]
CMD ["-l"]

# ---------------------------------------------------------------------------
# Stage 2: Runtime — minimal, no compiler
# ---------------------------------------------------------------------------
FROM ${WOLFI_BASE} AS runtime

RUN apk add --no-cache \
    libstdc++ \
    glibc glibc-locale-posix libgcc wolfi-baselayout \
    ca-certificates tzdata procps file

RUN addgroup -g 1000 app && \
    adduser -u 1000 -G app -s /bin/bash -D -h /home/app app && \
    mkdir -p /work && \
    chown app:app /work /home/app

WORKDIR /work
USER app

ENTRYPOINT ["/bin/bash"]
CMD ["-l"]
