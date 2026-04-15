---
description: Full security audit — secrets, CVEs, IAM, container hardening, K8s posture
---

## Changed Files on This Branch
!`git diff --name-status main...HEAD 2>/dev/null || git status --short`

## Python Dependency Vulnerabilities
!`uv run pip-audit --format=columns 2>/dev/null || echo "pip-audit not available — run: uv add --dev pip-audit"`

## Hardcoded Secrets Scan (source files)
!`grep -rn --include="*.py" --include="*.yaml" --include="*.yml" --include="*.json" \
  -E "(password|secret|api_key|apikey|token|private_key|access_key)\s*[=:]\s*['\"][^'\"]{8,}" \
  . --exclude-dir=.git --exclude-dir=__pycache__ --exclude-dir=.venv --exclude-dir=node_modules \
  2>/dev/null | grep -v "test_" | grep -v "_test" | head -40 || echo "No obvious hardcoded secrets found"`

## .env Files Present in Repo
!`find . -name ".env" -o -name ".env.*" | grep -v ".git" | grep -v ".venv" 2>/dev/null`

## Dockerfile(s) Content
!`find . -name "Dockerfile*" -not -path "./.git/*" -not -path "./.venv/*" 2>/dev/null | \
  xargs -I{} sh -c 'echo "=== {} ===" && cat {}' 2>/dev/null | head -120`

## Kubernetes Privilege Checks
!`grep -rn --include="*.yaml" --include="*.yml" \
  -E "(privileged:\s*true|hostNetwork:\s*true|hostPID:\s*true|allowPrivilegeEscalation:\s*true|runAsUser:\s*0)" \
  k8s/ 2>/dev/null | head -20 || echo "No privilege escalation issues found in k8s/"`

## Missing SecurityContext on K8s Deployments
!`grep -rL "securityContext" k8s/ 2>/dev/null | grep -E "\.(yaml|yml)$" | head -10 || echo "All manifests have securityContext"`

## Missing Resource Limits on K8s Deployments
!`grep -rL "resources:" k8s/ 2>/dev/null | grep -E "\.(yaml|yml)$" | head -10 || echo "All manifests define resources"`

## Terraform IAM Bindings — Broad Roles Check
!`grep -rn --include="*.tf" -E "(roles/editor|roles/owner|roles/iam.securityAdmin|allUsers|allAuthenticatedUsers)" \
  infra/ 2>/dev/null | grep -v "#" | head -20 || echo "No overly-broad IAM roles found in infra/"`

## Service Account Key Files
!`find . -name "*.json" -not -path "./.git/*" -not -path "./.venv/*" 2>/dev/null | \
  xargs grep -l '"type": "service_account"' 2>/dev/null | head -5 || echo "No SA key files found"`

---

Perform a structured security review using all findings above:

### 1. Secrets & Credentials
Flag any hardcoded values, env files committed to git, SA keys present in repo.
For each: file:line, severity, exact remediation (move to Secret Manager, use ESO, etc.)

### 2. Dependency CVEs
List Critical and High CVEs from pip-audit. For each:
- Package + version
- CVE ID and description
- Fixed version to pin

### 3. Container Security
Review each Dockerfile for:
- Running as root (missing USER directive) — CRITICAL if missing
- Base image: prefer distroless or slim, reject `latest` tag
- COPY vs ADD (ADD can fetch URLs — prefer COPY)
- Secrets passed as ENV or ARG in build stage
- Multi-stage build to reduce attack surface
- .dockerignore present and adequate

### 4. Kubernetes Security Posture
For each manifest check:
- SecurityContext: runAsNonRoot, readOnlyRootFilesystem, allowPrivilegeEscalation: false
- Resource requests AND limits (both CPU and memory)
- NetworkPolicy restricting ingress/egress
- ServiceAccount: automountServiceAccountToken: false if not needed
- Liveness + Readiness probes present
- imagePullPolicy: Always for mutable tags

### 5. GCP IAM & Workload Identity
- Roles assigned: are they least-privilege? Suggest granular alternatives to primitive roles
- Workload Identity bindings correct?
- Any allUsers or allAuthenticatedUsers on GCS/APIs?
- Audit log sinks configured?

### 6. Code-Level Issues
- SQL injection: raw string interpolation in queries
- Command injection: user input passed to subprocess/os.system
- Path traversal: user-controlled file paths
- Mass assignment: Pydantic models exposing internal fields
- Authentication missing on endpoints (check for missing Depends(get_current_user))

### 7. Logging & Data Handling
- PII or sensitive fields logged (email, token, password in structlog calls)
- Secrets in exception messages or stack traces
- Cloud Logging export to external destinations configured safely?

### Summary Table

| # | Finding | Severity | File | Fix |
|---|---------|----------|------|-----|
| 1 | ... | CRITICAL/HIGH/MEDIUM/LOW | file:line | ... |

Overall security posture: PASS / NEEDS WORK / FAIL
