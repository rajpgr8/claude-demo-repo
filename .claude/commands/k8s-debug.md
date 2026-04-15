---
description: Debug a failing or unhealthy pod/deployment. Usage: /project:k8s-debug NAMESPACE
---

## Target Namespace
!`echo "Namespace: ${ARGUMENTS:-default}"`

## All Pods in Namespace
!`kubectl get pods -n ${ARGUMENTS:-default} -o wide 2>/dev/null || echo "Cannot reach cluster — check kubectl context"`

## Non-Running Pods
!`kubectl get pods -n ${ARGUMENTS:-default} --field-selector=status.phase!=Running -o wide 2>/dev/null | head -30`

## Recent Events (newest last)
!`kubectl get events -n ${ARGUMENTS:-default} --sort-by='.lastTimestamp' 2>/dev/null | tail -35`

## Deployments + Rollout Status
!`kubectl get deployments -n ${ARGUMENTS:-default} 2>/dev/null`
!`kubectl rollout status deployment -n ${ARGUMENTS:-default} 2>/dev/null || true`

## HPA Status
!`kubectl get hpa -n ${ARGUMENTS:-default} 2>/dev/null || echo "No HPA in this namespace"`

## PodDisruptionBudgets
!`kubectl get pdb -n ${ARGUMENTS:-default} 2>/dev/null || echo "No PDB in this namespace"`

## Describe First Non-Running Pod
!`FAILED=$(kubectl get pods -n ${ARGUMENTS:-default} --field-selector=status.phase!=Running \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null); \
  [ -n "$FAILED" ] && kubectl describe pod "$FAILED" -n ${ARGUMENTS:-default} || echo "All pods running"`

## Logs: Current + Previous Container
!`FAILED=$(kubectl get pods -n ${ARGUMENTS:-default} --field-selector=status.phase!=Running \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null); \
  if [ -n "$FAILED" ]; then \
    echo "=== CURRENT LOGS ==="; \
    kubectl logs "$FAILED" -n ${ARGUMENTS:-default} --tail=80 2>/dev/null; \
    echo "=== PREVIOUS CONTAINER LOGS ==="; \
    kubectl logs "$FAILED" -n ${ARGUMENTS:-default} --previous --tail=80 2>/dev/null || echo "(no previous logs)"; \
  fi`

## Init Container Logs (if any)
!`FAILED=$(kubectl get pods -n ${ARGUMENTS:-default} --field-selector=status.phase!=Running \
  -o jsonpath='{.items[0].metadata.name}' 2>/dev/null); \
  INIT=$(kubectl get pod "$FAILED" -n ${ARGUMENTS:-default} \
  -o jsonpath='{.spec.initContainers[0].name}' 2>/dev/null); \
  [ -n "$INIT" ] && kubectl logs "$FAILED" -n ${ARGUMENTS:-default} -c "$INIT" --tail=50 || echo "No init containers"`

## Node Resource Conditions
!`kubectl get nodes -o wide 2>/dev/null`
!`kubectl describe nodes 2>/dev/null | grep -A8 "Conditions:" | head -50`

## Node Allocatable vs Requested
!`kubectl describe nodes 2>/dev/null | grep -E "(Allocatable|Requests|Limits)" | head -30`

---

Diagnose the issue. Work through these root causes in priority order:

### 1. OOMKilled
If reason is OOMKilled in describe output:
- Show current memory limit
- Estimate actual usage from logs (look for "out of memory" or OOM killer messages)
- Recommend new limit (1.5x–2x observed peak)
- Show the exact YAML patch: `kubectl patch deployment NAME -n NS --patch '...'`

### 2. CrashLoopBackOff
Identify the crash cause from logs:
- Application startup failure (import error, config missing, DB connection refused)
- Health check passing too soon — readiness probe fires before app is ready
- Missing required env var or Secret (look for KeyError or validation errors in logs)
- Port conflict
For each: show fix with exact YAML or env var addition

### 3. ImagePullBackOff / ErrImagePull
- Wrong image tag (does it exist in Artifact Registry?)
- Missing imagePullSecret
- Artifact Registry permissions — Workload Identity GSA needs roles/artifactregistry.reader
- Command to verify: `gcloud artifacts docker images list REGION-docker.pkg.dev/PROJECT/REPO`

### 4. Pending — Unschedulable
- Insufficient CPU or memory on nodes
- PVC not bound (check `kubectl get pvc -n NS`)
- Node selector / affinity rule has no matching nodes
- GKE Autopilot: resource request exceeds Autopilot node class limits
- Taint/toleration mismatch
For each: show exact fix

### 5. Init Container Failing
- DB not reachable (wrong host, CloudSQL Auth Proxy not running)
- Migration failed (show alembic error, suggest running manually)
- Dependency service not ready (use `nslookup` or `curl` inside a debug pod)

### 6. Liveness / Readiness Probe Failing
- Wrong HTTP path or port
- App takes >initialDelaySeconds to start — increase it
- Show current probe config and recommended values

### 7. Missing Secret or ConfigMap
- Identify the missing resource name from events
- Show `kubectl create secret` or ESO ExternalSecret YAML to fix

### 8. GKE Autopilot Restrictions
- `hostNetwork`, `privileged`, `hostPath` not allowed — show compliant alternative
- Sandbox (gVisor) incompatibility with certain syscalls
- Resource class mismatch (compute-intensive workloads need explicit resource class)

### Final Output
- Root cause (1–2 sentences)
- Exact fix command or YAML
- How to verify: `kubectl get pods -n NS -w` + expected output
