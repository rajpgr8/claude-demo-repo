---
description: Estimate GCP cost impact of infra or K8s changes in the current diff
---

## Changed Terraform Files
!`git diff --name-only main...HEAD | grep -E "\.tf$" || echo "No .tf files changed"`

## Terraform Diff (resources added/changed/deleted)
!`git diff main...HEAD -- '*.tf' | head -400`

## Current Terraform Resources Summary
!`grep -rh "^resource " infra/ 2>/dev/null | sort | uniq | head -60 || echo "No infra/ found"`

## Changed Kubernetes Files
!`git diff --name-only main...HEAD | grep -E "k8s/" || echo "No k8s/ files changed"`

## Kubernetes Resource Changes (CPU/Memory/Replicas)
!`git diff main...HEAD -- 'k8s/' | grep -E "(cpu|memory|replicas|minReplicas|maxReplicas)" | head -60 || echo "No K8s resource changes"`

## Current HPA Config
!`kubectl get hpa --all-namespaces -o yaml 2>/dev/null | grep -E "(minReplicas|maxReplicas|name:)" | head -30 || echo "kubectl not configured"`

## Existing Cost Labels Coverage
!`grep -rn --include="*.tf" "labels" infra/ 2>/dev/null | grep -E "(env|team|service|cost.center)" | head -20 || echo "No cost labels found"`

---

Analyse all changes and produce a GCP cost estimate. Use published GCP pricing (acknowledge these are estimates).

### Resource-by-Resource Breakdown

For each added, modified, or deleted resource:

| Resource Name | Type | Change | SKU / Config | Est. Monthly Cost |
|--------------|------|--------|-------------|------------------|
| ... | google_container_cluster | ADD | Autopilot, us-central1 | $X/mo |
| ... | google_sql_database_instance | MODIFY | db-n1-standard-2 → db-n1-standard-4 | +$Y/mo |
| ... | google_storage_bucket | ADD | Standard, 100GB est. | $Z/mo |

Use these GCP pricing anchors:
- GKE Autopilot: ~$0.10/vCPU-hr, ~$0.011/GB-hr (request-based billing)
- CloudSQL Postgres db-n1-standard-2: ~$100/mo, db-n1-standard-4: ~$190/mo
- Cloud Run: $0.00002400/vCPU-sec, $0.00000250/GB-sec (after free tier)
- GCS Standard: $0.020/GB/mo (us-central1), egress extra
- Pub/Sub: $0.04/GB after 10GB free
- BigQuery: $5/TB queried, $0.02/GB/mo storage
- Cloud Armor: $5/policy/mo + $0.75/million requests
- Cloud Load Balancing: $0.025/hr per forwarding rule

### Kubernetes Cost Impact
For changes in resource requests/replicas:
- Calculate vCPU and memory delta across all replicas
- Multiply by HPA maxReplicas for worst-case
- Show: `delta_cpu * $0.10/hr * 730hr = $X/mo` (GKE Autopilot)

### Top 3 Cost Drivers
1. Most expensive new/changed resource + justification
2. Second biggest driver
3. Third biggest driver

### Optimization Opportunities
- **Committed Use Discounts**: Any always-on VMs or CloudSQL → CUD saves 37–55%
- **Spot VMs / Preemptible**: Batch workloads, non-critical workers → 60–80% cheaper
- **GKE Autopilot vs Standard**: Break-even analysis for current resource profile
- **Storage class**: Is Standard used where Nearline ($0.010/GB) or Coldline ($0.004/GB) would work?
- **Right-sizing**: Any oversized CloudSQL tier or VM family mismatch?
- **Sustained use**: Resources running >25% of month already get automatic SUDs

### Risk Flags
- Resources without `lifecycle { prevent_destroy = true }` on stateful data (CloudSQL, GCS with data)
- Missing cost allocation labels (`env`, `team`, `service`, `cost-center`) — blocks FinOps reporting
- HPA without `maxReplicas` cap — potential bill spike
- Missing GCP Budget Alert — recommend `google_billing_budget` resource
- NAT gateway charges on high-egress workloads (check if Cloud NAT is in diff)

### Cost Delta Summary

| | Monthly |
|-|---------|
| Current estimated spend | $X/mo |
| After this change | $Y/mo |
| **Delta** | **+$Z/mo** |
| Annualized delta | $Z*12/yr |

All figures are estimates. Verify at https://cloud.google.com/products/calculator
Suggest adding `infracost` to CI for automated PR cost diffs.
