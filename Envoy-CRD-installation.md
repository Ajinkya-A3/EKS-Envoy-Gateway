# Envoy Gateway — CRD Installation Strategy

## Overview

Envoy Gateway's Helm chart bundles two distinct sets of CRDs:

| CRD Group | Source | Managed By |
|-----------|--------|------------|
| Gateway API (standard/experimental) | `kubernetes-sigs/gateway-api` | Kubernetes SIG Network |
| Envoy Gateway custom resources | `gateway.envoyproxy.io/*` | Envoy Proxy project |

Because **Gateway API CRDs are independently versioned and shared across multiple controllers** (e.g. Kgateway, Stunner, other Gateway API implementations already running in the cluster), we install them **separately and manually** rather than letting the Envoy Gateway chart own them. Letting multiple Helm releases own the same CRDs leads to ownership conflicts, pruning races, and failed upgrades.

---

## Why This Split Exists

### Problem

The default `gateway-crds-helm` chart installs **both** Gateway API CRDs and Envoy-specific CRDs together. If you already have Gateway API CRDs installed (from another controller or a manual install), a second Helm-managed install of the same CRDs causes:

- `Error: rendered manifests contain a resource that already exists` on fresh installs
- ArgoCD prune deleting CRDs owned by a different Helm release
- Version skew if two releases try to upgrade the same CRD to different versions
- CRD ownership conflicts between `gateway-crds-helm` and any other Gateway API consumer

### Solution

Split the install into three independent concerns:

```
Gateway API CRDs       →  kubectl (SSA, unowned by Helm)
Envoy Gateway CRDs     →  gateway-crds-helm (envoyGateway.enabled=true only)
Envoy Gateway itself   →  gateway-helm (skipCrds=true)
```

---

## Gateway API CRDs — Manual Install (Server-Side Apply)

Gateway API CRDs are installed directly via `kubectl` using **server-side apply**. This avoids Helm ownership entirely and is safe to re-apply on upgrades.

```bash
kubectl apply --server-side \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml
```

> **Why `--server-side`?**  
> Standard `kubectl apply` (client-side) tracks ownership via the `last-applied-configuration` annotation. For large CRDs this annotation can exceed annotation size limits, causing `metadata.annotations: Too long` errors. Server-side apply (SSA) uses field managers instead, avoids the annotation bloat, and correctly handles merge conflicts across multiple managers.

This installs the following standard-channel Gateway API CRDs:

- `gatewayclasses.gateway.networking.k8s.io`
- `gateways.gateway.networking.k8s.io`
- `httproutes.gateway.networking.k8s.io`
- `referencegrants.gateway.networking.k8s.io`
- `grpcroutes.gateway.networking.k8s.io`

---

## Envoy Gateway CRDs — Helm (Envoy-specific only)

The `gateway-crds-helm` chart is installed with Gateway API CRDs **disabled** so it only manages Envoy-specific CRDs:

```bash
helm install eg-crds ./gateway-crds-helm \
  --set crds.gatewayAPI.enabled=false \
  --set crds.envoyGateway.enabled=true
```

This installs the following CRDs under `gateway.envoyproxy.io`:

```
gateway.envoyproxy.io_backends.yaml
gateway.envoyproxy.io_backendtrafficpolicies.yaml
gateway.envoyproxy.io_clienttrafficpolicies.yaml
gateway.envoyproxy.io_envoyextensionpolicies.yaml
gateway.envoyproxy.io_envoypatchpolicies.yaml
gateway.envoyproxy.io_envoyproxies.yaml
gateway.envoyproxy.io_httproutefilters.yaml
gateway.envoyproxy.io_securitypolicies.yaml
```

---

## Envoy Gateway — Helm (no CRDs)

The main gateway chart is installed with `--skip-crds` since both CRD groups are already managed above:

```bash
helm install eg ./gateway-helm \
  -n envoy-gateway-system \
  --create-namespace \
  --skip-crds
```

---

## ArgoCD Applications

