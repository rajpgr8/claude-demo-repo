---
description: Guide a staging deployment — builds image, updates kustomize overlay, syncs ArgoCD
---

## Current Branch and Last Commit
!`git branch --show-current && git log -1 --oneline`

## Uncommitted Changes
!`git status --short`

## Changed Files vs Main
!`git diff --name-status main...HEAD`

## Migration Files Changed?
!`git diff --name-status main...HEAD | grep -E "migrations/versions/" || echo "No migration files changed"`

## Current Staging Pod Status
!`kubectl get pods -n staging -o wide 2>/dev/null || echo "kubectl not configured or staging namespace missing"`

## Current Image Tag Running in Staging
!`kubectl get deployment -n staging -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}' 2>/dev/null || echo "No deployments found"`

## ArgoCD App Health
!`argocd app get staging --grpc-web 2>/dev/null || echo "argocd CLI not configured — check ARGOCD_SERVER env"`

## Recent Deployment History
!`kubectl rollout history deployment -n staging 2>/dev/null | head -15 || echo "No rollout history"`

---

Walk me through deploying to staging. Use the context above and follow these steps:

### Step 1 — Pre-flight Checks
- Is the working tree clean? If not, list what would be missed
- Did any migration files change? If YES: warn that `alembic upgrade head` must run in staging before traffic shifts — provide the kubectl exec command to run it in the running pod
- Is `make lint` and `make test` confirmed green? If not shown, advise running them first
- Are there any blockers to deploying (e.g., broken CI, open CRITICAL security findings)?

### Step 2 — Build and Push Image
Provide the exact commands:
```bash
export SHORT_SHA=$(git rev-parse --short HEAD)
export IMAGE="REGION-docker.pkg.dev/PROJECT_ID/REPO_NAME/SERVICE_NAME:git-${SHORT_SHA}"

docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag "${IMAGE}" \
  --push \
  .
```
Remind: replace REGION, PROJECT_ID, REPO_NAME, SERVICE_NAME with actuals from config.

### Step 3 — Update Kustomize Overlay
```bash
cd k8s/overlays/staging
kustomize edit set image SERVICE_NAME="${IMAGE}"
git add kustomization.yaml
git commit -m "chore: deploy git-${SHORT_SHA} to staging"
git push
```

### Step 4 — ArgoCD Sync
```bash
argocd app sync staging --grpc-web
argocd app wait staging --health --timeout 180 --grpc-web
```
Watch the rollout:
```bash
kubectl rollout status deployment -n staging --timeout=120s
```

### Step 5 — Post-deploy Verification
```bash
# Health check
curl -sf https://staging.YOUR_DOMAIN/health | jq .

# Check pod logs for errors
kubectl logs -n staging -l app=SERVICE_NAME --tail=50 --since=2m

# Verify new image is running
kubectl get pods -n staging -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
```

### Step 6 — Rollback (if needed)
```bash
# Via ArgoCD (preferred — rolls back GitOps state too)
argocd app rollback staging --grpc-web

# Via kubectl (emergency only)
kubectl rollout undo deployment/SERVICE_NAME -n staging
```

### Risk Assessment
Based on what changed, flag:
- HIGH: migrations, auth changes, external API integrations, new env vars required
- MEDIUM: new endpoints, dependency upgrades, config changes
- LOW: bug fixes, logging changes, test additions
