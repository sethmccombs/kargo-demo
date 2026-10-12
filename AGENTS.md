# AGENTS.md — kargo-demo

## Repo
A Kargo quickstart (adapted) that promotes a `product` app through a pre-prod
`dev` gate out to two production regions (`east`, `west`). Integrates with the
sibling `argocd-demo` repo for Argo CD and GitOps-managed cert-manager.

## Git workflow — ALWAYS follow
- NEVER commit or push to `main`. `main` only changes via PR merge.
- Work on `feat/<topic>` / `fix/<topic>` branches; push and open a PR.
- Keep commits small and atomic.

## Validate before committing
```bash
kustomize build base
kustomize build stages/dev
kustomize build stages/east
kustomize build stages/west
kustomize build argocd
# Kargo/stage YAML is applied with kubectl/kargo once the Kargo CRDs exist.
```

## Layout
- `base/`  product manifests (Deployment + Service), image `public.ecr.aws/nginx/nginx`.
- `stages/{dev,east,west}/`  kustomize overlays; `east`/`west` stamp `topology.kubernetes.io/region`.
- `argocd/`  ApplicationSet -> `product-{dev,east,west}` (project `default`,
            `targetRevision: stage/<stage>`, `kargo.akuity.io/authorized-stage: product:<stage>`).
- `kargo/`  Project, Warehouse, PromotionTask, Stages, PromotionPolicy.

## Gotchas
- Argo CD `ApplicationSet` uses `authorized-stage` annotations and points at
  `stage/<stage>` branches; those branches only exist after Kargo promotes.
- Kargo pushes to `stage/*` branches via a Git credential Secret
  (`kargo.akuity.io/cred-type: git`). Do not commit the PAT.
- `main` holds the kustomize *source*; `stage/*` hold the *rendered* manifests.
- cert-manager is installed by Argo CD (argocd-demo) and is a Kargo prerequisite.

## Secrets
- Never commit Git PATs / credentials. Provide repo password out-of-band as a
  Kubernetes Secret in the Kargo project namespace.