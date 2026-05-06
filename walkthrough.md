# FluxCD → ArgoCD Migration Walkthrough

## Summary

The `nkp-flux` repository has been fully migrated from FluxCD to ArgoCD. Zero Flux CRD references remain. The Kustomize base/overlay structure is preserved intact.

---

## Verification Results

| Check | Result |
|---|---|
| Remaining `${...}` Flux substitution syntax | ✅ NONE |
| Remaining `kustomize.toolkit.fluxcd.io` CRD references | ✅ NONE |
| Remaining `dispatch.d2iq.io` GitopsRepository references | ✅ NONE |
| `flux-kustomizations/` directory | ✅ DELETED |

---

## What Changed

### Bootstrap files (`bootstrap/`)

| Old File | New File | Action |
|---|---|---|
| `01-nkp-infra-manager.yaml` | `01-nkp-infra-manager.yaml` | Updated — ClusterRoleBinding now targets `argocd-application-controller` SA in `argocd` namespace instead of the Flux SA |
| `02-flux-gitops-repo.yaml` | *(removed)* | Deleted — FluxCD `GitopsRepository` CRD, no longer needed |
| `02-flux-kustomize-post-build-cm.yaml` | `02-argocd-env-values-reference.yaml` | Repurposed as a reference doc only |
| *(new)* | `03-argocd-appproject.yaml` | ArgoCD `AppProject` (`nkp-infra`) scoped to your repo with NKP/CAPI/ESO resource allowlist |
| *(new)* | `04-argocd-root-app.yaml` | ArgoCD root "App of Apps" pointing to `argocd/applications/` |

### New ArgoCD Applications (`argocd/applications/`)

Replaces the old `gitops/flux-kustomizations/` directory entirely.

| Application | Sync Wave | Path Reconciled | Replaces |
|---|---|---|---|
| `nkp-workspaces` | 0 | `gitops/workspaces` | *(was rendered by root Kustomization)* |
| `nkp-external-secrets` | 1 | `gitops/external-secrets` | `ks-external-secrets.yaml` |
| `nkp-cluster-credentials` | 2 | `gitops/cluster-credentials` | `ks-cluster-credentials.yaml` |
| `nkp-clusters` | 3 | `gitops/clusters` | `ks-clusters.yaml` |

Sync waves enforce the same ordering that `dependsOn` provided in Flux.

### Cluster Overlays — Variable Substitution

Replaced Flux `postBuild.substituteFrom` (ConfigMap injection) with **Kustomize `replacements`** for all 5 clusters.

Each cluster overlay now has:
- **`env-values.yaml`** — a per-cluster `ConfigMap` with 7 environment values to fill in
- **Updated `cluster-patch.yaml`** — Flux `${VAR}` placeholders replaced with literal strings (`CONTROL_PLANE_VIP`, `PRISM_ELEMENT`, etc.)
- **Updated `kustomization.yaml`** — `env-values.yaml` added as a resource + `replacements` blocks

Variables substituted per cluster:

| Variable | Fields Targeted in Cluster Spec |
|---|---|
| `CONTROL_PLANE_VIP` | `spec.controlPlaneEndpoint.host`, `spec.topology...nutanix.controlPlaneEndpoint.host` |
| `LB_IP_START` | `spec.topology...serviceLoadBalancer.configuration.addressRanges.0.start` |
| `LB_IP_END` | `spec.topology...serviceLoadBalancer.configuration.addressRanges.0.end` |
| `PC_URL` | `spec.topology...prismCentralEndpoint.url` |
| `PRISM_ELEMENT` | Control plane + worker `machineDetails.cluster.name` |
| `OS_IMAGE_NAME` | Control plane + worker `machineDetails.image.name` |
| `NETWORK_NAME` | Control plane + worker `machineDetails.subnets.0.name` (where applicable) |

### Other Changes
- `gitops/workspaces/kustomization.yaml` — added `test-workspace`
- `gitops/kustomization.yaml` — removed `flux-kustomizations` reference
- `gitops/flux-kustomizations/` — **directory deleted**

---

## How to Bootstrap

### Step 1 — Fill in cluster values
Edit each `env-values.yaml` with your actual Nutanix environment values:
```
gitops/clusters/overlays/engineering/engineering-cluster/env-values.yaml
gitops/clusters/overlays/engineering/engineering-cluster-2/env-values.yaml
gitops/clusters/overlays/finance/finance-cluster/env-values.yaml
gitops/clusters/overlays/it/it-cluster/env-values.yaml
gitops/clusters/overlays/sales/sales-cluster/env-values.yaml
```

### Step 2 — Push to your repo
```bash
git remote set-url origin https://github.com/rakeshsurapaneni/argocd-nkp
git add -A && git commit -m "feat: migrate from FluxCD to ArgoCD"
git push
```

### Step 3 — Apply bootstrap files (one-time, in order)
```bash
kubectl apply -f bootstrap/00-infra-flux-validation-project.yaml
kubectl apply -f bootstrap/01-nkp-infra-manager.yaml
kubectl apply -f bootstrap/03-argocd-appproject.yaml
kubectl apply -f bootstrap/04-argocd-root-app.yaml
```

### Step 4 — Verify in ArgoCD
ArgoCD will detect the root app and spawn the 4 child Applications. Check:
```bash
kubectl get applications -n argocd
```
Expected: `nkp-infra-apps`, `nkp-workspaces`, `nkp-external-secrets`, `nkp-cluster-credentials`, `nkp-clusters`

> [!IMPORTANT]
> ArgoCD must have access to `https://github.com/rakeshsurapaneni/argocd-nkp`. If the repo is private, add the Git credentials in ArgoCD before applying the root app:
> `argocd repo add https://github.com/rakeshsurapaneni/argocd-nkp --username <user> --password <token>`
