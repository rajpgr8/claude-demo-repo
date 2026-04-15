---
name: incident-runbook
description: >
  Generate or execute incident runbooks for known failure patterns. Triggers when user mentions
  "incident", "outage", "pagerduty", "on-call", "prod is down", "service degraded",
  "rollback", "high error rate", or "SLO breach". Provides structured triage steps,
  diagnosis commands, and stakeholder comms templates.
---

# Incident Runbook Skill

This skill activates on incident-related language and provides structured runbooks for known
failure patterns in this stack. For live incident coordination, delegate to @../../agents/incident-responder.md.

## Quick Decision Tree

When triggered, first ask (or infer from context):

1. **Is this a new incident or post-mortem?**
   - New incident → go to TRIAGE
   - Post-mortem → go to POST-MORTEM TEMPLATE in @runbook-template.md

2. **What is the symptom?**
   - Error rate spike → Runbook: DEPLOY-REGRESSION or DEPENDENCY-DOWN
   - Latency spike → Runbook: DB-SATURATION or RESOURCE-EXHAUSTION
   - Complete unavailability → Runbook: DEPLOY-REGRESSION (start here)
   - Pods crashing → Runbook: OOM or CRASHLOOP
   - Data not processing → Runbook: PUBSUB-BACKLOG

3. **What changed recently?**
   ```bash
   # Last 5 ArgoCD syncs
   argocd app history prod --grpc-web | head -10

   # Last 5 deployments
   kubectl rollout history deployment -n prod

   # Recent Terraform applies (check GCP audit log)
   gcloud logging read \
     'protoPayload.serviceName="deploymentmanager.googleapis.com" OR protoPayload.methodName=~"terraform"' \
     --freshness=2h --limit=10
   ```

## Runbook Index

| ID | Name | Trigger Signals |
|----|------|----------------|
| RB-001 | Deploy Regression | Error spike post-deploy, new pod crashloop |
| RB-002 | OOMKilled Cascade | Multiple OOMKilled, memory charts maxed |
| RB-003 | DB Connection Exhaustion | "too many connections" errors, all pods degraded |
| RB-004 | Pub/Sub Backlog | Processing lag, queue depth growing, no crashes |
| RB-005 | CloudSQL Auth Proxy Failure | DB errors from K8s only, proxy sidecar unhealthy |
| RB-006 | Certificate Expiry | TLS handshake errors, browser cert warnings |
| RB-007 | GCS/Dependency Outage | External service errors, GCP status page incident |
| RB-008 | Node Resource Exhaustion | Pods Pending, node pressure, slow scheduling |
| RB-009 | Secret/Config Missing | Pod fails to start, KeyError or env var missing |
| RB-010 | HPA Not Scaling | Load increasing, replicas at max, latency climbing |

## Immediate Actions by Runbook

### RB-001: Deploy Regression
```bash
# 1. Confirm recent deploy is the cause
kubectl rollout history deployment/SERVICE -n prod | head -5

# 2. Rollback immediately (don't wait to confirm cause)
argocd app rollback prod --grpc-web
# OR
kubectl rollout undo deployment/SERVICE -n prod

# 3. Watch recovery
kubectl rollout status deployment/SERVICE -n prod
kubectl get pods -n prod -w

# 4. Verify error rate dropping (check Cloud Monitoring dashboard)
```
**Resolution**: Rollback. Fix forward in staging. Re-deploy only after root cause found.

### RB-002: OOMKilled Cascade
```bash
# 1. Confirm OOMKilled
kubectl get pods -n prod -o wide
kubectl describe pod FAILED_POD -n prod | grep -A3 "Last State"

# 2. Emergency: increase memory limit
kubectl set resources deployment/SERVICE -n prod \
  --limits=memory=2Gi --requests=memory=512Mi

# 3. Watch pods restart with new limit
kubectl rollout status deployment/SERVICE -n prod

# 4. Identify memory leak (next working hours, not during incident)
# Check Cloud Monitoring: Kubernetes > Workloads > Memory Usage
```
**Resolution**: Raise limit. Investigate leak after stabilisation.

