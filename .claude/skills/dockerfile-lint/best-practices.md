# Dockerfile Best Practices Reference

Used by @SKILL.md during Dockerfile review.

---

## CRITICAL Rules — Block Build or PR

### CRIT-1: Never Run as Root
Every production Dockerfile MUST have a non-root USER.

```dockerfile
# WRONG
FROM python:3.12-slim
# No USER — defaults to root

# CORRECT
FROM python:3.12-slim
RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser
USER appuser
```

For GKE Autopilot: also set `runAsNonRoot: true` in SecurityContext — the Dockerfile and K8s config
must both enforce this. Autopilot will reject pods running as root.

### CRIT-2: No Secrets in ENV or ARG

```dockerfile
# WRONG — secret baked into image layer (visible in docker history)
ENV DATABASE_URL=postgres://user:password@host/db
ARG API_KEY=sk-live-abc123

# CORRECT — inject at runtime via K8s Secret / ESO
# In Dockerfile: no secret values
# In K8s Deployment:
#   env:
#     - name: DATABASE_URL
#       valueFrom:
#         secretKeyRef:
#           name: app-secrets
#           key: database-url
```

If ENV or ARG contains a value matching: password, secret, key, token, api — flag CRITICAL.

### CRIT-3: Never Use ADD for URLs

```dockerfile
# WRONG — downloads from arbitrary URL at build time (supply chain risk)
ADD https://example.com/script.sh /tmp/script.sh

# CORRECT — explicit curl with checksum verification
RUN curl -fsSL https://example.com/script.sh -o /tmp/script.sh && \
    echo "EXPECTED_SHA256  /tmp/script.sh" | sha256sum -c && \
    chmod +x /tmp/script.sh
```

---

## HIGH Rules — Fix Before Production

### HIGH-1: Never Use :latest Tag

```dockerfile
# WRONG — non-deterministic, breaks reproducibility
FROM python:latest
FROM python:3.12       # also avoid — minor version may change

# CORRECT — pinned minor version (acceptable)
FROM python:3.12-slim

# BEST — digest pinned (fully reproducible)
FROM python:3.12-slim@sha256:a8b9c2d3e4f5...
```

### HIGH-2: Use Slim or Distroless Base Images

Image size ranking (smallest to largest for Python):
1. `gcr.io/distroless/python3-debian12` — no shell, no package manager, smallest attack surface
2. `python:3.12-slim` — Debian slim, ~130MB, most practical for most workloads
3. `python:3.12-alpine` — Alpine, ~50MB, but musl libc causes issues with some C extensions
4. `python:3.12` — Full Debian, ~1GB — never use for production

```dockerfile
# Recommended for most Python microservices
FROM python:3.12-slim AS base

# For maximum security (no shell — harder to debug but production-hardened)
FROM gcr.io/distroless/python3-debian12:nonroot AS runtime
```

### HIGH-3: Clean Package Manager Cache

```dockerfile
# WRONG — leaves cache in layer (~200MB for apt)
RUN apt-get update && apt-get install -y libpq-dev

# CORRECT — clean in same RUN layer (different layer = cache not freed)
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Alpine — use --no-cache flag
RUN apk add --no-cache libpq-dev
```

### HIGH-4: Multi-Stage Build — Required When Build Tools Present

Single-stage with build deps in prod = security risk + bloated image.

```dockerfile
# ─── Stage 1: Build ────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /app

# Install build dependencies (not needed in runtime)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# Install Python deps into isolated prefix
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# ─── Stage 2: Runtime ──────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

WORKDIR /app

# Copy only installed packages from builder
COPY --from=builder /install /usr/local

# Copy app source
COPY --chown=appuser:appgroup app/ ./app/

# Non-root user
RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --no-create-home appuser
USER appuser

EXPOSE 8080
CMD ["python", "-m", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### HIGH-5: No Package Version Ranges in pip install

```dockerfile
# WRONG — non-deterministic
RUN pip install fastapi uvicorn

# WRONG — range allows unexpected upgrades
RUN pip install "fastapi>=0.100"

