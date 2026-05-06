# Migration: FluxCD → ArgoCD for NKP GitOps

## Background

The `nkp-flux` repo manages NKP (Nutanix Kubernetes Platform) workload cluster lifecycle and workspace management via **FluxCD**. The goal is to replace all FluxCD primitives with equivalent **ArgoCD** primitives while keeping the existing Kustomize base/overlay structure and all NKP-specific Kubernetes resources (CAPI `Cluster`, ESO `ExternalSecret`, Kommander `Workspace`, RBAC, etc.) intact.

---

## Current FluxCD Architecture (as-found)

```
bootstrap/
  00-infra-flux-validation-project.yaml   # NKP Project (Kommander CRD) – bootstrap step
  01-nkp-infra-manager.yaml               # ClusterRole + ClusterRoleBinding for the Flux SA
  02-flux-gitops-repo.yaml                # FluxCD GitopsRepository (D2iQ dispatch CRD)
  02-flux-kustomize-post-build-cm.yaml    # ConfigMap used by Flux postBuild substitution

gitops/
  kustomization.yaml                      # Kustomize root – includes workspaces + flux-kustomizations
  workspaces/                             # Kommander Workspace objects (base + overlays per team)
  external-secrets/                       # ESO SecretStore, ServiceAccount, RoleBinding (base + overlays)
  cluster-credentials/                    # ExternalSecret objects for CAPI cluster credentials
  clusters/                               # CAPI Cluster objects
  flux-kustomizations/                    # FluxCD Kustomization CRDs that reconcile the above three paths
    ks-external-secrets.yaml
    ks-cluster-credentials.yaml
    ks-clusters.yaml
```

**Key FluxCD concepts used:**
| FluxCD concept | File(s) | Notes |
|---|---|---|
| `GitopsRepository` (D2iQ) | `bootstrap/02-flux-gitops-repo.yaml` | Points to this Git repo + triggers root Kustomization |
| `kustomize.toolkit.fluxcd.io/v1 Kustomization` | `flux-kustomizations/*.yaml` | Each drives reconciliation of a sub-path with `dependsOn`, `serviceAccountName`, `prune`, `wait` |
| `postBuild.substituteFrom` (ConfigMap `cluster-values`) | `ks-clusters.yaml` | Replaces `${IP_START}`, `${PRISM_ELEMENT}`, `${OS_IMAGE_NAME}`, `${NETWORK_NAME}` in cluster manifests |
| Flux SA (`infra-flux-validation`) | `01-nkp-infra-manager.yaml` | Bound to ClusterRole with NKP/CAPI/ESO permissions |

---

## Key Design Decisions for ArgoCD

### 1. Variable Substitution (`postBuild.substituteFrom`)
> [!IMPORTANT]
> **This is the most important design question requiring your input.**
>
> Flux's `postBuild.substituteFrom` injects values from a ConfigMap (`cluster-values`) into manifests at sync time — replacing `${IP_START}`, `${PRISM_ELEMENT}`, `${OS_IMAGE_NAME}`, `${NETWORK_NAME}`. **ArgoCD has no native equivalent.**
>
> There are two clean approaches:
>
> **Option A – Bake values directly into each cluster's `cluster-patch.yaml`** *(recommended — already partially done)*
> The cluster-patch files already have cluster-specific values for IP addresses. The `${PRISM_ELEMENT}`, `${OS_IMAGE_NAME}`, and `${NETWORK_NAME}` are the remaining variables. We replace these placeholders with a Kustomize `ConfigMapGenerator` or inline `replacements` block in each cluster's `kustomization.yaml`, or simply **hard-code the values** since they are the same across all clusters. This is the cleanest approach for ArgoCD.
>
> **Option B – ArgoCD ApplicationSet with a Git generator + Helm-style values files**
> More complex, uses ArgoCD's ApplicationSet templating. This is powerful but a heavier restructure.
>
> **Which option do you prefer?** The plan below defaults to **Option A** — using Kustomize `replacements` in each overlay to substitute the environment-specific variables, and an `argocd-cm` ConfigMap to hold the shared values (PRISM_ELEMENT, OS_IMAGE_NAME, NETWORK_NAME).

### 2. ArgoCD `Application` vs `AppProject` placement
ArgoCD `Application` objects will be placed in the `argocd` namespace on the management cluster (same cluster where ArgoCD runs). Each Flux `Kustomization` → one ArgoCD `Application`.

### 3. Dependency ordering (`dependsOn`)
ArgoCD does not have native `dependsOn` between Applications. We handle ordering via:
- A **sync wave annotation** (`argocd.argoproj.io/sync-wave`) on each Application — waves: `0` (workspaces), `1` (external-secrets), `2` (cluster-credentials), `3` (clusters).
- ArgoCD's built-in **sync phases** ensure wave 0 completes before wave 1, etc.

### 4. RBAC / ServiceAccount
The existing `ClusterRole` + `ClusterRoleBinding` in `01-nkp-infra-manager.yaml` can be reused directly — just the ServiceAccount subject changes from the Flux namespace to the ArgoCD namespace (`argocd`). Alternatively, ArgoCD uses its own service accounts (the `argocd-application-controller` SA). We will create an `AppProject` with the appropriate resource allow-lists and bind the existing ClusterRole to `argocd-application-controller`.

### 5. Repository structure

The **Kustomize base/overlay structure does not change at all.** We only:
1. Delete the `flux-kustomizations/` directory (replaced by ArgoCD Applications).
2. Replace `bootstrap/02-flux-gitops-repo.yaml` with ArgoCD `Application` + `AppProject` manifests.
3. Update `bootstrap/01-nkp-infra-manager.yaml` to bind the ClusterRole to the `argocd` ServiceAccount instead of the Flux one.
4. Update `bootstrap/02-flux-kustomize-post-build-cm.yaml` — reuse as an ArgoCD ConfigMap plugin, OR fold values into Kustomize overlays.
5. Add an `argocd/` bootstrap directory with the initial `Application` objects.

---

## Open Questions

> [!IMPORTANT]
> **Q1 – Variable substitution approach:** Do you want to hard-code `PRISM_ELEMENT`, `OS_IMAGE_NAME`, and `NETWORK_NAME` directly in each overlay's `cluster-patch.yaml` (clean, simple), OR use Kustomize `replacements` with a `configmap.yaml` per environment (more structured, still Kustomize-native)?
>
> Note: `IP_START` is already effectively hardcoded per cluster in the existing patch files (e.g. `${IP_START}.137`). Only `PRISM_ELEMENT`, `OS_IMAGE_NAME`, `NETWORK_NAME` are genuinely shared across clusters via the ConfigMap.

> [!IMPORTANT]
> **Q2 – ArgoCD namespace/cluster:** Is ArgoCD already installed on the NKP management cluster? If so, what namespace (`argocd` by default)? The bootstrap Application will be applied there.

> [!IMPORTANT]
> **Q3 – Git repo URL:** The current repo URL is `https://github.com/anishp55/nkp-flux`. Should the ArgoCD Applications point to your fork (e.g. `https://github.com/rakeshsurapaneni/nkp-flux`) or keep the original? Also, what Git credential/SSH secret should be used?

> [!NOTE]
> **Q4 – `test-workspace`:** The `test-workspace` overlay exists in `external-secrets/overlays/test-workspace` and `workspaces/test-workspace/` but is **not included** in the root `gitops/workspaces/kustomization.yaml`. Should it be included in the ArgoCD migration or remain excluded?

---

## Proposed Changes

### Phase 1 – Bootstrap files

#### [MODIFY] [01-nkp-infra-manager.yaml](file:///Users/rakeshsurapaneni/Desktop/NKP/nkp-flux/bootstrap/01-nkp-infra-manager.yaml)
- Change `ClusterRoleBinding` subject from SA `infra-flux-validation` in namespace `infra-flux-validation` → SA `argocd-application-controller` in namespace `argocd`.
- Rename resources to `nkp-argocd-manager-role` / `nkp-argocd-manager-binding`.

#### [DELETE] bootstrap/02-flux-gitops-repo.yaml
Replaced entirely by ArgoCD Application manifests.

#### [MODIFY] bootstrap/02-flux-kustomize-post-build-cm.yaml → `bootstrap/02-argocd-env-values-cm.yaml`
- Keep the ConfigMap (same data: `IP_START`, `PRISM_ELEMENT`, `NETWORK_NAME`, `OS_IMAGE_NAME`) but rename it and add a note that these are cluster-environment defaults.
- Values will be referenced via Kustomize `replacements` in each overlay instead of Flux `postBuild`.

#### [NEW] bootstrap/03-argocd-appproject.yaml
An ArgoCD `AppProject` (`nkp-infra`) that:
- Allows source from this Git repo.
- Allows destination to the management cluster (`in-cluster`) and all workspace namespaces (`engineering`, `finance`, `sales`, `it`, `capx-system`).
- Allows all NKP/CAPI/ESO resource kinds needed.

#### [NEW] bootstrap/04-argocd-root-app.yaml
An ArgoCD `Application` (the "App of Apps") that points to the new `argocd/applications/` directory and bootstraps all child Applications.

---

### Phase 2 – New `argocd/` directory (replaces `flux-kustomizations/`)

#### [NEW] argocd/applications/app-workspaces.yaml
ArgoCD `Application` replacing `ks-clusters.yaml` (workspaces):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nkp-workspaces
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "0"
spec:
  project: nkp-infra
  source:
    repoURL: https://github.com/anishp55/nkp-flux
    targetRevision: main
    path: gitops/workspaces
  destination:
    server: https://kubernetes.default.svc
    namespace: kommander
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

#### [NEW] argocd/applications/app-external-secrets.yaml
ArgoCD `Application` replacing `ks-external-secrets.yaml` (sync-wave: "1").

#### [NEW] argocd/applications/app-cluster-credentials.yaml
ArgoCD `Application` replacing `ks-cluster-credentials.yaml` (sync-wave: "2").

#### [NEW] argocd/applications/app-clusters.yaml
ArgoCD `Application` replacing `ks-clusters.yaml` (sync-wave: "3").

#### [NEW] argocd/applications/kustomization.yaml
Kustomize file listing all application YAMLs.

---

### Phase 3 – Variable substitution in cluster overlays

Each cluster overlay's `kustomization.yaml` gains Kustomize `replacements` that substitute `PRISM_ELEMENT`, `OS_IMAGE_NAME`, `NETWORK_NAME` from a local `env-values.yaml` ConfigMap. The `${IP_START}.*` values will be **hard-coded** directly in each `cluster-patch.yaml` (they already effectively are — only the `${IP_START}` prefix is shared, and each cluster has unique offsets).

#### Files to update (add `replacements` + new `env-values.yaml`):
- `gitops/clusters/overlays/engineering/engineering-cluster/`
- `gitops/clusters/overlays/engineering/engineering-cluster-2/`
- `gitops/clusters/overlays/finance/finance-cluster/`
- `gitops/clusters/overlays/it/it-cluster/`
- `gitops/clusters/overlays/sales/sales-cluster/`

Each overlay gets an `env-values.yaml` with concrete IP ranges and Nutanix settings.

---

### Phase 4 – Cleanup

#### [DELETE] `gitops/flux-kustomizations/` (entire directory)
These three Flux Kustomization CRD files are fully replaced by ArgoCD Application objects.

#### [MODIFY] `gitops/kustomization.yaml`
Remove the `flux-kustomizations` entry (ArgoCD doesn't need it rendered by the root kustomization).

---

## Verification Plan

### Automated Checks
- `kustomize build gitops/clusters/overlays/engineering/engineering-cluster` — verify all `${...}` placeholders are resolved.
- `kustomize build gitops/` — verify root renders cleanly without Flux CRDs.
- `kubectl apply --dry-run=client -f bootstrap/` — verify bootstrap manifests are valid.

### Manual Steps
1. Apply bootstrap files to management cluster: `kubectl apply -f bootstrap/`.
2. Apply the ArgoCD root Application: `kubectl apply -f bootstrap/04-argocd-root-app.yaml`.
3. Verify in ArgoCD UI that all 4 child Applications appear and sync successfully.
4. Verify Kommander `Workspace` objects are created in correct namespaces.
5. Verify CAPI `Cluster` objects are created with correct IP addresses (no `${...}` literals remaining).
