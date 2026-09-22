#syntax=docker/dockerfile:1

# Ruby base image built on Wolfi OS with Shopify's patched Ruby (pshopify).
#
# Shopify's Ruby builds are API-compliant with stock Ruby but include backported
# bug fixes and performance improvements from the next major version. See:
#   https://railsatscale.com/2023-06-16-open-sourcing-shopifys-ruby-builds/
#   https://github.com/Shopify/ruby-definitions
#
# Definition files are copied verbatim from Shopify/ruby-definitions to avoid
# drift between the inlined definition and the upstream source of truth.
#
# Build with: docker buildx build --build-arg RUBY_VERSION=4.0.6-pshopify2 ...

ARG WOLFI_BASE=cgr.dev/chainguard/wolfi-base:latest

# ---------------------------------------------------------------------------
# Stage 1: Compile Ruby from Shopify's source using ruby-build
# ---------------------------------------------------------------------------
FROM ${WOLFI_BASE} AS builder

RUN apk add --no-cache \
    build-base gcc glibc-dev linux-headers make pkgconf bison \
    openssl-dev readline-dev zlib-dev yaml-dev libffi-dev gmp-dev \
    git curl bash autoconf rust \
    ruby-3.4

# renovate: datasource=github-releases depName=rbenv/ruby-build
ARG RUBY_BUILD_VERSION=v20260520
RUN git clone --branch "${RUBY_BUILD_VERSION}" --depth 1 \
    https://github.com/rbenv/ruby-build.git /tmp/ruby-build

# renovate: datasource=github-tags depName=Shopify/ruby-definitions versioning=regex:^v?(?<major>\d+)\.(?<minor>\d+)\.(?<patch>\d+)-pshopify(?<build>\d+)$
ARG RUBY_VERSION=3.4.10-pshopify1

# Definition from https://github.com/Shopify/ruby-definitions/blob/main/rubies/${RUBY_VERSION}
# The ruby-build definition is inlined here because ruby-build doesn't support
# loading definitions from a URL or file at build time in a single RUN layer.
# Using printf instead of heredocs because Docker's Dockerfile frontend cannot
# parse shell heredocs inside case/esac — the ;; after the heredoc terminator
# is interpreted as a Dockerfile instruction, not shell syntax.
RUN VERSION="${RUBY_VERSION}" && \
    DEF_FILE="/tmp/ruby-build/share/ruby-build/${VERSION}" && \
    if [ "${VERSION}" = "3.4.10-pshopify1" ]; then \
      printf '%s\n' \
        'install_package "openssl-3.0.21" "https://github.com/openssl/openssl/releases/download/openssl-3.0.21/openssl-3.0.21.tar.gz#617e29af8e421f46649484a4937e48c685e47f46488167c982f88bc4ec1d522f" openssl --if needs_openssl:1.0.2-3.x.x' \
        'install_git "ruby-3.4.10-pshopify1" "https://github.com/Shopify/ruby.git" "v3.4.10-pshopify1" autoconf enable_shared standard' \
        > "${DEF_FILE}"; \
    elif [ "${VERSION}" = "4.0.6-pshopify2" ]; then \
      printf '%s\n' \
        'install_package "openssl-3.0.21" "https://github.com/openssl/openssl/releases/download/openssl-3.0.21/openssl-3.0.21.tar.gz#617e29af8e421f46649484a4937e48c685e47f46488167c982f88bc4ec1d522f" openssl --if needs_openssl:1.1.1-3.x.x' \
        'install_git "ruby-4.0.6-pshopify2" "https://github.com/Shopify/ruby.git" "v4.0.6-pshopify2" autoconf enable_shared standard' \
        > "${DEF_FILE}"; \
    else \
      echo "Unknown RUBY_VERSION: ${VERSION}" && exit 1; \
    fi

# RUBY_BUILD_SKIP_MIRROR=1 avoids hitting cache.ruby-lang.org (pshopify isn't there).
ENV RUBY_BUILD_SKIP_MIRROR=1
RUN /tmp/ruby-build/bin/ruby-build --verbose "${RUBY_VERSION}" /usr/local 2>&1 || { cat /tmp/ruby-build.*.log 2>/dev/null; exit 1; }

# Patch default gems with known CVE fixes that pshopify hasn't picked up yet.
# Each gem update is tracked here; remove when the next pshopify release includes it.
# CVE-2026-80212: resolv gem DoS via uncontrolled memory growth (fixed >= 0.7.2)
# CVE-2026-33210: json gem DoS or information disclosure (fixed >= 2.19.2)
# Using gem install --default to add updated default gems alongside the old
# ones, then removing the old vulnerable gemspecs so Trivy only sees the fixed
# versions.
RUN /usr/local/bin/gem install --no-document --default resolv json && \
    find /usr/local/lib/ruby/gems -path '*/specifications/default/resolv-0.7.*.gemspec' -delete && \
    find /usr/local/lib/ruby/gems -path '*/specifications/default/resolv-0.6.*.gemspec' -delete && \
    find /usr/local/lib/ruby/gems -path '*/specifications/default/json-2.18.*.gemspec' -delete && \
    find /usr/local/lib/ruby/gems -path '*/specifications/default/json-2.17.*.gemspec' -delete

# Strip debug symbols to reduce image size.
RUN find /usr/local -type f \( -name '*.so' -o -name '*.so.*' \) -exec strip --strip-unneeded {} + 2>/dev/null || true && \
    find /usr/local -type f -name 'ruby' -exec strip --strip-unneeded {} + 2>/dev/null || true && \
    rm -rf /usr/local/share/ri /usr/local/share/doc

# ---------------------------------------------------------------------------
# Stage 2: Runtime image — minimal, no build tools
# ---------------------------------------------------------------------------
FROM ${WOLFI_BASE} AS runtime

RUN apk add --no-cache \
    glibc glibc-locale-posix libgcc yaml zlib gmp libffi \
    libcrypto3 libssl3 wolfi-baselayout \
    bash busybox curl openssl ca-certificates tzdata procps file \
    sqlite-libs jemalloc libzstd1 brotli libcurl-openssl4

COPY --from=builder /usr/local /usr/local

# ldconfig so shared libs (libruby.so, jemalloc) are found without LD_LIBRARY_PATH.
RUN ldconfig

ENV GEM_HOME=/usr/local/bundle \
    BUNDLE_SILENCE_ROOT_WARNING=1 \
    LD_PRELOAD=libjemalloc.so.2 \
    MALLOC_ARENA_MAX=2

RUN addgroup -g 1000 ruby && \
    adduser -u 1000 -G ruby -s /bin/bash -D -h /home/ruby ruby && \
    mkdir -p /work /usr/local/bundle && \
    chown ruby:ruby /work /usr/local/bundle /home/ruby

WORKDIR /work
USER ruby

ENTRYPOINT ["/bin/bash"]
CMD ["-l"]

# ---------------------------------------------------------------------------
# Stage 3: Dev image — adds build toolchain for native gem compilation
# ---------------------------------------------------------------------------
FROM runtime AS dev

USER root

RUN apk add --no-cache \
    build-base gcc glibc-dev linux-headers make pkgconf \
    yaml-dev openssl-dev zlib-dev gmp-dev readline-dev libffi-dev sqlite-dev \
    git curl bash apk-tools ca-certificates-bundle \
    wolfi-baselayout glibc-locale-posix busybox sqlite-libs

USER ruby
