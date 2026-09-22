#syntax=docker/dockerfile:1

# Node.js base image built on Wolfi OS.
#
# Provides Node.js LTS (currently Krypton / v24) with dev and runtime variants.
# Like the Go image, the value is centralized security scanning and consistent
# rebuilds — not build complexity.
#
# The NODE_VERSION ARG carries the major version for Renovate tracking. The
# Wolfi APK uses the same major-only format (nodejs-24).
#
# Build with: docker buildx build --build-arg NODE_VERSION=24 ...

ARG WOLFI_BASE=cgr.dev/chainguard/wolfi-base:latest

# ---------------------------------------------------------------------------
# Stage 1: Dev image — Node.js toolchain + build tools
# ---------------------------------------------------------------------------
FROM ${WOLFI_BASE} AS dev

# renovate: datasource=node-version depName=node
ARG NODE_VERSION=24

RUN apk add --no-cache \
    nodejs-${NODE_VERSION} \
    npm \
    build-base gcc python3 \
    git curl bash ca-certificates

# Node binaries install to /usr/bin via Wolfi APK — no custom PATH needed.
ENV PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

RUN addgroup -g 1000 app && \
    adduser -u 1000 -G app -s /bin/bash -D -h /home/app app && \
    mkdir -p /work && \
    chown app:app /work /home/app

WORKDIR /work
USER app

ENTRYPOINT ["/bin/bash"]
CMD ["-l"]

# ---------------------------------------------------------------------------
# Stage 2: Runtime — minimal, no compiler
# ---------------------------------------------------------------------------
FROM ${WOLFI_BASE} AS runtime

ARG NODE_VERSION=24

RUN apk add --no-cache \
    nodejs-${NODE_VERSION} \
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
