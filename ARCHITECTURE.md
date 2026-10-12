# Architecture

Notes based on the [Kargo Quickstart](https://docs.kargo.io/quickstart/), adapted
to promote a `product` app from a `dev` pre-production gate out to two production
regions (`east`, `west`).

## Core concepts

- **Freight** — an immutable, uniquely identified set of artifacts (here, a
  container image tag) produced from a `Warehouse`. It is the unit that is
  promoted and verified through the pipeline.
- **Warehouse** — monitors artifact sources (e.g. a container registry) for
  changes and produces new `Freight`. It answers "what changed / what should be
  tested this time?".
- **Stage** — a phase in the delivery pipeline that maps to an Argo CD
  Application and a concrete environment (namespace). A Stage requests Freight
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

1. `Warehouse` polls `public.ecr.aws/nginx/nginx`; a new tag updates the Freight
   set. New `Freight` becomes immediately available to `dev` (direct) and is
   picked up as available if it flows through `dev`.
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

## Possible change to explain, and a real-customer pattern

**The change:** replacing the tutorial's linear `test → uat → prod` with a
**pre-prod `dev` gate that fans out to parallel production regions** (`east`,
`west`).

**Real-customer pattern:** multi-region production delivery. A team can verify
Freight once in a shared pre-production Stage, then promote the *same* Freight to
multiple production regions (e.g., US-East / US-West / EU) rather than
re-building and re-inventing promotion logic per region. Because promotion is
Git-based and metadata is retained, the promotion is auditable and reproducible
across regions.

## Positioning Kargo vs. scripts/homegrown tooling

- **Git is the truth and the audit trail.** Every promotion is a commit to a
  `stage/<stage>` branch; Kargo retains history of *what* (Freight SHA), *how*
  (PromotionTask steps), *when/whom*. Scripts bury this in logs/locally.
- **Declarative, reusable, reviewable.** Pipelines are `Warehouse`/`Stage`/
  `PromotionTask` CRs in Git (self-documenting), so changes go through review —
  unlike a bespoke shell script no one owns.
- **Built-in verification & ordering.** Stages express upstream/downstream
  dependencies and Argo CD health; Freight can't skip a gate. Homegrown tooling
  re-implements promotion gating, retries, and health checks from scratch.
- **UI, RBAC, and self-service.** The Kargo Dashboard gives engineers a promotion
  "push-button" plus a live Freight timeline; RBAC gates who can promote where.
- **Standards vs. bespoke.** Kargo + Argo CD is a lintable, testable, upgradeable
  code path rather than an ad-hoc deployment engine that drifts from your
  GitOps model.