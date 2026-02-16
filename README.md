# helm-repo — GKE Platform Bootstrap Charts

This repository contains Helm charts used to bootstrap the **Big Bang** and **Management** clusters in the GKE three-tier platform architecture.

## Repository Structure

```
helm-repo/
├── charts/
│   ├── argocd/                    # Wrapper → upstream argo-cd chart
│   ├── cert-manager/              # Wrapper → upstream cert-manager chart
│   ├── venafi/                    # Wrapper → Venafi enhanced issuer (enterprise)
│   ├── capi-operator/             # Wrapper → upstream CAPI Operator chart
│   ├── capg-provider/             # Custom templates (InfrastructureProvider CR)
│   └── cluster-resource-sets/     # Custom templates (CRS for bootstrapping)
├── .gitignore
└── README.md
```

## Chart Types

| Chart | Type | Source |
|-------|------|--------|
| `argocd` | **Wrapper** (dependency) | `https://argoproj.github.io/argo-helm` |
| `cert-manager` | **Wrapper** (dependency) | `https://charts.jetstack.io` |
| `venafi` | **Wrapper** (dependency) | `oci://private-registry.venafi.cloud/charts` |
| `capi-operator` | **Wrapper** (dependency) | `https://kubernetes-sigs.github.io/cluster-api-operator` |
| `capg-provider` | **Custom templates** | Your own — deploys CAPG InfrastructureProvider CR |
| `cluster-resource-sets` | **Custom templates** | Your own — CRS for mgmt/workload cluster bootstrap |

### Wrapper Chart = Dependency on upstream + your values overrides

You **do not** write templates for wrapper charts. You declare the upstream chart as a
`dependency` in `Chart.yaml`, then provide your overrides in `values.yaml`.

### Custom Template Chart = Your own Kubernetes manifests

For `capg-provider` and `cluster-resource-sets`, the `templates/` directory contains
actual Kubernetes YAML that Helm will render and apply.

## How to Use

### Initial setup (one-time per chart)
```bash
cd charts/argocd
helm dependency update    # Downloads upstream .tgz + creates Chart.lock
cd ../cert-manager
helm dependency update
cd ../capi-operator
helm dependency update
# venafi uses OCI — helm dependency update handles this too
cd ../venafi
helm dependency update
```

### Install on Big Bang cluster
```bash
# 1. cert-manager (required first — CAPI Operator depends on it)
helm install cert-manager charts/cert-manager \
  -n cert-manager --create-namespace --wait

# 2. ArgoCD
helm install argocd charts/argocd \
  -n argocd --create-namespace \
  -f charts/argocd/values-bigbang.yaml --wait

# 3. CAPI Operator (installs Core + Bootstrap + ControlPlane providers)
helm install capi-operator charts/capi-operator \
  -n capi-operator-system --create-namespace --wait --timeout 90s

# 4. CAPG Infrastructure Provider
helm install capg-provider charts/capg-provider \
  -n capg-system --create-namespace --wait

# 5. ClusterResourceSets
helm install crs charts/cluster-resource-sets \
  -n capi-system --wait
```

### Install on Management cluster (via CRS or manually)
```bash
helm install cert-manager charts/cert-manager \
  -n cert-manager --create-namespace --wait

helm install argocd charts/argocd \
  -n argocd --create-namespace \
  -f charts/argocd/values-mgmt.yaml --wait

helm install capi-operator charts/capi-operator \
  -n capi-operator-system --create-namespace --wait --timeout 90s

helm install capg-provider charts/capg-provider \
  -n capg-system --create-namespace --wait

# Venafi (enterprise cert management)
helm install venafi charts/venafi \
  -n venafi-system --create-namespace --wait
```
