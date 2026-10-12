# Architecture

Notes based on the [Kargo Quickstart](https://docs.kargo.io/quickstart/), adapted
to promote a `product` app from a `dev` pre-production gate out to two production
regions (`east`, `west`).

## Core concepts

- **Freight** — an immutable, uniquely identified set of artifacts (here, a
  container image tag) produced from a `Warehouse`. It is the unit that is
  promoted and verified through the pipeline.
  - Freight can also be manifests like a Helm chart, a git revision or a combination of all three
- **Warehouse** — monitors artifact sources (e.g. a container registry) for
  changes and produces new `Freight`. It answers "what changed / what should be
  tested this time?".
- **Stage** — a phase in the delivery pipeline that maps to an Argo CD
  Application and a concrete environment. A Stage requests Freight
  (from a Warehouse and/or an upstream Stage), and once verified/healthy there,
  downstream Stages can request it.

## The pipeline

```
registry: public.ecr.aws/nginx/nginx
             │  (Warehouse discovers new image tag → Freight)
             ▼
          ┌─────┐
          │ dev │  ← subscribes to the Warehouse directly (pre-prod gate)
          └──┬──┘
       verified ▲
        ┌───────┴────────┐
        ▼                ▼
     ┌──────┐        ┌──────┐
     │ east │        │ west │  ← subscribe to Stage `dev`
     └──────┘        └──────┘
```

- `dev` subscribes directly to the `Warehouse` (`sources.direct`).
- `east` and `west` simulate production regions; each subscribes upstream to
  `dev`, so Freight must be verified in `dev` before it can be promoted onward.
- Promotions are **manual** (per `PromotionPolicy` with no auto-promotion),
  matching the quickstart's drag-and-drop flow.

## How a promotion works (end to end)

1. The `Warehouse` polls `public.ecr.aws/nginx/nginx`; (can be any image)
   - a new tag updates the Freight
      set. New `Freight` becomes immediately available to `dev`
2. Promoting to a Stage runs the reusable `demo-promo-process` `PromotionTask`:
   - `git-clone` the ops repo, opening `stage/<stage>` from `main`,
   - `kustomize-set-image` in `base` to the exact Freight image tag,
   - `kustomize-build` the stage overlay into the `stage/<stage>` branch,
   - `git-commit` + `git-push` the branch (Git is the audit trail),
   - `argocd-update` points the `product-<stage>` Argo CD Application at the new
     commit hash.
3. Argo CD sees the branch revision change and syncs the namespace. Stage health
   is derived from the Argo CD Application's health, which "verifies" the Freight
   for that Stage.
4. Once `dev` is verified, Freight is available to `east`/`west`; promoting there
   repeats the same process into their own namespaces/regions.


## Bonus Notes (more to elaborate in person, just jotting down some notes here)

- ApplicationSet / Fan Out Example - I opted for a list generator in the application set YAML, but also left some comments to stub out a method I've used in the past, to generate apps off a YAML file (or multiple in nested directories) useful for single-tenant clusters
- OSS ArgoCD vs Kargo - 
  - ArgoCD is the deployment tool, while Kargo is the layer on top, allowing promotion across environments (clusters, namespaces, etc)
  - ArgoCD is the "what's deployed" and "is it healthy/synced,etc" - Kargo handles the "what version, what envs, is it verified (whatever that may mean in a given case)"
  - There are some shared functionality - such that ArgoCD with the source hydrator can "render" manifests, while Kargo could also perform that as a step in a given stage
- Rendered Manifests - I've got some (rather hack-y) bash scripts to render and commit helm charts, for CI-based rendering - not in place here, but I've got some demo repos I could show if desired