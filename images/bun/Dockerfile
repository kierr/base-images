#syntax=docker/dockerfile:1

# Bun base image built on Wolfi OS.
#
# Provides the Bun runtime with dev and runtime variants. Value is the same as
# other base images: centralized security scanning and consistent rebuilds.
#
# The BUN_VERSION ARG carries the full version (e.g., 1.4.2) for Renovate
# tracking. The Wolfi APK is unversioned (just `bun`), so the ARG is used only
# for documentation and Renovate updates — the installed version is whatever
# Wolfi currently ships.
#
# Build with: docker buildx build --build-arg BUN_VERSION=1.4.2 ...

ARG WOLFI_BASE=cgr.dev/chainguard/wolfi-base:latest

# ---------------------------------------------------------------------------
# Stage 1: Dev image — Bun + build tools
# ---------------------------------------------------------------------------
FROM ${WOLFI_BASE} AS dev

# renovate: datasource=github-releases depName=oven-sh/bun extractVersion=^bun-v(?<version>.+)$
ARG BUN_VERSION=1.4.2

RUN apk add --no-cache \
    bun \
    build-base gcc python3 \
    git curl bash ca-certificates

ENV PATH="/usr/bin:$PATH"

RUN addgroup -g 1000 app && \
    adduser -u 1000 -G app -s /bin/bash -D -h /home/app app && \
    mkdir -p /work && \
    chown app:app /work /home/app

WORKDIR /work
USER app

ENTRYPOINT ["/bin/bash"]
CMD ["-l"]

# ---------------------------------------------------------------------------
# Stage 2: Runtime — minimal
# ---------------------------------------------------------------------------
FROM ${WOLFI_BASE} AS runtime

RUN apk add --no-cache \
    bun \
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
