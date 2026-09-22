# base-images

> "A poor-man's Chainguard"

Chainguard's public images went closed-source, so now I build my own based on [Wolfi OS](https://wolfi.dev/).

## Images

| Image | Variants | Source |
|---|---|---|
| `ghcr.io/kierr/base-images/ruby` | `3.4` (runtime), `3.4-dev` (builder) | [Shopify/ruby](https://github.com/Shopify/ruby) `v3.4.10-pshopify1` via [ruby-definitions](https://github.com/Shopify/ruby-definitions) |
| `ghcr.io/kierr/base-images/ruby` | `4.0` / `latest` (runtime), `4.0-dev` / `dev` (builder) | [Shopify/ruby](https://github.com/Shopify/ruby) `v4.0.6-pshopify2` via [ruby-definitions](https://github.com/Shopify/ruby-definitions) |
| `ghcr.io/kierr/base-images/go` | `1.26` (runtime), `1.26-dev` (builder) | Wolfi APK `go-1.26` |
| `ghcr.io/kierr/base-images/go` | `1.27` / `latest` (runtime), `1.27-dev` / `dev` (builder) | Wolfi APK `go-1.27` |
| `ghcr.io/kierr/base-images/node` | `24` / `latest` (runtime), `24-dev` / `dev` (builder) | Wolfi APK `nodejs-24` (LTS) |
| `ghcr.io/kierr/base-images/bun` | `1.4` / `latest` (runtime), `1.4-dev` / `dev` (builder) | Wolfi APK `bun` |

Each image has two variants:
- **runtime** — minimal image with only runtime dependencies
- **dev** — adds the full build toolchain for compiling native extensions / Go modules / npm packages

I use [Shopify's patched Ruby builds (pshopify)](https://github.com/Shopify/ruby-definitions) instead of vanilla Ruby, for the same reason Shopify does. See [Open Sourcing Shopify's Ruby Builds](https://railsatscale.com/2023-06-16-open-sourcing-shopifys-ruby-builds/) for the full rationale. The ruby-build definitions are copied verbatim from [Shopify/ruby-definitions](https://github.com/Shopify/ruby-definitions).

## Usage

```dockerfile
# Ruby 3.4
FROM ghcr.io/kierr/base-images/ruby:3.4-dev AS build
# ...
FROM ghcr.io/kierr/base-images/ruby:3.4

# Ruby 4.0
FROM ghcr.io/kierr/base-images/ruby:4.0-dev AS build
# ...
FROM ghcr.io/kierr/base-images/ruby:4.0

# Go 1.26
FROM ghcr.io/kierr/base-images/go:1.26-dev AS build
# ...
FROM ghcr.io/kierr/base-images/go:1.26

# Go 1.27
FROM ghcr.io/kierr/base-images/go:1.27-dev AS build
# ...
FROM ghcr.io/kierr/base-images/go:1.27

# Node 24
FROM ghcr.io/kierr/base-images/node:24-dev AS build
# ...
FROM ghcr.io/kierr/base-images/node:24

# Bun
FROM ghcr.io/kierr/base-images/bun:1.4-dev AS build
# ...
FROM ghcr.io/kierr/base-images/bun:1.4
```

## Local build

```bash
# Ruby 3.4 runtime
docker buildx build --load -f images/ruby/Dockerfile --target runtime \
  --build-arg RUBY_VERSION=3.4.10-pshopify1 \
  -t ghcr.io/kierr/base-images/ruby:3.4 --platform linux/amd64 images/ruby

# Ruby 4.0 dev
docker buildx build --load -f images/ruby/Dockerfile --target dev \
  --build-arg RUBY_VERSION=4.0.6-pshopify2 \
  -t ghcr.io/kierr/base-images/ruby:4.0-dev --platform linux/amd64 images/ruby

# Go 1.27 runtime
docker buildx build --load -f images/go/Dockerfile --target runtime \
  --build-arg GO_VERSION=1.27.1 \
  -t ghcr.io/kierr/base-images/go:1.27 --platform linux/amd64 images/go

# Node 24 dev
docker buildx build --load -f images/node/Dockerfile --target dev \
  -t ghcr.io/kierr/base-images/node:24-dev --platform linux/amd64 images/node

# Bun dev
docker buildx build --load -f images/bun/Dockerfile --target dev \
  -t ghcr.io/kierr/base-images/bun:1.4-dev --platform linux/amd64 images/bun
```

## Keeping images updated

CI rebuilds all images every hour against the latest Wolfi packages. This pulls in:
- Wolfi OS security patches (glibc, openssl, etc.)
- Updated runtime patch versions from Wolfi APKs
- Trivy CRITICAL/HIGH vulnerability scan gates every build

Runtime versions are pinned via Dockerfile ARGs — update the ARG and push to main to pick up a new version.

## Adding a new image

1. Create `images/<name>/Dockerfile` with `runtime` and `dev` targets
2. Add a `build-<name>` job in `.github/workflows/ci.yml`
3. Push to main

## License

MIT
