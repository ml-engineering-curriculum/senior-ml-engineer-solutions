# exercise-03 — Calibration and Uncertainty (Reference Solution)

Paired exercise: `lessons/mod-303-advanced-modeling/exercises/exercise-03-calibration-and-uncertainty.md`.

## 1. Solution overview

This exercise produces `calibration-plan.md` — a design that says, for one shipping model, (1) how its outputs will be calibrated, (2) what kind of uncertainty the system actually needs, (3) how those two things will be measured, and (4) how the post-training calibration will be re-fit on drift. The artefact has five parts:

1. A **decision frame** — what downstream decision consumes the model's score, and what it costs to be wrong in each direction. Calibration is useful only if a downstream threshold reads the probability.
2. A **calibration method pick** — Platt scaling, isotonic regression, temperature scaling, histogram binning, or beta calibration — with the reasoning tied to the decision frame and the sample size.
3. An **uncertainty posture** — distinguish aleatoric (data-inherent) from epistemic (model-ignorance) uncertainty and name which the system acts on. Then pick a producer for it (deep ensemble, MC dropout, conformal prediction, Bayesian last layer).
4. A **measurement plan** — which calibration metric (ECE, MCE, Brier, reliability diagram, proper scoring rule) under which binning, measured on which slice.
5. A **recalibration schedule** — when the calibrator is re-fit, on what data, and what triggers re-fit between scheduled runs.

The reference solution below walks through the five parts on one illustrative problem, labels it clearly as illustrative, and gives the grading rubric that applies to every learner's plan.

The exercise exists because "my model outputs a probability" and "my model's probability means what you think it means" are two different sentences, and senior ML engineers get this wrong regularly.

## 2. Implementation (worked answer)

### 2.1 Decision frame

Before picking a method, write down the decision.

- **Where does the score flow?** A ranker whose output is consumed only by sort order does not need calibration — monotone transformations do not change the ranking. A threshold-based gate (block / review / auto-approve) absolutely needs calibration.
- **What are the asymmetric costs of being wrong?** False-positive cost vs. false-negative cost. A calibrated probability with a decision-theoretic threshold is what lets the ops team move the threshold when costs change, without retraining.
- **Is the probability shown to a human?** "87% likely fraud" means something specific; if it is not calibrated, the UI is lying.
- **Are multiple models being combined?** Averaging an uncalibrated score with a calibrated one makes both worse.

Senior-engineer check: if the decision frame does not need a probability — only a sort order — document that explicitly and skip calibration. Calibrating a pure ranker is wasted work and introduces a moving part that can rot.

### 2.2 Calibration method pick

Post-hoc calibrators (fit on a *held-out calibration set*, never on the training set):

| Method | What it does | When it fits | Pitfalls |
|---|---|---|---|
| **Platt scaling** | Logistic regression on the raw score; two parameters. | Small calibration set (hundreds). Works best when miscalibration is sigmoidal (common for margin-style models). | Can't fix non-monotone miscalibration. |
| **Isotonic regression** | Non-parametric, piecewise-constant, monotone. | Larger calibration set (thousands+). Handles arbitrary monotone miscalibration. | Over-fits small sets; produces piecewise-constant outputs (degenerate ties). |
| **Temperature scaling** | One scalar divides the logits before softmax. | Modern deep classifiers that are confidently wrong (over-confident softmax). Single parameter, cheap, strong baseline. | Only rescales; cannot fix per-class miscalibration. |
| **Histogram binning** | Bin scores, replace with empirical rate per bin. | Interpretable, matches reliability-diagram construction. | Bin boundaries and counts are hyperparameters; sensitive to tails. |
| **Beta calibration** | Two-parameter beta-family map; strictly more flexible than Platt. | Binary classifiers where Platt is close but mis-shapes the tails. | Still parametric; same weaknesses as Platt when the shape is wildly wrong. |

Decision rule the artefact should state:

- Deep-network classifier with over-confident softmax, multi-class → **temperature scaling** first.
- Binary classifier, small calibration set (<1000) → **Platt** or **beta**.
- Binary classifier, large calibration set and non-monotone-looking reliability curve → **isotonic**.
- Need a piecewise-constant interpretable calibrator for audit → **histogram binning**.
- Multi-class with per-class miscalibration (class imbalance) → **temperature + per-class bias** (vector scaling) or **Dirichlet calibration**.

Fit protocol (non-negotiable):

