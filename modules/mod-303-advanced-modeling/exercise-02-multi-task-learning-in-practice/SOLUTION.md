# exercise-02 — Multi-Task Learning in Practice (Reference Solution)

Paired exercise: `lessons/mod-303-advanced-modeling/exercises/exercise-02-multi-task-learning-in-practice.md`.

## 1. Solution overview

This exercise produces `mtl-design.md` — a design document for a multi-task learning (MTL) system that a senior ML engineer can defend in an architecture review. The artefact has four parts:

1. A **task-set justification** that says, task by task, why these tasks belong in one model and what would make us split them back out.
2. An **architecture choice** — hard-sharing vs. soft-sharing vs. task-specific heads vs. mixture-of-experts — with the reasoning tied to the task relationships.
3. A **loss-weighting plan** — equal weights, uncertainty-weighted (Kendall et al.), GradNorm, Pareto-MTL, or a staged schedule — chosen against the measured task-loss scales and the deployment constraints.
4. A **negative-transfer monitoring plan** — how we would detect that adding task B to the shared model is hurting task A, and what the rollback looks like.

The reference solution below walks through the four parts on one illustrative task set, labels it clearly as illustrative, and gives the grading rubric that applies across every learner's design.

MTL is a decision, not a default. The point of this artefact is to make the decision explicit *before* training, so that when a joint model ships and task B regresses, the team can tell whether it is a negative-transfer problem (predicted and monitored) or a surprise (which means the design document missed something).

## 2. Implementation (worked answer)

### 2.1 Task-set justification

For each candidate task set, record:

- **Each task's label definition** and the **loss function** used in the single-task baseline. Different losses (logistic, MSE, ranking, token-level cross-entropy) do not share gradients cleanly.
- **The task relationship type.** Four useful categories:
  - *Shared-input, shared-structure* (e.g., predicting age and gender from the same face image). Highest MTL upside.
  - *Shared-input, correlated-output* (e.g., click probability and dwell-time regression on the same impression). Moderate upside, high risk of label-leakage if one label is derived from the other.
  - *Shared-input, independent-output* (e.g., spam detection and language identification on the same text). MTL mostly a cost play.
  - *Partially-shared-input* (e.g., tasks that run on overlapping but non-identical feature sets). Needs explicit missingness handling; often the deciding complexity.
- **The decision rule** that made MTL the preferred regime over N independent models: labels-are-scarce-for-task-B, serving-cost-per-request-is-the-bottleneck, inference-latency-budget-forces-one-forward-pass, auxiliary-task-is-known-to-regularise-the-primary-task.
- **The exit criterion** — what measured outcome would make us split the task back out. This has to exist up front; otherwise MTL becomes the sunk-cost regime.

### 2.2 Architecture choice

The four canonical shapes and when each is right:

| Shape | What it is | When it fits |
|---|---|---|
| **Hard parameter sharing** | One shared trunk; one lightweight head per task. | Tasks are shared-input, shared-structure, and similar in difficulty. The default when MTL is justified at all. |
| **Soft parameter sharing** | Per-task trunks with a regulariser (e.g., L2 or cross-stitch) pulling them toward each other. | Tasks are related but have different dynamic ranges or very different gradient magnitudes. More capacity, more compute. |
| **Shared trunk + task-specific heads with task embeddings** | Hard sharing, but each task's head is conditioned on a learned task vector. | Many related tasks (dozens+); you want one model but need per-task capacity knobs. |
| **Mixture-of-experts (MoE) with task-gated routing** | Shared expert pool; a router picks experts per token/example, biased by task. | Many tasks of heterogeneous difficulty; compute budget allows expert activation sparsity. |

Senior-engineer notes:

- Hard sharing is the strongest baseline. If a design reaches for MoE on day one without an ablation against hard sharing, the reviewer should push back.
- "Task-specific heads" is not a shape by itself — every MTL shape has task-specific heads. The shape is defined by what happens *below* the heads.
- In practice, the hardest architectural question is not shape, it is **capacity per task**. A shared trunk sized for the hardest task over-serves the easiest tasks and vice versa; a bottleneck layer right before the per-task heads is cheap insurance.

### 2.3 Loss-weighting plan

MTL fails most often at the loss combiner, not the architecture. Five options to consider:

1. **Equal weights.** Fine as a *baseline*; almost never the final answer because task losses live at different numerical scales.
2. **Fixed weights set by grid search.** Cheap, interpretable, and brittle to data drift.
3. **Uncertainty weighting (Kendall, Gal & Cipolla, 2018).** Learn a log-variance per task; each task's loss is divided by its learned variance and a regularisation term is added. Robust first choice when task losses are heteroscedastic.
4. **GradNorm (Chen et al., 2018).** Normalise task-loss gradients at a shared layer to a target rate. Needs an extra backward pass, worth it when tasks have very different gradient magnitudes.
5. **Pareto / multi-objective (MGDA / Pareto-MTL).** Solve for a descent direction that improves all tasks simultaneously. Expensive; use when no fixed scalarisation is defensible.

The decision rule the design document should state:

