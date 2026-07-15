# interaction-ruler
A counterfactual ruler for measuring whether motion forecasters actually learn multi-agent interaction — not just accuracy.

# Typical SOTA Models Summary
## QCNet


# interaction-ruler

Do trajectory prediction models actually learn *which neighbor matters*,
or do they just get accurate by extrapolating their own motion?

Standard metrics (minADE/minFDE) can't tell the difference — a model that
ignores every neighbor can still score well. This repo provides a
**counterfactual ruler** that measures interaction directly: delete a
neighbor from the model's input and see whether the prediction changes
*more* for neighbors that actually matter.

## The one-line finding

Interaction ability is a property of **architecture / inductive bias**, not
of optimization. In a single training run, accuracy keeps improving while
interaction selectivity saturates within the first few epochs — and even the
current SOTA sits far below a perfectly-selective oracle.

## How the ruler works

1. **Relevance** — for each neighbor, a model-free score for how much it
   *should* influence the focal agent (based on time-headway + lateral
   corridor, not "will it collide").
2. **Perturbation** — remove that neighbor from the observation window only;
   the ground-truth future is never touched.
3. **Score** — *distance-matched AUC*: among neighbor pairs at similar
   distance, how often is the model more sensitive to the relevant one?
   - `0.500` = blind (constant-velocity baseline)
   - `1.000` = perfectly selective (oracle)

Both anchors are verified, so any model's score is placed on a calibrated
scale.

## Status

- [x] Ruler defined and calibrated at both ends
- [x] Passes a 6-scene hand-labeled sanity exam
- [x] QCNet adapter validated (minADE₁ 1.610 vs official 1.69)
- [ ] Second and third model families
- [ ] Long-tail correlation (does interaction-AUC predict tail performance?)

## Repo layout

- `PLAN.md` — the single source of truth for *what we're proving*. Read this first.
- `LOG.md` — every experiment: hypothesis → result → did the claim change?
- `src/` — the ruler and model adapters
- `notebooks/` — execution only; no reasoning lives here

## Data

Argoverse 2 (official validation split, re-preprocessed without filtering).
The train/val/test split is **frozen**; the test set is touched exactly once.