- Calibration set is **separate from the training set and from the test set**. A model calibrated on its own training data is not calibrated; it is overfit.
- Re-fit the calibrator whenever the base model is retrained. A calibrator trained against last month's logits is measuring a model that no longer exists.
- Freeze the calibrator alongside the model artefact, same version, same registry entry. Mismatch here is the single most common deployment bug.

### 2.3 Uncertainty posture

Two kinds of uncertainty, and they do different jobs:

- **Aleatoric uncertainty** — irreducible noise in the data. Two identical inputs that got different labels contribute aleatoric uncertainty. Not reduced by more data.
- **Epistemic uncertainty** — the model's ignorance. High on out-of-distribution inputs, low on dense regions of training data. Reduced by more data.

Pick based on what the downstream system needs to *do*:

- "Defer to human review when the model is uncertain" → epistemic (we should defer on OOD inputs).
- "Report a prediction interval" → aleatoric + epistemic, usually via a proper scoring rule.
- "Flag anomalous inputs" → epistemic (OOD detection).
- "Weight predictions in an ensemble" → either, depending on which ensemble.

Producers (ordered by production cost):

| Producer | Cost | Produces | Senior notes |
|---|---|---|---|
| **Max-softmax / single-model confidence** | Free | A number. Not uncertainty. | The default when no plan exists. Rarely enough on its own. |
| **MC Dropout (Gal & Ghahramani 2016)** | Low (keep dropout on at inference, average N forward passes) | Approximate epistemic. | Works, but N inference passes = N× latency. |
| **Deep ensembles (Lakshminarayanan et al. 2017)** | Medium at train, high at serve | Robust epistemic + aleatoric via output variance. | Strongest baseline for calibrated uncertainty; costs scale linearly with ensemble size. |
| **Bayesian last layer / SWAG** | Medium | Approximate posterior over the last layer. | Cheaper than full ensemble; needs care. |
| **Conformal prediction** | Low | Distribution-free prediction *sets* with coverage guarantees. | Guarantees are marginal and conditional-on-exchangeability; still often the pragmatic winner for threshold systems. |

If the decision frame is "defer to human on uncertain", conformal prediction with a target coverage of 1 − α and a scoring function matched to the loss is usually the right answer. It is the only producer in the list that gives a *guaranteed* coverage property without distributional assumptions on the model.

### 2.4 Measurement plan

Metrics (each with a specific use):

- **Expected Calibration Error (ECE).** Weighted average gap between predicted probability and empirical rate across bins. Easy to compute, very bin-dependent. Report ECE with **the number of bins** and **the binning scheme** (equal-width vs. equal-mass). An ECE number without its binning is uninterpretable.
- **Maximum Calibration Error (MCE).** The worst-bin gap. Useful because the worst bin is often what causes incidents.
- **Brier score.** Proper scoring rule; decomposes into calibration + refinement + irreducible loss. The right choice when you want a single number that reflects both calibration and sharpness.
- **Negative log-likelihood (NLL).** Proper scoring rule that is sensitive to calibration at the extremes; preferred when the downstream cost is log-loss-shaped.
- **Reliability diagram.** The picture ECE compresses. Always include the diagram alongside ECE — a flat, boring ECE can hide a U-shaped reliability curve.
- **Coverage (for prediction sets).** Measured against the nominal 1 − α. If you ship conformal at α = 0.1, your test set coverage should be ≈ 0.9.

Slicing (non-negotiable):

- **Compute every calibration metric globally and per slice.** The slices are the same ones you use for fairness and reliability work: user cohorts, locales, content categories, device classes, time windows.
- Aggregate ECE can look good while the model is wildly miscalibrated on a 2% slice that matters.
- Time windows matter: a model calibrated on last quarter's data, measured on last week's data, is measuring drift as much as calibration.

Confidence intervals: for a given slice size, ECE has non-trivial sampling noise. Bootstrap the ECE on the slice and report the interval, not a point estimate. This makes "did calibration regress or is it noise" a measurable question.

### 2.5 Recalibration schedule

The artefact must name:

- **A scheduled recalibration cadence** (e.g., weekly) with a named calibration-set assembly step. The calibration set is a fresh holdout, **not** the training set.
- **Triggers for off-schedule recalibration**:
  - Base-model retraining (always re-fit the calibrator when the base model changes).
  - Feature-pipeline schema change.
  - Observed drift in input distribution (PSI, KL, or Wasserstein crossing a named threshold on any monitored feature).
  - Observed drift in output distribution (score histogram shifts).
  - Observed ECE regression on any monitored slice above a named threshold.
- **A gate and a rollback.** Any new calibrator must pass per-slice ECE, Brier, and coverage gates against the prior calibrator on the fresh holdout before being promoted. If it regresses, roll back the calibrator independently of the base model.

