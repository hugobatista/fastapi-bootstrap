# =============================================================================
# Stage 0: uv — pinned version for supply chain security
# =============================================================================
# Pulls the exact uv version specified by UV_VERSION (overridable at build time
# with --build-arg UV_VERSION=x.y.z). This avoids depending on :latest which
# could introduce breaking changes or compromised releases.
ARG UV_VERSION=0.11.18
FROM ghcr.io/astral-sh/uv:${UV_VERSION} AS uv-dist

# =============================================================================
# Stage 1: Builder — export locked deps + build wheel
# =============================================================================
FROM python:3.13-slim AS builder
ENV PIP_ROOT_USER_ACTION=ignore

WORKDIR /app

COPY --from=uv-dist /uv /uvx /usr/local/bin/

# uv.lock pins every transitive dependency — this is the source of truth
# for reproducible builds. The lockfile must be committed.
COPY pyproject.toml uv.lock README.md ./

# Private .whl deps (if any) that aren't on PyPI. Copied into the builder
# but installed in the runtime stage so the builder only needs to export/build.
COPY .dependencies /app/.dependencies

# Source code is needed for the build step (version comes from __about__.py).
# uv export does not need it, but the wheel build does.
COPY src src/

# Export locked runtime deps to requirements.txt (with hashes for pip verify)
# then build the project wheel. --no-dev excludes dev/test/lint groups.
RUN uv export --no-dev --no-emit-project -o requirements.txt \
 && uv build

# =============================================================================
# Stage 2: Runtime — minimal image with locked deps + app
# =============================================================================
FROM python:3.13-slim
ENV PIP_ROOT_USER_ACTION=ignore

LABEL org.opencontainers.image.source=https://github.com/hugobatista/fastapi-bootstrap

# Non-root user for security — all runtime operations run as this user
RUN addgroup --system app && adduser --system --group app

# Install locked deps with --require-hashes so pip verifies every package
# against the checksums recorded by uv export. This prevents MITM or
# registry compromise from injecting malicious code.
COPY --from=builder /app/requirements.txt ./
RUN pip install --no-cache --upgrade pip \
 && pip install --no-cache --require-hashes -r ./requirements.txt

# Install private .whl dependencies (if any)
# These live outside the lockfile since they're local/pinned artifacts.
COPY --from=builder /app/.dependencies /app/.dependencies
RUN if ls -1 /app/.dependencies/*.whl >/dev/null 2>&1; then \
    pip install --no-cache-dir /app/.dependencies/*.whl; \
  fi

# Install the app wheel — no source code in the final image, only the
# installed package. Reduces attack surface and image size.
COPY --from=builder /app/dist/*.whl /tmp/
RUN pip install --no-cache-dir /tmp/*.whl && rm /tmp/*.whl

# Runtime configuration (override via -e at container start)
ENV API_SERVER_API_KEY="devapikey"
ENV TURNSTILE_ENABLED=false

USER app
CMD ["api-server"]
