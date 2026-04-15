# Terraform Change Checklist

Use this checklist for every Terraform apply to prod. Reference as @checklist.md from SKILL.md.

## Pre-Plan

- [ ] Working directory is clean (`git status`)
- [ ] Correct workspace selected (`terraform workspace show`)
- [ ] Correct var file identified for environment
- [ ] State backend accessible (`terraform state list | head -5`)
- [ ] State backup taken:
  ```bash
  gsutil cp gs://TF_STATE_BUCKET/env:/terraform.tfstate \
    ./backups/tfstate-$(date +%Y%m%d-%H%M%S).json
  ```

## Plan Review

- [ ] `terraform validate` passes with no errors
- [ ] Plan output reviewed line by line — not just the summary
- [ ] **No `-/+` (replace) or `-` (destroy) on any of these**:
  - `google_sql_database_instance`
  - `google_container_cluster` / `google_container_node_pool`
  - `google_storage_bucket` (with data)
  - `google_compute_network` / `google_compute_subnetwork`
  - `google_kms_crypto_key` / `google_kms_key_ring`
  - `google_secret_manager_secret`
- [ ] All new IAM bindings reviewed — no primitive roles
- [ ] No `allUsers` or `allAuthenticatedUsers` in bindings
- [ ] All new resources have required labels (env, team, service, cost-center)
- [ ] `prevent_destroy = true` present on all stateful resources in changeset

## Security Gates

- [ ] No `google_service_account_key` resources added
- [ ] No public CloudSQL instances (`authorized_networks { value = "0.0.0.0/0" }`)
- [ ] No GCS buckets with `predefined_acl = "publicRead"` or public IAM
- [ ] VPC Service Controls not weakened
- [ ] Audit log config not removed or reduced

## Cost Review

- [ ] Cost delta estimated (see SKILL.md Step 5)
- [ ] If delta > $100/mo: approved by team lead in PR comments
- [ ] New resources have appropriate committed use discount opportunity noted
- [ ] Budget alert resource present (or existing alert covers new spend)

## Staging Gate (for prod changes)

- [ ] Same change applied to staging successfully (link to staging apply output)
- [ ] Staging application verified healthy after apply
- [ ] If CloudSQL tier change: query performance verified on staging under load

## Prod Apply

- [ ] Apply window: off-peak hours for changes with potential downtime
- [ ] On-call engineer aware of change
- [ ] Incident channel open and ready (`#incidents` in Slack)
- [ ] Rollback plan documented:
  - Terraform: `terraform apply -var-file=envs/prod.tfvars` with previous plan
  - State manipulation: `terraform state mv` if needed
  - Manual: GCP Console fallback for emergency

## Post-Apply Verification

- [ ] `terraform plan` after apply shows "No changes"
- [ ] Application health checks green
- [ ] Cloud Monitoring dashboard shows no anomalies
- [ ] New resources visible in GCP Console with correct labels
- [ ] State committed to remote backend (auto, but verify)

## High-Risk Change Types — Extra Steps

### CloudSQL Changes
- Run `ANALYZE` on affected tables after instance resize
- Verify connection count returns to baseline after restart
- Check slow query log in Cloud Monitoring for 15 min after change

### GKE Changes
- Verify node pool upgrade doesn't trigger unexpected pod evictions
- Check PodDisruptionBudgets aren't blocking drain
- Watch `kubectl get pods -n prod -w` during node pool operations

### Networking Changes
- Verify no connectivity lost between services
- Check NAT gateway logs for unexpected drops
- Verify VPC firewall rules not blocking expected traffic

### KMS Changes
- NEVER rotate a key without application support for the new key version
- Test decrypt with new version before disabling old version
- Maintain key version history for at least 90 days

---

## Severity Classification

| Change Type | Severity | Requires |
|------------|----------|---------|
| Label / description only | LOW | Standard review |
| Add new resource | LOW-MEDIUM | Cost review |
| Modify non-stateful resource | MEDIUM | Security review |
| CloudSQL tier change | HIGH | Staging gate + off-peak |
| Network change | HIGH | Connectivity testing |
| IAM change | HIGH | Security review |
| GKE cluster change | CRITICAL | Full runbook |
| Destroy stateful resource | CRITICAL | Manual approval + data backup |