### RB-003: DB Connection Exhaustion
```bash
# 1. Check active connections (from a running pod)
kubectl exec -n prod deploy/SERVICE -- \
  python -c "
import psycopg2, os
conn = psycopg2.connect(os.environ['DATABASE_URL'])
cur = conn.cursor()
cur.execute('SELECT count(*), state FROM pg_stat_activity GROUP BY state')
print(cur.fetchall())
"

# 2. Kill idle connections (emergency)
kubectl exec -n prod deploy/SERVICE -- \
  python -c "
import psycopg2, os
conn = psycopg2.connect(os.environ['DATABASE_URL'])
cur = conn.cursor()
cur.execute(\"SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND query_start < NOW() - INTERVAL '5 minutes'\")
conn.commit()
print('Killed idle connections:', cur.rowcount)
"

# 3. Restart pods to reset connection pools
kubectl rollout restart deployment/SERVICE -n prod
```
**Resolution**: Restart pods. Reduce `pool_size` in SQLAlchemy config. Add PgBouncer long-term.

### RB-004: Pub/Sub Backlog
```bash
# 1. Check subscription backlog
gcloud pubsub subscriptions describe SUBSCRIPTION_NAME \
  --format='table(name,pushConfig.pushEndpoint)'

# Cloud Monitoring metric:
# pubsub.googleapis.com/subscription/num_undelivered_messages

# 2. Scale up consumers
kubectl scale deployment/pubsub-consumer -n prod --replicas=20

# 3. Check for poison pill (message causing nack loop)
gcloud pubsub subscriptions pull SUBSCRIPTION_NAME \
  --limit=5 --format=json | jq '.[].message.data' | base64 -d
```
**Resolution**: Scale consumers. Dead-letter queue for poison messages.

### RB-005: CloudSQL Auth Proxy Failure
```bash
# 1. Check proxy sidecar logs
kubectl logs FAILING_POD -n prod -c cloud-sql-proxy | tail -30

# 2. Check Workload Identity binding is correct
kubectl get pod FAILING_POD -n prod -o jsonpath='{.spec.serviceAccountName}'
# Verify: gcloud iam service-accounts get-iam-policy GSA_EMAIL

# 3. Restart deployment (usually fixes transient proxy issues)
kubectl rollout restart deployment/SERVICE -n prod
```
**Resolution**: Restart. If WI binding wrong: fix annotation on K8s SA.

### RB-006: Certificate Expiry
```bash
# 1. Check cert status
kubectl get certificates -n prod
kubectl describe certificate CERT_NAME -n prod

# 2. Check cert-manager logs
kubectl logs -n cert-manager deploy/cert-manager | tail -30

# 3. Force renewal
kubectl delete certificate CERT_NAME -n prod
# cert-manager will recreate and re-issue automatically

# 4. Verify new cert issued
kubectl get certificates -n prod -w
```
**Resolution**: Delete cert resource — cert-manager auto-reissues. Check ACME rate limits if failed.

### RB-008: Node Resource Exhaustion
```bash
# 1. Check node status
kubectl get nodes
kubectl describe nodes | grep -E "Pressure|Allocatable" | head -30

# 2. Find resource hogs
kubectl top pods --all-namespaces --sort-by=memory | head -20
kubectl top pods --all-namespaces --sort-by=cpu | head -20

# 3. For GKE Autopilot — just scale down over-allocated workloads
# Autopilot provisions new nodes automatically when requests are made

# 4. Evict stuck pods (if DiskPressure)
kubectl get pods --all-namespaces | grep Evicted | \
  awk '{print "kubectl delete pod " $2 " -n " $1}' | bash
```

### RB-009: Missing Secret or ConfigMap
```bash
# 1. Find the missing resource
kubectl get events -n prod | grep "secret\|configmap" | tail -20

# 2. Check ExternalSecret status
kubectl get externalsecrets -n prod
kubectl describe externalsecret SECRET_NAME -n prod

# 3. Force ESO refresh
kubectl annotate externalsecret SECRET_NAME -n prod \
  force-sync=$(date +%s) --overwrite

# 4. Verify secret created
kubectl get secret APP_SECRET_NAME -n prod
```

## Communication Templates

See @runbook-template.md for full post-mortem template and stakeholder comms.

Quick status update (paste into Slack #incidents):
```
[SEV-X UPDATE | HH:MM UTC]
Impact: [current user-facing impact]
Status: INVESTIGATING / MITIGATING / MONITORING / RESOLVED
Action: [what we just did or are doing]
Next update: [HH:MM UTC or "resolved"]
```