### 2.6 Illustrative worked plan (not a real product)

> **Illustrative problem** — an in-house fraud screening classifier where the output probability drives a three-way action: auto-approve (p < 0.2), human review (0.2 ≤ p < 0.7), auto-block (p ≥ 0.7). This example is a composite for calibration; the thresholds and numbers are not measured.

Decision frame:

- Thresholds consume the probability directly → calibration required.
- Asymmetric costs: false-block is CX-visible and reputation-costly; false-approve is dollar-cost. The ops team wants to be able to retune thresholds without retraining the base model.
- The probability is shown to the human reviewer → calibration also affects reviewer trust.
- Costs: calibration is on the critical path.

Calibration method pick:

- The base classifier is a deep neural network with over-confident softmax, binary task. Calibration-set size will be tens of thousands.
- **Temperature scaling** is the first pick (deep-network over-confidence is the dominant miscalibration mode in this family). **Isotonic regression** is held as the fallback if the reliability diagram shows non-monotone miscalibration after temperature-scaling.
- Protocol: calibration set is the most recent two weeks of labelled traffic, held out from training. Calibrator version is pinned to the base-model version in the model registry.

Uncertainty posture:

- Downstream job: defer to human when uncertain. We need **epistemic** uncertainty.
- Pick **conformal prediction** on the calibrated score, with target coverage 1 − α = 0.9. The non-conformity score is 1 − p for the predicted class. Inputs whose calibrated probability falls in the conformal "both classes in the set" regime are routed to review, regardless of the point threshold.
- Deep ensembles considered and rejected on cost (K× inference latency is outside this model's SLO).

Measurement plan:

- ECE (15 equal-mass bins), MCE, Brier, and reliability diagram, computed globally and on these named slices: new-account cohort (<30 days), large-transaction cohort (illustrative dollar threshold), weekend traffic, each merchant category. Bootstrapped 95% CIs on each.
- Conformal coverage measured globally and on the same slices. Target: ≥ 0.88 on every slice (slightly below the 0.9 marginal nominal because conditional coverage is weaker than marginal).
- Dashboard updated nightly; weekly review meeting owns the metrics.

Recalibration schedule:

- **Weekly** scheduled recalibration on the most recent two weeks of labelled traffic.
- Off-schedule triggers:
  - Base-model retrain.
  - PSI on any monitored feature > 0.2.
  - Slice-level ECE regression > 2pp vs. the previous week on any named slice.
- Gate: new calibrator must beat or match prior calibrator on global Brier and on per-slice ECE (not regress by more than 0.3pp on any slice). Rollback: the previous calibrator is kept as a named artefact in the registry.

## 3. Validation steps

Walk through this checklist on the artefact:

1. **Decision frame is first.** If the artefact opens with the calibrator choice before stating what the probability is used for, the author skipped the hardest step. Return.
2. **Calibration set is separate.** The document names three disjoint sets: train, calibration, test. If it does not, the calibration will overfit silently.
3. **Calibration method matches the diagnosis.** The pick must cite a reason grounded in the reliability diagram or the base-model class (e.g., "temperature scaling because softmax over-confidence"), not state-of-the-art.
4. **Uncertainty kind is named.** "Uncertainty" by itself is not enough. The document must say aleatoric or epistemic, and tie the choice to what the downstream system does with it.
5. **Metrics named with their binning.** ECE without its binning scheme is not reportable.
6. **Reliability diagram included or scripted.** A number without the picture is weak.
7. **Per-slice measurement is mandatory.** Global metrics alone do not pass review.
8. **Recalibration cadence has off-schedule triggers.** A calendar-only cadence misses every meaningful drift event.
9. **Rollback path exists for the calibrator independent of the base model.** The calibrator is a separately-versioned artefact. If the only rollback is "retrain everything," the cadence is broken.
10. **Alignment with NIST AI RMF MEASURE.** The measurement plan ties to measurable characteristics of trustworthy AI; the document should cite this explicitly for any regulated-domain deployment.

Static check:

```bash
grep -c "Decision frame"       calibration-plan.md    # expect 1
grep -c "Calibration set"      calibration-plan.md    # expect 1+
grep -c "ECE\|Brier\|NLL"      calibration-plan.md    # expect >= 2
grep -c "Slice\|slice"         calibration-plan.md    # expect >= 2
grep -c "Rollback"             calibration-plan.md    # expect >= 1
grep "last reviewed"           calibration-plan.md    # must exist, within 90 days
```

## 4. Rubric / review checklist

- [ ] Decision frame is the first section and names what the probability drives (sort, threshold, UI display, ensemble).
- [ ] Asymmetric-cost reasoning is explicit if a threshold consumes the probability.
- [ ] Calibration method picked with a stated reason tied to base-model class and calibration-set size; fallback method named.
- [ ] Calibration set, training set, and test set are disjoint and named.
- [ ] Uncertainty kind (aleatoric vs. epistemic) is named and tied to the downstream action.
- [ ] Uncertainty producer named with cost reasoning (ensemble vs. MC dropout vs. conformal vs. Bayesian last layer).
- [ ] Measurement plan names metrics with their binning / target coverage, and includes reliability diagrams or a scripted equivalent.
- [ ] Per-slice measurement is mandatory and the slices are listed.
- [ ] Recalibration cadence has both scheduled and trigger-based re-fits, with explicit drift thresholds.
- [ ] Calibrator has its own registry entry, versioned with the base model, with a rollback path.
- [ ] NIST AI RMF MEASURE alignment cited for monitoring; MITRE ATLAS referenced if the calibrator is a visible surface for adversarial score manipulation.
- [ ] Artefact has a "last reviewed" date and an owner for the calibration dashboard.

Review-weighted points (for grading):

- Decision frame correctness (does the model actually need calibration?): 15%
- Calibration method pick + fit protocol (set discipline): 25%
- Uncertainty posture (kind + producer tied to downstream action): 20%
- Measurement plan (metrics, binning, slices, CIs): 25%
- Recalibration schedule + rollback: 15%

## 5. Common mistakes

- **Calibrating a pure ranker.** If the downstream only reads the sort order, calibration is wasted work. The decision frame should expose this.
- **Fitting the calibrator on the training set.** Classic silent overfit. The reliability diagram on the training data looks perfect; the deployed model does not.
- **Reporting ECE without the binning scheme.** ECE values across different bin counts and binning schemes are not comparable. Always report both.
- **Aggregate metric only.** A 2% user cohort with 20% ECE can be invisible in a 1% global ECE.
- **Treating "uncertainty" as a single thing.** Aleatoric and epistemic do different jobs. Conflating them leads to the wrong producer and the wrong downstream action.
- **MC dropout as a panacea.** It is a cheap approximation, not a guarantee; and the N× inference latency is often fatal in production.
- **Deep ensemble as the only serious answer.** Ensembles are strong, but conformal prediction gives a distribution-free coverage guarantee on a much smaller serving budget — do not dismiss it.
- **Forgetting the calibrator rollback.** The calibrator is a versioned artefact. Deploying a new base model and silently re-fitting the calibrator without a rollback is a single-commit-wide outage waiting to happen.
- **No off-schedule drift trigger.** Weekly recalibration catches gradual drift but not launch-day distribution shocks. Both are needed.
- **Confusing calibration with sharpness.** A model that always predicts the base rate is perfectly calibrated and useless. Proper scoring rules (Brier, NLL) penalise this; ECE alone does not. The design must include at least one proper scoring rule.
- **Displaying a probability to a user without calibrating it.** This is a UX-truthfulness bug, not just a modeling bug.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. Source of the MEASURE function (measurable characteristics of trustworthy AI, including accuracy, reliability, and bias) that justifies per-slice calibration measurement and bootstrapped CIs.
- MITRE ATLAS — <https://atlas.mitre.org/>. Source of the adversarial-ML context for score-manipulation threats; calibrators that depend on observed input distributions can be targeted.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source of the input-validation and data-poisoning context relevant to the calibration-set assembly step.

Local context:

- Paired lesson: `lessons/mod-303-advanced-modeling/exercises/exercise-03-calibration-and-uncertainty.md`.
- Module README: `modules/mod-303-advanced-modeling/README.md`.

Prior art named by name:

- Platt (1999) — probabilistic outputs for support vector machines (Platt scaling).
- Zadrozny & Elkan (2002) — transforming classifier scores into accurate multiclass probability estimates (isotonic regression for calibration).
- Guo, Pleiss, Sun, Weinberger (2017) — on calibration of modern neural networks (temperature scaling).
- Kull, Silva Filho, Flach (2017) — beta calibration.
- Gal & Ghahramani (2016) — dropout as a Bayesian approximation (MC dropout).
- Lakshminarayanan, Pritzel, Blundell (2017) — simple and scalable predictive uncertainty estimation using deep ensembles.
- Vovk, Gammerman, Shafer — algorithmic learning in a random world (conformal prediction).
