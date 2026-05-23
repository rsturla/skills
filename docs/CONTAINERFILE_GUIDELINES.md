# Containerfile Guidelines

## Containerfile, Not Dockerfile

The OCI specification defines container images, not "Docker images." Use **Containerfile** as the vendor-neutral name.
The syntax is identical to Dockerfile.

- Podman/Buildah search for `Containerfile` first, fall back to `Dockerfile`
- Docker only recognizes `Dockerfile`
- Use `Containerfile` in all projects. Use `Dockerfile` only when Docker is the sole build tool.
- Use `.containerignore`, not `.dockerignore` (Podman checks `.containerignore` first)

## Build Tool

Use **podman**, never docker:

```bash
podman build -t myapp:latest .
podman run --rm myapp:latest
```

## Multi-Stage Builds

Separate build-time dependencies from runtime. Compilers, build tools, and source code stay in the build stage.

```dockerfile
FROM registry.example.com/ubi9/go-toolset:latest AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /app/server .

FROM registry.example.com/ubi9/ubi-micro:latest
COPY --from=builder /app/server /usr/local/bin/server
USER 1001
ENTRYPOINT ["server"]
```

## Base Images

- Use distroless or UBI-micro images — every package is attack surface
- Pin versions to digests or specific tags, never `latest`
- Prefer Red Hat UBI or Hummingbird images for production

## Security

- **Run as non-root**: `USER 1001` (or named user). Never run production containers as root.
- **No secrets in images**: never `COPY .env` or `ARG PASSWORD=`. Use runtime injection.
- **Scan images**: Grype, Trivy, or Clair in CI. Generate SBOMs at build time.
- **Minimal packages**: install only what's needed. Remove caches in the same layer.

## Layer Optimization

- Chain related `RUN` commands with `&&` — clean up in the same layer
- Order layers by change frequency: base → deps → source → config
- `COPY go.mod go.sum` before `COPY . .` to cache dependency downloads

```dockerfile
# Good — cache-friendly layer ordering
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o /app/server .

# Bad — any source change invalidates dep cache
COPY . .
RUN go mod download && go build -o /app/server .
```

## .containerignore

Always include. Exclude:

```text
.git
.env
*.md
build/
dist/
node_modules/
__pycache__/
.idea/
.vscode/
```

## Labels

Use OCI standard labels:

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/org/repo"
LABEL org.opencontainers.image.description="Service description"
LABEL org.opencontainers.image.version="1.0.0"
```

## Anti-Patterns

- `Dockerfile` when Podman is the build tool
- `FROM ubuntu:latest` or any `latest` tag
- Running as root in production
- Secrets baked into image layers
- Installing dev tools in production images
- Large base images when distroless works
- Missing `.containerignore`
