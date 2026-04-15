---
name: terraform-plan
description: >
  Automatically review and validate Terraform changes. Triggers when user mentions
  "terraform", "tf plan", "infra change", "infra PR", or when *.tf files are modified.
  Reviews plan output for destruction risk, security issues, cost impact, and compliance.
---

# Terraform Plan Review Skill

This skill activates automatically when Terraform changes are discussed or when .tf files
are part of the current context. It can also be invoked explicitly with /project:terraform-plan.

## What This Skill Does

1. Runs `terraform validate` to catch syntax errors early
2. Generates a plan and analyses it for risk
3. Checks security posture using the rules in @../../rules/terraform.md
4. Estimates cost impact of changes
5. Flags destruction risk before any apply is attempted

## Usage

**Automatic**: Mention "terraform plan", "infra change", or share .tf files in context
**Explicit**: `/project:terraform-plan [environment]` where environment is dev/staging/prod

## Execution Steps

### Step 1: Validate

```bash
terraform -chdir=infra validate
```

If validate fails: stop here, show errors, do not proceed to plan.

### Step 2: Plan

```bash
# Select workspace
terraform -chdir=infra workspace select ${ENVIRONMENT:-staging}

# Plan with var file
terraform -chdir=infra plan \
  -var-file=envs/${ENVIRONMENT:-staging}.tfvars \
  -out=tfplan.binary \
  2>&1 | tee tfplan.txt

# Human-readable output
terraform -chdir=infra show tfplan.binary
```

### Step 3: Analyse Plan Output

Parse the plan output for:

**Destruction Risk — check first**
```bash
grep -E "^.*#.*will be (destroyed|replaced)" tfplan.txt
grep -E "^\s+\-/\+" tfplan.txt | head -20
```

If ANY stateful resource (CloudSQL, GCS, GKE cluster, VPC) shows destroy or replace:
- STOP
- Display a CRITICAL warning with the resource name and the field that triggered replacement
- Show how to fix without destroying (lifecycle ignore_changes, state mv, etc.)
- Do NOT suggest running apply

**Changes Summary**
```bash
grep -E "Plan: [0-9]+ to add, [0-9]+ to change, [0-9]+ to destroy" tfplan.txt
```

**Resources Being Changed**
```bash
grep -E "^  # " tfplan.txt | head -30
```

### Step 4: Security Review

Apply rules from @../../rules/terraform.md to changed resources in the plan.
Flag any new resources that violate:
- Missing `prevent_destroy` on stateful resources
- Overly-broad IAM roles
- Public exposure (allUsers, 0.0.0.0/0)
- Missing required labels
- SA key creation

### Step 5: Cost Estimate

For resources in the plan, estimate monthly cost delta using pricing anchors in @../../agents/cost-optimizer.md.

Format:
```
COST IMPACT
+ google_sql_database_instance.replica    +$92/mo  (db-n1-standard-2, HA)
~ google_container_cluster.main           ~$0/mo   (label change only)
─────────────────────────────────────────────────
Total delta: +$92/month (+$1,104/year)
```

### Step 6: Approval Checklist

Output a checklist for the engineer to confirm before running apply:

```
PRE-APPLY CHECKLIST
[ ] No stateful resources being destroyed
[ ] All security findings addressed or accepted with justification
[ ] Cost impact reviewed and approved by team lead (if >$100/mo delta)
[ ] Staging applied and verified before prod
[ ] Runbook link added to PR if complex change
[ ] State backup confirmed: gsutil cp gs://BUCKET/terraform.tfstate ./tfstate.backup

Run apply:
terraform -chdir=infra apply tfplan.binary
```

## Supporting Files

- @../../rules/terraform.md — full Terraform coding rules and security requirements
- @../../agents/terraform-reviewer.md — deep review agent for complex IaC PRs

## Error Recovery

If plan fails with "state lock":
```bash
# Check who holds the lock
terraform -chdir=infra force-unlock LOCK_ID

# Find lock ID from error message or
gsutil cat gs://STATE_BUCKET/terraform.tfstate.lock.info
```

If plan fails with "provider authentication":
```bash
gcloud auth application-default login
# or for CI:
gcloud auth activate-service-account --key-file=$GOOGLE_CREDENTIALS
```
