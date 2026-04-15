# Kubernetes Manifest Rules

Apply these rules whenever editing files under k8s/ or any *.yaml/*.yml Kubernetes manifests.

## Mandatory Security Checks — Flag Before Applying

Every Deployment, StatefulSet, DaemonSet MUST have:

```yaml
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000          # non-root UID
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      automountServiceAccountToken: false   # set true only if pod needs K8s API access
      containers:
        - securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
```

If any of these are missing from a manifest you are creating or editing: ADD THEM and explain why.

## Resource Requests and Limits — Always Required

Every container must define both requests AND limits. Never omit either.
Use VPA recommendations if available; otherwise use these starting points:

- Small service: requests cpu:100m memory:128Mi — limits cpu:500m memory:256Mi
- Medium service: requests cpu:250m memory:256Mi — limits cpu:1000m memory:512Mi
- Large service: requests cpu:500m memory:512Mi — limits cpu:2000m memory:1Gi

Never set cpu limit to a value that causes excessive throttling — prefer higher limit or no cpu limit
if the workload is bursty. Always explain tradeoff when removing cpu limit.

## GKE Autopilot Restrictions

These will cause pod rejection — NEVER include:
- hostNetwork: true
- hostPID: true
- hostIPC: true
- privileged: true
- hostPath volumes (use emptyDir or PVC instead)
- Node pools with specific taints (Autopilot manages nodes)
- DaemonSets that require host-level access

If a manifest contains any of these, REMOVE and propose the compliant alternative.

## Health Probes — Always Required

Every container must have both readiness and liveness probes.

```yaml
readinessProbe:
  httpGet:
    path: /health/ready     # separate from liveness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30   # give app time to start
  periodSeconds: 15
  failureThreshold: 3
  timeoutSeconds: 5
```

Rule: initialDelaySeconds for liveness must be >= initialDelaySeconds for readiness.
Rule: /health/ready and /health/live should be separate endpoints.

## Labels — Required on All Resources

Every resource must have these labels:

```yaml
metadata:
  labels:
    app: SERVICE_NAME
    version: IMAGE_TAG        # used by Istio/traffic management
    env: staging              # dev / staging / prod
    team: TEAM_NAME
    managed-by: argocd        # or helm, kustomize
```

## Image Tags — Never Use latest

Always use a specific immutable tag. Preferred format: git-SHORT_SHA.
Always set imagePullPolicy: Always when using a mutable tag (e.g., branch name).
Prefer digest-pinned images for production: image@sha256:...

## HPA — Always Set maxReplicas

Never deploy HPA without maxReplicas. Unbounded scaling = unbounded cost.
Recommended starting values:
- maxReplicas: 10 for non-critical services
- maxReplicas: 20 for critical services (set budget alert alongside)
Always include both CPU and memory metrics if the app is memory-bound.

## PodDisruptionBudget — Required for Production

Every production Deployment with >1 replica must have a PDB:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: SERVICE_NAME-pdb
  namespace: prod
spec:
  minAvailable: 1       # or maxUnavailable: 1 — choose one
  selector:
    matchLabels:
      app: SERVICE_NAME
```

## NetworkPolicy — Principle of Least-Privilege Traffic

Never leave a namespace without NetworkPolicies. Provide a default-deny template:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: NAMESPACE
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Then add explicit allow policies per service.

## Secrets — Never in Manifests

Never put base64-encoded secrets in manifest files or Kustomize secretGenerator.
Always use ExternalSecret referencing GCP Secret Manager:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: SERVICE_NAME-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  target:
    name: SERVICE_NAME-secrets
  data:
    - secretKey: DATABASE_URL
      remoteRef:
        key: projects/PROJECT_ID/secrets/db-url
```

## Kustomize Overlay Conventions

- base/ contains minimal shared config
- overlays/{dev,staging,prod} only override what differs (replicas, image, resources)
- Never duplicate entire manifests in overlays — use patches
- Use strategic merge patches for modifying existing fields
- Use JSON6902 patches for adding fields not present in base
