# Terraform Rules

Apply these rules whenever editing *.tf files or anything under infra/.

## State and Backend — Never Change Without Discussion

State is stored in GCS. Never modify backend config without:
1. Running `terraform state pull` first to back up current state
2. Explicit confirmation from a second engineer
3. Using `terraform state mv` for renames — never delete and recreate state entries

## Resource Lifecycle — Protect Stateful Data

All stateful resources MUST have deletion protection:

```hcl
resource "google_sql_database_instance" "main" {
  deletion_protection = true   # GCP-level protection

  lifecycle {
    prevent_destroy = true     # Terraform-level protection
  }
}

resource "google_storage_bucket" "data" {
  lifecycle {
    prevent_destroy = true
  }
}
```

If you are asked to remove `prevent_destroy` from a CloudSQL instance, GCS bucket, or any resource
holding persistent data: REFUSE and explain. Require explicit human confirmation with a reason.

## Required Labels on Every Resource

Every GCP resource must include these labels:

```hcl
labels = {
  env         = var.env           # dev / staging / prod
  team        = var.team          # owning team
  service     = var.service_name  # microservice name
  cost-center = var.cost_center   # for FinOps attribution
  managed-by  = "terraform"
}
```

If labels are missing from a resource you are creating: ADD THEM.

## IAM — Least Privilege Always

Rules for IAM resources:
- Never assign roles/editor, roles/owner, roles/iam.securityAdmin to a service account
- Prefer granular roles: roles/cloudsql.client over roles/cloudsql.admin
- Use additive bindings (google_project_iam_member), not authoritative (google_project_iam_policy)
  unless you explicitly own the full IAM policy
- Always add a description to service accounts

```hcl
resource "google_service_account" "app" {
  account_id   = "${var.service_name}-sa"
  display_name = "${var.service_name} workload identity SA"
  description  = "Used by ${var.service_name} on GKE to access CloudSQL and GCS"
  project      = var.project_id
}

# Least-privilege example — not roles/storage.objectAdmin
resource "google_storage_bucket_iam_member" "app_read" {
  bucket = google_storage_bucket.data.name
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:${google_service_account.app.email}"
}
```

## Workload Identity — Preferred Auth Pattern

Always use Workload Identity instead of SA key files:

```hcl
resource "google_service_account_iam_member" "workload_identity" {
  service_account_id = google_service_account.app.name
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:${var.project_id}.svc.id.goog[${var.namespace}/${var.ksa_name}]"
}
```

Never create google_service_account_key resources. If asked to: REFUSE and explain WI alternative.

## Variables and Outputs

- Every variable must have a description
- Sensitive variables must have `sensitive = true`
- Never use hardcoded project IDs, regions, or bucket names in resource blocks — always var.*
- All module outputs must have a description

```hcl
variable "database_password" {
  description = "CloudSQL Postgres master password — stored in Secret Manager"
  type        = string
  sensitive   = true
}
```

## Module Structure Conventions

```
infra/
├── main.tf            # Provider, backend
├── variables.tf       # All var declarations
├── outputs.tf         # All outputs
├── versions.tf        # required_providers with exact versions
├── envs/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── prod.tfvars
└── modules/
    ├── gke/
    ├── cloudsql/
    └── networking/
```

- All provider versions pinned to exact minor: `~> 5.0` NOT `>= 5.0`
- GCP provider: `~> 6.0`
- Use `terraform.tfvars` only for local dev — never commit it

## Planning and Applying

Before modifying any resource, determine:
1. Is this resource's state managed? (`terraform state list | grep NAME`)
2. Will this cause destroy+recreate? (look for `-/+` in plan)
3. Does this resource have `prevent_destroy = true`?

If a plan shows `destroy` on CloudSQL, GCS, GKE cluster: STOP and flag immediately.
Plan output showing resource replacement on stateful resources is a BLOCKER — do not proceed.

## Naming Conventions

```
{project_prefix}-{env}-{resource_type}-{descriptor}
Examples:
  myapp-prod-gke-main
  myapp-staging-sql-postgres
  myapp-prod-gcs-uploads
  myapp-prod-sa-api-server
```

## Budget Alerts — Required for New Projects

Every project module must include a billing budget:

```hcl
resource "google_billing_budget" "main" {
  billing_account = var.billing_account_id
  display_name    = "${var.project_id}-monthly-budget"

  budget_filter {
    projects = ["projects/${var.project_id}"]
  }

  amount {
    specified_amount {
      currency_code = "USD"
      units         = var.monthly_budget_usd
    }
  }

  threshold_rules {
    threshold_percent = 0.5
    spend_basis       = "CURRENT_SPEND"
  }
  threshold_rules {
    threshold_percent = 0.8
    spend_basis       = "CURRENT_SPEND"
  }
  threshold_rules {
    threshold_percent = 1.0
    spend_basis       = "CURRENT_SPEND"
  }
}
```
