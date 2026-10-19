# kargo-demo

A [Kargo](https://kargo.io) quickstart environment (completed and then extended)
that promotes a `product` application through a **pre-production `dev` stage
that fans out to two production regions (`east` and `west`)**, tying into the
sibling [argocd-demo](https://github.com/sethmccombs/argocd-demo) Argo CD setup.

This is the Kargo portion of the take-home brief, submitted as a separate Git
repository per the instructions.

## Repo layout

```
kargo-demo/
├── base/                    # product Deployment + Service (image public.ecr.aws/nginx/nginx)
├── stages/
│   ├── dev/                 #  overlay: ns dev
│   ├── east/                #  overlay: ns east + topology.kubernetes.io/region: east
│   └── west/                #  overlay: ns west + topology.kubernetes.io/region: west
├── argocd/
│   └── appset.yaml          # Argo CD ApplicationSet -> product-dev/east/west
│                            #   (project default, targetRevision stage/<stage>, path .)
├── kargo/
│   ├── project.yaml         # Kargo Project `product` (namespace product)
│   ├── warehouse.yaml       # Warehouse -> public.ecr.aws/nginx/nginx (real image)
│   ├── promotion-task.yaml  # reusable promotion process (git-clone -> set-image ->
│   │                        #   kustomize-build -> commit -> push -> argocd-update)
│   ├── stages.yaml          # dev (direct from Warehouse); east & west upstream=dev (fan-out)
│   └── promotion-policy.yaml# per-stage (manual promotion)
├── ARCHITECTURE.md          # Freight/Warehouse/Stage, walkthrough, customer pattern, pitch
└── README.md
```

## The pipeline

`public.ecr.aws/nginx/nginx ──(Warehouse)──▶ dev ──▶ east` (parallel `west`)

- **`dev`** is a pre-production gate that subscribes **directly** to the
  `Warehouse` (image tag changes become new `Freight`).
- **`east` and `west`** are the production regions, each subscribing to **`dev`**
  — Freight must be verified in `dev` before it is available to `east`/`west`.
- This fan-out is the **meaningful change beyond the base Kargo tutorial**, which
  teaches a linear `test → uat → prod` chain.

## Prerequisites (in the live environment)

- Argo CD running (from `argocd-demo`).
- **cert-manager** — GitOps-managed as an Argo CD Application (see `argocd-demo`).
- `kargo` CLI installed.

## Setup (runbook)

1. Install Kargo additively onto the existing cluster (chart 1.12.4, values in
   `kargo/helm-values.yaml`). The admin secrets are passed at install time and
   are not committed. See the comment at the top of that file for the full command:
   `helm install kargo oci://ghcr.io/akuity/kargo-charts/kargo --version 1.12.4 --namespace kargo-system --create-namespace -f kargo/helm-values.yaml --set-string api.adminAccount.passwordHash=... --set-string api.adminAccount.tokenSigningKey=...`
   Kargo is then served at `https://bazzite.tailc1546e.ts.net:8443` via `tailscale serve`.
2. `kargo login <kargo-server>` (admin) and install/download the `kargo` CLI.
3. Apply the Argo CD `ApplicationSet`:
   `kubectl apply -k argocd/`
4. Create the Git credential Secret Kargo uses to push stage branches:
   `kubectl -n product apply -f -` with a Secret labelled
   `kargo.akuity.io/cred-type: git` (repoURL, username, password/PAT).
5. Apply the Kargo resources:
   `kubectl apply -f kargo/project.yaml -f kargo/warehouse.yaml \
     -f kargo/promotion-task.yaml -f kargo/stages.yaml -f kargo/promotion-policy.yaml`
6. Promote Freight `dev → east → west` in the Kargo UI/CLI and confirm Argo CD
   syncs the `stage/<stage>` branches.

## The "beyond the base tutorial" change

The base quickstart uses a linear `test → uat → prod` pipeline. This environment
uses a **pre-production `dev` gate that fans out to two production regions**,
demonstrating how a single verified Freight can be promoted to (and across)
multiple regions — mirroring the `east`/`west` regions used in `argocd-demo`.

## Validation

```bash
kustomize build base
kustomize build stages/dev
kustomize build stages/east
kustomize build stages/west
```