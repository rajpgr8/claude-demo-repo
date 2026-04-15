# CI/CD Pipeline Rules

Apply these rules whenever editing files under .github/workflows/, cloudbuild.yaml,
skaffold.yaml, or any CI/CD pipeline configuration.

## GitHub Actions — Security Requirements

### Never Inline Secrets
Never use ${{ secrets.MY_SECRET }} directly in a `run:` step. Use env: to map:

```yaml
# WRONG — secret exposed in process list
- run: gcloud auth activate-service-account --key-file=${{ secrets.SA_KEY }}

# CORRECT — mapped via env
- name: Deploy
  env:
    SA_KEY: ${{ secrets.SA_KEY }}
  run: echo "$SA_KEY" | gcloud auth activate-service-account --key-file /dev/stdin
```

### Prefer Workload Identity Federation over Service Account Keys

For GCP auth, always use Workload Identity Federation from GitHub Actions:

```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL/providers/PROVIDER
    service_account: deploy-sa@PROJECT_ID.iam.gserviceaccount.com
```

Never use `credentials_json` with a base64 SA key. If you see this pattern: flag it and replace.

### Pin Third-Party Actions to SHA

Never use `uses: actions/checkout@main` or `uses: actions/checkout@v4`.
Always pin to a commit SHA:

```yaml
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

For first-party actions (google-github-actions) and well-known publishers only: v4-style tags are acceptable.

### Permissions — Principle of Least Privilege

Every workflow must declare minimal permissions:

```yaml
permissions:
  contents: read          # default — add others only as needed
  id-token: write         # only for OIDC/WIF auth
  packages: write         # only for GHCR pushes
  pull-requests: write    # only for PR comment actions
```

Never use `permissions: write-all`.

## Pipeline Stage Order — Non-Negotiable

CI pipeline must run in this order — no skipping:

```
lint → test → build → security-scan → push-image → deploy-staging → smoke-test → [manual gate] → deploy-prod
```

1. **lint**: ruff + mypy — fast feedback, fail early
2. **test**: pytest with coverage — fail if <80% coverage
3. **build**: docker buildx multi-arch — produces image artifact
4. **security-scan**: trivy for CVEs, gitleaks for secrets — fail on CRITICAL CVEs
5. **push-image**: push to Artifact Registry — only after all checks pass
6. **deploy-staging**: kustomize + ArgoCD sync
7. **smoke-test**: health check + 2–3 critical API calls against staging
8. **deploy-prod**: requires manual approval via GitHub Environment protection rule

If asked to move deploy-prod before tests or remove security-scan: REFUSE. Explain why.

## Caching — Always Configure for Python

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.12"
    cache: "pip"           # or use uv cache explicitly

# For uv:
- name: Install uv
  uses: astral-sh/setup-uv@v4

- name: Cache uv
  uses: actions/cache@v4
  with:
    path: ~/.cache/uv
    key: uv-${{ runner.os }}-${{ hashFiles('uv.lock') }}
```

Always cache Docker layers too:

```yaml
- uses: docker/build-push-action@v6
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

## Trivy Security Scan — Required in CI

```yaml
- name: Scan image for CVEs
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE }}
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1             # Fail CI on CRITICAL or HIGH CVEs
    ignore-unfixed: true     # Don't fail on CVEs with no fix available
```

Upload results to GitHub Security tab:
```yaml
- uses: github/codeql-action/upload-sarif@v3
  if: always()
  with:
    sarif_file: trivy-results.sarif
```

## Coverage Gate

```yaml
- name: Run tests with coverage
  run: uv run pytest --cov=app --cov-report=xml --cov-fail-under=80

- uses: codecov/codecov-action@v4
  with:
    files: coverage.xml
    fail_ci_if_error: true
```

Never lower coverage threshold below 80% without team discussion and PR comment justification.

## Environment Protection Rules

Production deployments MUST have:
- Required reviewers: minimum 1 (use a team, not individual)
- Wait timer: 0 minutes (reviewer approval is the gate)
- Deployment branch: main only

```yaml
deploy-prod:
  environment: production     # maps to GitHub Environment with protection rules
  needs: [smoke-test]
  if: github.ref == 'refs/heads/main'
```

## Image Tagging Strategy

Always tag with git SHA (immutable) AND environment (mutable for latest tracking):

```bash
IMAGE_BASE="REGION-docker.pkg.dev/PROJECT_ID/REPO/SERVICE"
SHORT_SHA=$(git rev-parse --short HEAD)

docker buildx build \
  --tag "${IMAGE_BASE}:git-${SHORT_SHA}" \
  --tag "${IMAGE_BASE}:staging-latest" \   # only for staging
  --push .
```

Never tag prod with `latest`. Production always uses `git-SHA`.

## Cloud Build vs GitHub Actions

Use GitHub Actions for: PR checks, unit tests, linting, image build/push
Use Cloud Build for: anything that needs private VPC access (CloudSQL migrations,
  internal smoke tests against private endpoints)

If a pipeline step needs to reach private GCP resources: use Cloud Build trigger,
not GitHub Actions (no VPC peering available in GH Actions without self-hosted runners).