The same split is expressed as three ArgoCD Applications with sync waves to enforce install order.

### Wave -2 — Gateway API CRDs (out-of-band)

Gateway API CRDs are **not managed by ArgoCD**. They are applied manually or via a bootstrap job before ArgoCD sync begins. See [Manual Install](#gateway-api-crds--manual-install-server-side-apply) above.

### Wave -1 — Envoy Gateway CRDs

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: envoy-gateway-crds
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
spec:
  project: default
  source:
    repoURL: oci://docker.io/envoyproxy
    chart: gateway-crds-helm
    targetRevision: v1.8.0
    helm:
      values: |
        crds:
          gatewayAPI:
            enabled: false       # Gateway API CRDs managed separately
            channel: experimental
          envoyGateway:
            enabled: true        # Only Envoy-specific CRDs
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
      - CreateNamespace=false
```

### Wave 0 — Envoy Gateway

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: envoy-gateway
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: default
  source:
    repoURL: oci://docker.io/envoyproxy
    chart: gateway-helm
    targetRevision: v1.8.0
    helm:
      skipCrds: true             # CRDs managed by envoy-gateway-crds app
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ServerSideApply=true
      - CreateNamespace=true
```

---

## Helm Chart Structure Reference

```
gateway-crds-helm/
├── Chart.lock
├── Chart.yaml
├── charts/
│   └── crds/
│       ├── Chart.yaml
│       └── crds/
│           ├── gatewayapi-crds.yaml          # Disabled via crds.gatewayAPI.enabled=false
│           └── generated/
│               ├── gateway.envoyproxy.io_backends.yaml
│               ├── gateway.envoyproxy.io_backendtrafficpolicies.yaml
│               ├── gateway.envoyproxy.io_clienttrafficpolicies.yaml
│               ├── gateway.envoyproxy.io_envoyextensionpolicies.yaml
│               ├── gateway.envoyproxy.io_envoypatchpolicies.yaml
│               ├── gateway.envoyproxy.io_envoyproxies.yaml
│               ├── gateway.envoyproxy.io_httproutefilters.yaml
│               └── gateway.envoyproxy.io_securitypolicies.yaml

gateway-helm/
├── templates/                                # Controller deployment, RBAC, etc.
└── values.yaml
```

---

## Upgrade Procedure

### Upgrading Gateway API CRDs

```bash
# Always use --server-side to avoid annotation size limits
kubectl apply --server-side \
  -f https://github.com/kubernetes-sigs/gateway-api/releases/download/<NEW_VERSION>/standard-install.yaml
```

### Upgrading Envoy Gateway CRDs + Controller

Update `targetRevision` in both ArgoCD Applications and let sync waves handle ordering, or run:

```bash
helm upgrade eg-crds ./gateway-crds-helm \
  --set crds.gatewayAPI.enabled=false \
  --set crds.envoyGateway.enabled=true

helm upgrade eg ./gateway-helm \
  -n envoy-gateway-system \
  --skip-crds
```

---

## Common Failure Modes

| Symptom | Cause | Fix |
|---------|-------|-----|
| `metadata.annotations: Too long` on CRD apply | Client-side apply hitting annotation size limit | Use `--server-side` flag |
| ArgoCD prunes Gateway API CRDs unexpectedly | Gateway API CRDs owned by `envoy-gateway-crds` app with `prune: true` | Keep Gateway API CRDs out of Helm/ArgoCD ownership; apply manually |
| `resource already exists` on Helm install | Helm trying to create CRDs that already exist without `--skip-crds` | Always use `--skip-crds` on the main `gateway-helm` install |
| Envoy Gateway controller crashes on startup | CRD app in wave -1 hasn't synced before wave 0 starts | Verify sync wave ordering; check ArgoCD Application health before wave 0 |
| `gatewayAPI.enabled=true` overrides manual install | Helm re-installs Gateway API CRDs and takes ownership | Ensure `crds.gatewayAPI.enabled=false` is set on the CRD app |