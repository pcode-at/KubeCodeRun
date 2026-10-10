# KubeCodeRun API server.
#
# Builds with the classic Docker builder (no BuildKit-only features), so it
# works with `az acr build`. Defaults to the official Python image; pass the
# Docker Hardened Images instead with:
#   --build-arg BUILDER_IMAGE=dhi.io/python:3.13-debian13-dev
#   --build-arg RUNTIME_IMAGE=dhi.io/python:3.13-debian13
# Builder and runtime must ship Python at the same path, because the venv
# symlinks to the builder's interpreter.

ARG BUILDER_IMAGE=python:3.13-slim-trixie
ARG RUNTIME_IMAGE=python:3.13-slim-trixie
ARG BUILD_DATE
ARG VERSION=0.0.0-dev
ARG VCS_REF

################################
# Builder stage - install dependencies with uv
################################
FROM ${BUILDER_IMAGE} AS builder

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Install uv for fast dependency installation
RUN pip install --no-cache-dir uv

ENV UV_COMPILE_BYTECODE=1 UV_LINK_MODE=copy
# Use the system interpreter (don't download Python)
ENV UV_PYTHON_DOWNLOADS=0

# Version for hatch-vcs
ARG VERSION=0.0.0-dev
ENV SETUPTOOLS_SCM_PRETEND_VERSION=${VERSION}

WORKDIR /app

# Install dependencies first (cached layer, only invalidated by lock/metadata changes)
COPY uv.lock pyproject.toml README.md /app/
RUN uv sync --locked --no-install-project --no-dev

# Copy application code and install project
COPY . /app
RUN uv sync --locked --no-dev

# Stage /usr/bin/env and the runtime directories (owned by the non-root user,
# UID 65532) in one tree, so the final stage adds them in a single layer.
# A separate COPY of /usr/bin/env is a no-op when builder and runtime share the
# same base image; the classic builder then commits an empty layer and the
# next COPY fails with "failed to get layer ...: layer does not exist" on
# naive-diff storage drivers (e.g. ACR Tasks). The new /app directories keep
# this layer non-empty.
RUN mkdir -p /rootfs/usr/bin /rootfs/app/logs /rootfs/app/data /rootfs/app/ssl && \
    cp -p /usr/bin/env /rootfs/usr/bin/env && \
    chown -R 65532:65532 /rootfs/app/logs /rootfs/app/data /rootfs/app/ssl

################################
# Final stage - minimal runtime image
################################
FROM ${RUNTIME_IMAGE} AS final

ARG BUILD_DATE
ARG VERSION=0.0.0-dev
ARG VCS_REF

LABEL maintainer="KubeCodeRun Contributors" \
    org.opencontainers.image.title="KubeCodeRun" \
    org.opencontainers.image.description="Secure API for executing code in isolated environments" \
    org.opencontainers.image.version="${VERSION}" \
    org.opencontainers.image.created="${BUILD_DATE}" \
    org.opencontainers.image.revision="${VCS_REF}" \
    org.opencontainers.image.source="https://github.com/aron-muon/KubeCodeRun" \
    org.opencontainers.image.licenses="Apache-2.0"

# Copy /usr/bin/env for execution patterns and the runtime directories
# (ownership 65532 preserved from the builder stage)
COPY --from=builder /rootfs/ /

# Copy the virtual environment from builder
# uv creates .venv with symlinks to the system Python (/usr/local/bin on the
# official image, /opt/python/bin on DHI), so builder and runtime must match
COPY --from=builder --chown=65532:65532 /app/.venv /app/.venv
COPY --from=builder --chown=65532:65532 /app/src /app/src
COPY --from=builder --chown=65532:65532 /app/dashboard /app/dashboard

WORKDIR /app

# Place virtual environment executables at the front of the path
ENV PATH="/app/.venv/bin:/usr/local/bin:/opt/python/bin:/usr/bin:/bin"

# Set environment variables
ENV PYTHONPATH=/app \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Health check
HEALTHCHECK --interval=30s --timeout=15s --start-period=10s --retries=3 \
    CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health', timeout=10).read()"]

# Expose ports
EXPOSE 8000 443

# Run as non-root (UID 65532); the official Python image defaults to root
USER 65532:65532

# Default command - run the API server
CMD ["python", "-m", "src.main"]