- If task losses are within one order of magnitude of each other and their gradients are stable → equal weights, revisit if any task plateaus.
- If task losses span more than one order of magnitude → uncertainty weighting first.
- If gradients at the shared trunk are dominated by one task (measure this before training the full joint model) → GradNorm.
- If "improving task A always regresses task B" shows up at the loss-weight-tuning stage → consider Pareto-MTL, but also re-examine whether the tasks belong together at all.

### 2.4 Negative-transfer monitoring plan

Negative transfer is the only way MTL fails in production, and it fails silently unless the monitoring is explicit. The plan must contain:

1. **Single-task baselines per task.** Non-negotiable. Train each task alone on the same data and hold the result. The joint model is judged against these baselines, not against each other.
2. **The regression gate.** For every production-relevant task metric, a numeric regression threshold vs. its single-task baseline. A joint-model candidate that regresses any task by more than the gate on its holdout slice does not ship.
3. **Slice-level regression gates.** Aggregate metric parity can hide per-slice regressions. For each task, name the slices (user cohorts, locales, content categories) where parity must also hold.
4. **The rollback.** Named, reversible. "If task B regresses in production monitoring, we route task B's traffic to the single-task-B model behind a flag within one business day." This requires the single-task models to be *kept warm*, not archived.
5. **Drift triggers.** When input distributions move (new product launch, seasonality), re-run the regression gate against a fresh holdout before retraining the joint model, not after.

### 2.5 Illustrative worked design (not a real product)

> **Illustrative task set** — ranking candidate items in a hypothetical in-app feed with three tasks: (A) click probability, (B) dwell-time regression conditional on click, (C) report / flag probability. This example is a composite for calibration; it is not a real product and the numbers below are not measured.

Task-set justification:

| Task | Label | Loss | Relationship | MTL justification | Exit criterion |
|---|---|---|---|---|---|
| A — click prob. | binary click vs. no-click | logistic | primary | Shared impression features, must latency-share. | Does not apply alone. |
| B — dwell time | seconds-dwelt \| click | Gaussian NLL (or Huber) | shared-input, output conditional on A | Adding B is known to regularise A on datasets where dwell is correlated with content quality. | If click AUC improves by ≥ 0.5pp with B dropped. |
| C — report prob. | binary report vs. no-report | logistic (rare positive class) | shared-input, very low base rate | Skip supervised pretraining of a dedicated model; use A as a shared representation. | If report-rate recall stalls below the moderation team's floor and single-task C can match it on the same features. |

Architecture:

- **Hard parameter sharing**, shared transformer trunk over the impression features, three task-specific heads. Rationale: tasks A and B are shared-input, shared-structure; task C is rare-label but shares the impression representation. Hard sharing is the strongest baseline; we will not reach for MoE until the ablation proves hard sharing is capacity-limited.
- A **bottleneck layer** immediately below the three heads, sized for the single-task-A baseline. This insulates A's capacity from drifting as B and C's heads grow.

Loss weighting:

- Measured on the single-task baselines, the losses are within one order of magnitude after standard pre-processing, except for C whose rare-positive loss is a factor of 3–5 smaller in magnitude. → **Uncertainty weighting** as the first choice. Equal weights saved as the fallback ablation.
- Gradient monitoring at the shared bottleneck: log the per-task gradient norm. If any task's gradient dominates for > 10% of steps, switch to GradNorm.

Negative-transfer monitoring:

- Single-task A, B, C baselines trained and kept warm.
- Regression gates:
  - A: offline AUC must not regress vs. single-task A baseline by more than 0.3pp on the global holdout and 0.5pp on any named slice (new-user, low-activity).
  - B: offline RMSE must not regress vs. single-task B baseline by more than 2% on the global holdout.
  - C: recall@fixed-precision must not regress vs. single-task C baseline on the moderation-labelled holdout.
- Rollback: traffic for any regressing task is routed to its single-task model via a request-router flag within one business day; the single-task models are kept trained on the latest feature pipelines.
- Drift: on any input-schema change or seasonality marker (illustrative examples: a major product launch, holiday period opening), re-run the gates against a fresh holdout before promoting any joint-model candidate.

What the full artefact contains:

1. The task-set justification table (§2.1).
2. The architecture block and ablation plan vs. hard-sharing baseline (§2.2).
3. The loss-weighting plan with measured task-loss magnitudes and the fallback chain (§2.3).
4. The monitoring plan with regression gates, named slices, rollback, and drift triggers (§2.4).
5. A worked design on the team's actual task set (not this illustrative one).
6. A **reviewer-facing risk register** — one row per known MTL failure mode with the chosen mitigation, cross-referenced to NIST AI RMF MEASURE where applicable.

## 3. Validation steps

Before considering the design done, walk through this checklist:

1. **Every task has a single-task baseline.** No baseline, no gate, no ship.
2. **Loss scales measured, not assumed.** The loss-weighting choice must cite the measured per-task loss magnitudes on the single-task baselines. "We'll use uncertainty weighting because it's the newest technique" is not a justification.
3. **Architecture choice has an ablation.** If the design is MoE, the ablation plan must include hard-sharing. If the design is soft sharing, the ablation plan must include hard-sharing. Hard sharing is always the baseline.
4. **Regression gates are numeric, not qualitative.** "No significant regression on A" is not a gate. "AUC regression ≤ 0.3pp vs. single-task baseline" is a gate.
5. **Named slices exist per task.** Aggregate parity can hide a 5% regression on a 2% user cohort.
6. **Rollback is tested, not asserted.** The single-task models are kept warm and the router flag has been exercised in a drill. "We can roll back" is a hope; "we have rolled back in staging last week" is a plan.
7. **Exit criteria exist.** The design states, per task, the measurement that would make the team split that task back out. Without this, the design is permanent by default.

Static check:

```bash
grep -c "Single-task baseline" mtl-design.md       # expect >= number of tasks
grep -c "Regression gate"       mtl-design.md       # expect >= number of tasks
grep -c "Rollback"              mtl-design.md       # expect >= 1
grep -c "Exit criterion\|Exit criteria" mtl-design.md  # expect >= 1
```

## 4. Rubric / review checklist

- [ ] Task-set table lists every task with label, loss, relationship type, MTL justification, and exit criterion.
- [ ] Architecture block names the shape (hard-sharing / soft-sharing / task-conditioned / MoE) with reasoning tied to task relationships, not just "state of the art."
- [ ] The design includes an explicit ablation vs. hard-parameter-sharing baseline, regardless of which shape was chosen.
- [ ] Loss-weighting plan cites measured per-task loss magnitudes from the single-task baselines.
- [ ] Loss-weighting choice follows the stated decision rule (equal → fixed → uncertainty → GradNorm → Pareto-MTL) and names the fallback.
- [ ] Monitoring plan has a numeric regression gate per task and at least one named slice per task.
- [ ] Rollback is named (artefact path, flag name) and the single-task fallback models are kept warm.
- [ ] Drift triggers are enumerated (schema changes, seasonality) and the gate re-evaluation step is spelled out.
- [ ] Risk register cross-references NIST AI RMF where applicable, especially MEASURE (monitoring slices) and MANAGE (rollback).
- [ ] The design document has a "last reviewed" date and names the person on the hook for keeping the single-task baselines current.

Review-weighted points (for grading):

- Task-set justification + exit criteria: 20%
- Architecture choice + ablation plan: 20%
- Loss-weighting plan grounded in measured magnitudes: 20%
- Negative-transfer monitoring (gates, slices, rollback): 30%
- Hygiene (last-reviewed date, static checks, cross-refs): 10%

## 5. Common mistakes

- **No single-task baselines.** Without them there is no way to tell whether MTL helped or hurt. This is the single most common failure.
- **Picking MTL because it is novel.** MTL is an engineering choice motivated by shared features, shared latency, or label scarcity on one task. "We wanted to try MTL" is not on the list.
- **Treating "task-specific heads" as the architecture.** Every MTL architecture has task-specific heads. The real architectural question is the shared trunk and the capacity allocation.
- **Losses on different scales with equal weights.** A rare-positive logistic loss and a dense regression loss will not cooperate at equal weights. The regression loss drowns the logistic.
- **Letting one task's gradient dominate the shared trunk.** If task A's gradient at the shared layer is 20× task B's, the shared trunk is effectively training only on task A. Monitor per-task gradient norms during warmup.
- **Aggregate-only metrics.** A joint model can look flat on global AUC while regressing badly on the new-user cohort. Name the slices up front.
- **No rollback path.** Ship the joint model, delete the single-task models to save storage, then discover task B regresses with no artefact to roll back to. Keep the single-task models warm.
- **"We'll monitor for negative transfer" without a gate.** Monitoring without a numeric threshold is a pager-waiting-to-happen.
- **Permanent MTL by default.** If the design has no exit criterion, the team will accept a regression rather than split the model back out, because the joint model has become infrastructure.
- **Ignoring data-sampling imbalance.** Task A has 10M labels, task C has 50k. Naïve joint training starves task C. Either weight the loss, oversample task C, or use a per-task batch sampler — and state the choice in the design.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. Source of the MEASURE + MANAGE functions referenced by the per-task regression gates, slice-level monitoring, and rollback plan.
- MITRE ATLAS — <https://atlas.mitre.org/>. Source of the adversarial-ML context when one task's labels can be adversarially poisoned to degrade the shared trunk.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source of the input-validation and data-poisoning posture referenced in the risk register.

Local context:

- Paired lesson: `lessons/mod-303-advanced-modeling/exercises/exercise-02-multi-task-learning-in-practice.md`.
- Module README: `modules/mod-303-advanced-modeling/README.md`.

Prior art named in the loss-weighting plan (cited by name, not by metric):

- Kendall, Gal, Cipolla (2018) — uncertainty weighting for multi-task losses.
- Chen et al. (2018) — GradNorm: gradient normalisation for adaptive loss balancing.
- Sener & Koltun (2018) — multi-task learning as multi-objective optimisation (MGDA / Pareto-MTL).
