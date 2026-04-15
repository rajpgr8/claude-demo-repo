---
name: dockerfile-lint
description: >
  Automatically review Dockerfiles for security hardening, build efficiency, and GCP/GKE
  compatibility. Triggers when a Dockerfile is opened, modified, or when user mentions
  "docker build", "container image", "base image", or "Dockerfile".
---

# Dockerfile Lint & Review Skill

Activates automatically on Dockerfile context or via /project:dockerfile-lint.

## What This Skill Does

Reviews Dockerfiles against security hardening standards, multi-stage build patterns,
GKE Autopilot compatibility, and build efficiency best practices.
References @best-practices.md for the full rule set.

## Execution

When triggered, read the Dockerfile and evaluate every rule in @best-practices.md.

### Lint Steps

**Step 1: Run hadolint (if available)**
```bash
hadolint Dockerfile 2>/dev/null || echo "hadolint not installed — proceeding with manual review"
```

**Step 2: Quick security scan**
```bash
# Check for root user
grep -n "USER" Dockerfile || echo "WARNING: No USER directive — runs as root"

# Check for latest tag
grep -n "FROM.*:latest" Dockerfile && echo "WARNING: Using :latest tag"

# Check for secrets in ENV/ARG
grep -nE "^(ENV|ARG).*(password|secret|key|token|api)" Dockerfile -i

# Check for ADD with URL
grep -n "^ADD http" Dockerfile && echo "WARNING: ADD with URL — use curl/wget in RUN instead"
```

**Step 3: Size analysis**
```bash
# Count RUN layers
grep -c "^RUN" Dockerfile || true

# Check if apt/apk cache is cleaned
grep -A5 "apt-get install\|apk add" Dockerfile | grep -c "rm -rf\|--no-cache" || \
  echo "WARNING: Package manager cache not cleaned"
```

**Step 4: Multi-stage check**
```bash
grep -c "^FROM" Dockerfile
```
If count == 1 and the image contains build tools (gcc, pip, npm, go, cargo): warn about missing multi-stage build.

### Output Format

```
DOCKERFILE REVIEW: path/to/Dockerfile

CRITICAL (fix before build)
  [L12] Running as root — no USER directive
  [L8]  Secret in ENV: ENV API_KEY=hardcoded_value

HIGH (fix before prod)
  [L3]  Base image uses :latest tag
  [L15] pip cache not cleared — adds ~200MB to image

MEDIUM (improve soon)
  [L20] No HEALTHCHECK directive
  [L5]  Using ADD for local file — prefer COPY

LOW / INFO (best practice)
  [ ]   No .dockerignore found
  [ ]   Single-stage build with build deps — consider multi-stage

OPTIMISED DOCKERFILE
[provide corrected Dockerfile inline]
```

## Supporting Reference

See @best-practices.md for the complete rule set with examples and rationale.