# CORRECT — always install from locked requirements
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

Use `pip-compile` or `uv lock` to generate deterministic requirements.txt from pyproject.toml.

---

## MEDIUM Rules — Fix Soon

### MED-1: Always Define HEALTHCHECK

```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:8080/health/live || exit 1
```

For distroless images (no curl): use a custom binary healthcheck or rely on K8s probes only
(omit HEALTHCHECK directive in distroless — document this decision).

### MED-2: Use COPY Not ADD for Local Files

```dockerfile
# WRONG — ADD has implicit tar extraction and URL fetching
ADD ./app /app

# CORRECT — explicit, predictable
COPY ./app /app

# COPY supports --chown (preferred)
COPY --chown=appuser:appgroup ./app /app
```

### MED-3: Minimise Layers — Combine Related RUN Commands

```dockerfile
# WRONG — 4 separate layers
RUN apt-get update
RUN apt-get install -y libpq-dev
RUN pip install -r requirements.txt
RUN rm -rf /var/lib/apt/lists/*

# CORRECT — grouped logically
RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq-dev \
    && rm -rf /var/lib/apt/lists/*
```

### MED-4: Pin apt Package Versions

```dockerfile
# WRONG — installs whatever version is current in apt
RUN apt-get install -y libpq-dev

# CORRECT — pinned version for reproducibility
RUN apt-get install -y libpq-dev=15.5-0+deb12u1
```

---

## LOW / INFO Rules — Best Practice

### LOW-1: .dockerignore Is Required

Must exclude at minimum:
```
.git
.gitignore
.env
.env.*
__pycache__
*.pyc
*.pyo
.venv
venv
.pytest_cache
.mypy_cache
.ruff_cache
dist
build
*.egg-info
tests/
docs/
*.md
Dockerfile*
docker-compose*
.claude/
```

### LOW-2: Explicit EXPOSE Port

```dockerfile
EXPOSE 8080
```

Documents the intended port. Does not actually publish — use `-p` or K8s Service for that.

### LOW-3: WORKDIR Instead of cd in RUN

```dockerfile
# WRONG
RUN cd /app && python setup.py install

# CORRECT
WORKDIR /app
RUN python setup.py install
```

### LOW-4: Label Your Image

```dockerfile
LABEL org.opencontainers.image.source="https://github.com/ORG/REPO"
LABEL org.opencontainers.image.description="SERVICE_NAME API server"
LABEL org.opencontainers.image.licenses="Apache-2.0"
```

### LOW-5: Avoid Installing Unnecessary Debug Tools in Prod

Do not install: curl, wget, vim, nano, htop, strace, tcpdump in production image.
Use ephemeral debug containers (`kubectl debug`) instead:
```bash
kubectl debug -it POD_NAME -n NAMESPACE \
  --image=gcr.io/distroless/base-debian12 \
  --target=app-container
```

---

## Complete Production-Ready Template

```dockerfile
# ─── Build Stage ─────────────────────────────────────────────────────────────
FROM python:3.12-slim AS builder

WORKDIR /build

RUN apt-get update \
    && apt-get install -y --no-install-recommends gcc libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/deps -r requirements.txt

# ─── Runtime Stage ────────────────────────────────────────────────────────────
FROM python:3.12-slim AS runtime

LABEL org.opencontainers.image.source="https://github.com/ORG/REPO"

WORKDIR /app

# Runtime OS dependencies only
RUN apt-get update \
    && apt-get install -y --no-install-recommends libpq5 \
    && rm -rf /var/lib/apt/lists/*

# Non-root user
RUN groupadd --gid 1000 appgroup \
    && useradd --uid 1000 --gid appgroup --no-create-home appuser

# Copy installed packages from build stage
COPY --from=builder /deps /usr/local

# Copy app with correct ownership
COPY --chown=appuser:appgroup app/ ./app/

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health/live')"

CMD ["python", "-m", "uvicorn", "app.main:app", \
     "--host", "0.0.0.0", "--port", "8080", \
     "--workers", "1", "--no-access-log"]
```
