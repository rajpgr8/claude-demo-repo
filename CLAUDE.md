# Project Context

Python 3.12 FastAPI microservice deployed on GKE Autopilot via ArgoCD.
Uses Terraform (GCP provider) for infra, Skaffold for local dev, GitHub Actions for CI.

## Stack
- **Runtime**: Python 3.12, FastAPI, Pydantic v2, SQLAlchemy 2.0 async
- **Database**: CloudSQL Postgres 15 via Cloud SQL Auth Proxy sidecar
- **Infra**: GKE Autopilot, Workload Identity, Pub/Sub, GCS, BigQuery
- **IaC**: Terraform >= 1.7 — state in GCS bucket, workspaces per env
- **GitOps**: ArgoCD (app-of-apps pattern), kustomize overlays in k8s/overlays/{dev,staging,prod}
- **Observability**: OpenTelemetry → Cloud Trace + Cloud Metrics; structured JSON logs → Cloud Logging
- **Package manager**: uv (not pip/poetry)

## Commands
- `make test` — pytest with coverage (min 80%)
- `make lint` — ruff check + ruff format --check + mypy
- `make build` — docker buildx build (multi-arch: linux/amd64,linux/arm64)
- `skaffold dev` — hot-reload local K8s dev loop (kind cluster)
- `skaffold run -p staging` — deploy to staging
- `kubectl apply -k k8s/overlays/staging` — direct kustomize apply
- `terraform -chdir=infra plan -var-file=envs/staging.tfvars` — plan staging
- `uv run alembic upgrade head` — apply DB migrations
- `uv run alembic downgrade -1` — roll back last migration

## Architecture Decisions
- No direct os.environ — always use app/core/config.py (Pydantic BaseSettings)
- Workload Identity for all GCP service calls — no JSON key files, ever
- All SQL via SQLAlchemy async sessions — no raw cursor usage outside migrations
- Secrets in Secret Manager — injected as env vars by External Secrets Operator
- Every new endpoint needs OTel spans — use @tracer.start_as_current_span()
- No synchronous I/O inside async endpoints — use run_in_executor if unavoidable
- Monorepo: app/ = service, infra/ = terraform, k8s/ = manifests, scripts/ = ops

## Non-Obvious Gotchas
- CloudSQL Auth Proxy: 127.0.0.1:5432 in K8s (sidecar), localhost:5432 locally
- GKE Autopilot does NOT allow hostNetwork:true, privileged:true, or hostPath volumes
- Pub/Sub push subscriptions expect HTTP 200/204 — anything else triggers redelivery
- BigQuery streaming inserts have ~90s eventual consistency — never read-after-write in tests
- Alembic auto-generate misses partial indexes and CHECK constraints — always review output
- structlog processors configured in app/core/logging.py — don't reconfigure elsewhere

## Coding Conventions
- Type hints required on ALL functions including private ones
- Use structlog for logging — never print() or logging.info()
- All new modules define __all__
- Migrations must be reversible — always implement downgrade()
- Error responses use app/core/exceptions.py handlers — no ad-hoc HTTPException in routes

## Team Workflow
- Never push directly to main — always PR with at least 1 reviewer
- Branch naming: feat/TICKET-desc, fix/TICKET-desc, infra/TICKET-desc
- Staging deploys automatically on merge to main
- Prod requires manual ArgoCD sync approval in Slack #deployments
- Tag releases as vMAJOR.MINOR.PATCH
- Conventional Commits: feat:, fix:, chore:, infra:, docs:
