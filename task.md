# FluxCD → ArgoCD Migration Tasks

## Phase 1 – Bootstrap files
- [x] Read all existing files (complete)
- [x] Modify `bootstrap/01-nkp-infra-manager.yaml` (update SA to argocd-application-controller)
- [x] Rename `bootstrap/00-infra-flux-validation-project.yaml` → update for ArgoCD
- [x] Create `bootstrap/03-argocd-appproject.yaml`
- [x] Create `bootstrap/04-argocd-root-app.yaml`
- [x] Delete `bootstrap/02-flux-gitops-repo.yaml`
- [x] Repurpose `bootstrap/02-flux-kustomize-post-build-cm.yaml`

## Phase 2 – ArgoCD Applications directory
- [x] Create `argocd/applications/kustomization.yaml`
- [x] Create `argocd/applications/app-workspaces.yaml`
- [x] Create `argocd/applications/app-external-secrets.yaml`
- [x] Create `argocd/applications/app-cluster-credentials.yaml`
- [x] Create `argocd/applications/app-clusters.yaml`

## Phase 3 – Kustomize replacements (cluster overlays)
- [x] engineering-cluster: env-values.yaml + update cluster-patch.yaml + update kustomization.yaml
- [x] engineering-cluster-2: env-values.yaml + update cluster-patch.yaml + update kustomization.yaml
- [x] finance-cluster: env-values.yaml + update cluster-patch.yaml + update kustomization.yaml
- [x] it-cluster: env-values.yaml + update cluster-patch.yaml + update kustomization.yaml
- [x] sales-cluster: env-values.yaml + update cluster-patch.yaml + update kustomization.yaml

## Phase 4 – Cleanup
- [x] Update `gitops/kustomization.yaml` (remove flux-kustomizations)
- [x] Update `gitops/workspaces/kustomization.yaml` (add test-workspace)
- [x] Delete `gitops/flux-kustomizations/` directory (4 files)
