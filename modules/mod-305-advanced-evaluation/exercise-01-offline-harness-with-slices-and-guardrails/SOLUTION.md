# exercise-01 — Offline Harness With Slices And Guardrails (Reference Solution)

Paired exercise: `lessons/mod-305-advanced-evaluation/exercises/exercise-01-offline-harness-with-slices-and-guardrails.md`.

## 1. Solution overview

This exercise produces `offline-harness.md` plus a small set of runnable artefacts — a harness configuration, a slice registry, and a guardrail policy — that together define the offline evaluation a model must pass before any online test (shadow, canary, A/B) is allowed to start. The artefact has five parts:

1. A **harness frame** that pins down what the harness is for: the model under test, the task, the eval dataset(s), their provenance, and the decision the harness feeds.
2. A **metric panel** — the headline metric, the secondary metrics, and the mandatory *floor* metrics (the ones a candidate cannot regress even if the headline moves up).
3. A **slice registry** — the named subpopulations the harness must report on separately, with the business reason each slice exists.
4. A **guardrail policy** — the thresholds that must hold on headline, floors, and slices for a candidate to be promoted, plus the explicit tie-breakers when thresholds conflict.
5. A **failure-handling and freshness plan** — what happens when a slice is too small to measure, when the eval set has drifted, when a metric is noisy, and how often the harness itself is re-audited.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric that calibrates every learner's harness to the same bar.

The exercise exists because offline metrics are the gate every subsequent online test leans on. A harness without slice reporting and floor metrics ships a model that looks better on average and is worse where it matters; a harness without a documented guardrail policy ships decisions that change shape with whoever is in the room. The NIST AI RMF MEASURE function treats reliable, slice-aware measurement as a precondition for risk management; this artefact is the team's MEASURE discipline for one model.

## 2. Worked answer / implementation

### 2.1 Harness frame

Six lines, in this order:

- **Model under test.** Name, version, task, output schema. One candidate, one baseline. "The current production model" counts as a baseline only if it is pinned to a version, not an alias.
- **Decision the harness feeds.** "Promote to shadow eval," "promote to 5% canary," "promote to full rollout." Different downstream decisions have different error budgets and may share the harness but not the gate.
- **Eval datasets.** The frozen golden set, any backtest windows, any adversarial / stress sets. Each with (a) provenance — where the labels come from, (b) freeze date, (c) known biases or gaps, (d) licence / data handling notes if relevant.
- **Scoring protocol.** How predictions are turned into metrics: is the model called online in eval, or scored offline against cached features? Random seeds pinned? Tie-breaking rules for duplicate inputs?
- **Who signs off.** The named role that approves a promotion when the harness says "pass." Not a person's name — a role, so the gate survives turnover.
- **Last reviewed.** A date, within the review cadence declared in §2.5.

### 2.2 Metric panel

Three tiers. All three are mandatory.

**Headline (one).** The single metric the team is optimising. One number. Ranked, classification, regression, generation — pick the one that aligns with the business objective, not the one that is easiest to compute. If the team cannot pick a single headline, the frame is ambiguous; iterate on §2.1 before the metrics.

**Secondary (two to four).** Supporting metrics that explain *why* the headline moved. For a ranker: NDCG at k alongside headline mean reciprocal rank. For a classifier: precision and recall alongside a thresholded F or AUC. These are not gates; they are explanations.

**Floors (two to five).** The metrics that must not regress regardless of headline. Floors are where "optimised the wrong thing" gets caught before launch. Typical floors:

- **Calibration floor.** Expected Calibration Error (or reliability-diagram deviation) bounded. A sharper model that lies about its confidence is worse for every downstream consumer.
- **Latency floor.** p95 inference latency within the serving budget. A slower model is a different model.
- **Fairness / slice-worst floor.** Worst-slice headline cannot regress more than X relative to the previous version. See §2.3.
- **Safety floor.** For generative or classification-with-consequence models, the rate of policy violations or protected-category errors. NIST AI RMF MEASURE 2.x treats this as a first-class metric, not a side check.
- **Stability floor.** Variance across random seeds / data folds. A model whose quality depends on the seed cannot be trusted in production.

**Rule of the panel:** moving the headline is necessary; not regressing any floor is sufficient. A candidate that improves the headline and regresses a floor fails until the regression is either (a) accepted with an explicit, dated, named-role waiver, or (b) mitigated.

### 2.3 Slice registry

A slice is a named subpopulation of the eval set where the model's behaviour has a specific business or risk meaning. Each slice entry contains:

- **Slice name.** Short, readable, used as the metric label.
- **Definition.** Deterministic predicate on input features or labels. If the predicate is not deterministic, the slice is not a slice; it is a vibe.
- **Why it exists.** One sentence: the business, fairness, or safety reason. If this sentence is "for completeness," delete the slice — slice sprawl makes every launch look worse without informing a decision.
- **Minimum sample size.** The smallest N at which the slice metric is reportable. Below this, the harness reports "insufficient sample" and the slice does not block or pass.
- **Guardrail class.** Whether this slice participates in the slice-worst floor (§2.2), is report-only, or is a hard reject.

Typical slice families (adapt to the problem):

- **Demographic / protected-attribute slices.** Required where applicable law or NIST AI RMF MEASURE 2.11 "fairness and bias" characterisation applies. These are hard-gated.
- **High-value-segment slices.** Top-N customers by revenue, enterprise plan, long-tail power users. Business-critical.
- **Temporal slices.** Last 7 / 30 / 90 days; holiday windows; post-incident windows. Catch drift.
- **Input-shape slices.** Short vs. long text, cold-start users with < N interactions, locale / language, device class.
- **Adversarial / stress slices.** Known-hard examples, red-team prompts, OWASP ML Top 10 prompt-injection cases for LLM features, MITRE ATLAS-tagged evasion examples for classifiers.
- **Regression-defence slices.** Examples the previous model got right. If the new model loses them, the headline alone will not catch it until it is too late.

Senior-engineer notes:

- **Slices must be versioned with the eval set.** A slice predicate that depends on a feature the production pipeline no longer emits is a silent reporting hole.
- **No slice is "infinite" in importance.** Each slice entry names a weight or a rank. Without it, every slice that moves blocks every launch.
- **"Protected attribute" slice reporting may be constrained by policy.** If raw attributes cannot be stored, the registry documents the proxy or sampled-audit protocol used instead, and dates the policy review.

### 2.4 Guardrail policy

The policy is the rulebook the harness runs before it returns `pass` or `fail`. Each rule names the condition and the decision. First match wins; the default is `fail`.

```
G1 (missing data):
  If any required dataset is missing, stale past its freshness window, or
  under its minimum sample size → FAIL. Harness returns "inconclusive,"
  not "pass."

G2 (floor regression):
  If any floor metric regresses more than its tolerance versus the pinned
  baseline → FAIL. The candidate cannot be promoted without a waiver.
  Waivers are dated, name a role, and expire.

G3 (worst-slice regression):
  If any hard-gated slice regresses the slice headline by more than its
  tolerance → FAIL.

G4 (headline regression):
  If the headline regresses versus the baseline by more than its noise
  band → FAIL.

G5 (headline flat, no improvement elsewhere):
  If headline is within noise and no secondary metric moves either →
  PASS-BUT-HOLD. The candidate is not worse; it is also not better.
  Shipping yes/no is a product call, not an eval call.

G6 (headline up, floors and slices hold):
  Headline improves beyond the noise band, no floor regresses, no hard-
  gated slice regresses beyond tolerance → PASS.

G7 (headline up, report-only slice regresses):
  Headline improves; a report-only slice regresses. → PASS WITH NOTE.
  The note is appended to the launch record and reviewed at the next
  cadence.

G8 (unknown input):
  Anything not covered above → FAIL. New cases get rules, not improvised
  judgements.
```

Tolerances are explicit numbers (percentage points, absolute units), not words. "Noise band" means the empirical standard error on the metric over the eval set, measured by bootstrap or seed variance, not a vibe.

Senior-engineer notes:

- **Waivers are an escape hatch, not a policy.** A waiver is logged with the regression size, the mitigation, the owner, and an expiry. A waiver that silently extends is a policy change; call it that.
- **Noise bands are measured per metric, per eval set.** A tolerance that works for a dense classifier on a 100k-row set will be too tight for a slice of 2k rows.
- **Policy versioning matters.** Changing a tolerance mid-release is a decision that needs its own review. Pin the policy version alongside the model version in the harness output.

### 2.5 Failure-handling and freshness plan

- **Insufficient-sample slices.** The harness reports the slice as "insufficient" and does not block or pass. The slice registry has a standing action to grow the eval set for that slice; the review cadence checks progress.
- **Stale eval set.** Each dataset has a freshness window. Past the window, the harness runs in "advisory" mode only — the output is informational, not gating. Re-freezing the set is a scheduled work item.
- **Flaky metrics.** A metric whose variance across pinned seeds exceeds a defined bound is quarantined: reported, not gated, and routed to an owner for stabilisation.
- **Harness audit.** The harness itself is audited on a cadence (quarterly is typical) for: slice coverage, metric relevance, noise-band validity, guardrail drift, and dataset refresh status.
- **Change log.** Every change to the metric panel, slice registry, or guardrail policy is logged with date, reviewer, and reason. The change log is read before every major launch.

### 2.6 Illustrative worked example (not a real model)

> **Illustrative feature** — a tabular classifier that predicts whether an inbound support ticket should be auto-routed to tier-2 engineering. Binary label, trained on 90 days of labeled routing decisions, served synchronously behind the ticket intake form. This example is a composite for calibration; do not treat it as a real system or incident.

Harness frame:

- Model under test: `router-v12` (gradient-boosted trees). Baseline: `router-v11` (pinned version in prod).
- Decision fed: promotion to shadow eval (traffic mirror, no user-visible effect).
- Eval datasets: frozen golden set `golden-2026-09.parquet` (last 90 days of labeled tickets, 28k rows, held-out from training); adversarial set `redteam-2026-09.parquet` (ambiguous and category-swapped tickets, 1.4k rows).
- Scoring protocol: offline batch scoring against cached features, five pinned seeds, bootstrap CIs at 95%.
- Sign-off: ML platform on-call plus the product owner for the routing surface.
- Last reviewed: 2026-10-20.

Metric panel:

- Headline: PR-AUC on the golden set.
- Secondary: precision at the operating threshold; recall at the operating threshold; threshold itself (so a shift in operating point is visible).
- Floors:
  - Calibration: ECE ≤ 0.03 on the golden set.
  - Latency: p95 serving latency ≤ 50 ms.
  - Slice-worst: no hard-gated slice regresses PR-AUC more than 0.02 absolute.
  - Safety: false-positive rate on the "do-not-auto-route" slice (tickets that policy says must go to a human) ≤ 0.5%.
  - Stability: PR-AUC standard deviation across the five seeds ≤ 0.005.

Slice registry (abbreviated):

| Slice | Definition | Why | Min N | Class |
|---|---|---|---|---|
| `locale:en` | `locale == 'en'` | Largest traffic share; headline must hold here. | 5000 | hard |
| `locale:non-en` | `locale != 'en'` | Risk of silent quality loss outside English. | 2000 | hard |
| `enterprise` | `plan_tier in {enterprise, strategic}` | Business-critical accounts; worst-slice guardrail. | 500 | hard |
| `cold-start-user` | `user_ticket_count < 3` | Thin features; known failure region. | 1000 | hard |
| `last-7-days` | ticket within last 7 days of eval set | Drift detection. | 2000 | hard |
| `do-not-auto-route` | policy flag `must_human_route == true` | Safety floor; false positives here are the harm case. | 300 | hard |
| `short-body` | `token_count < 20` | Input-shape robustness. | 1000 | report-only |
| `adversarial` | all rows in `redteam-2026-09.parquet` | Stress test. | 1400 | hard |

Guardrail policy:

Per §2.4 above, with these tolerances: headline PR-AUC improvement band ±0.004 (bootstrap-estimated noise); floor tolerances as listed; slice-worst tolerance 0.02 absolute; adversarial-slice tolerance 0.03.

Trace for `router-v12`:

1. G1? All datasets present, in-window, above minimum N. Does not fail.
2. G2? ECE = 0.025 (floor 0.03, OK); p95 = 42 ms (OK); stability sd = 0.004 (OK); safety FPR on `do-not-auto-route` = 0.3% (floor 0.5%, OK). Does not fail.
3. G3? Worst hard-gated slice is `locale:non-en`: PR-AUC moves from 0.71 to 0.72 (+0.01). Within tolerance. Does not fail.
4. G4? Headline PR-AUC moves 0.83 → 0.86 (+0.03). Well outside the noise band. Does not fail.
5. G5? Not applicable.
6. G6? Headline up beyond noise, floors hold, hard-gated slices hold. → PASS.
7. G7? `short-body` slice PR-AUC drops 0.68 → 0.66 (−0.02). Report-only; note appended for the next harness audit.

Harness output:

> `router-v12` PASS with note. Promote to shadow eval. Note: `short-body` slice regressed 0.02 (report-only); flag for the next quarterly audit. Last reviewed 2026-10-20 by ML platform on-call and routing product owner.

### 2.7 Runnable sketch: harness output contract

The harness is not just a dashboard; it is a function that returns a decision. A minimal contract (Python-shaped pseudocode — adapt to the team's stack):

```python
# harness/contract.py
from dataclasses import dataclass, field
from typing import Literal

HarnessDecision = Literal["pass", "pass-with-note", "pass-but-hold", "fail", "inconclusive"]

@dataclass
class MetricReport:
    name: str
    value: float
    baseline: float | None
    delta: float | None
    noise_band: float
    is_floor: bool

@dataclass
class SliceReport:
    name: str
    headline: MetricReport
    sample_n: int
    min_n: int
    class_: Literal["hard", "report-only"]
    status: Literal["pass", "regressed", "insufficient"]

@dataclass
class HarnessResult:
    model_version: str
    baseline_version: str
    policy_version: str
    eval_sets: list[str]
    decision: HarnessDecision
    headline: MetricReport
    floors: list[MetricReport]
    slices: list[SliceReport]
    notes: list[str] = field(default_factory=list)
    waivers: list[str] = field(default_factory=list)
    reviewed_at: str = ""
```

The contract is the point. A harness that returns a dashboard but not a decision pushes the gate onto whoever is in the room; a harness that returns a decision makes the gate auditable.

### 2.8 Runnable sketch: a single gate check

```python
# harness/gate.py
def evaluate_candidate(result: HarnessResult) -> HarnessDecision:
    # G1
    for s in result.slices:
        if s.sample_n < s.min_n and s.class_ == "hard":
            return "inconclusive"
    # G2
    for f in result.floors:
        if f.delta is not None and _regressed(f):
            return "fail"
    # G3
    for s in result.slices:
        if s.class_ == "hard" and s.status == "regressed":
            return "fail"
    # G4
    if result.headline.delta is not None and result.headline.delta < -result.headline.noise_band:
        return "fail"
    # G5
    if result.headline.delta is not None and abs(result.headline.delta) <= result.headline.noise_band:
        return "pass-but-hold"
    # G6 / G7
    report_only_regressions = [s for s in result.slices if s.class_ == "report-only" and s.status == "regressed"]
    if report_only_regressions:
        result.notes.extend(f"report-only slice regressed: {s.name}" for s in report_only_regressions)
        return "pass-with-note"
    return "pass"

def _regressed(m: MetricReport) -> bool:
    # Floor-direction aware regression: a floor can be "higher is worse" (ECE, latency) or
    # "lower is worse" (calibration-adjusted precision). The MetricReport should carry
    # direction; omitted here for brevity.
    return m.delta is not None and m.delta < -m.noise_band
```

The gate is intentionally simple. The complexity lives in the data (slices, noise bands, tolerances), not in the control flow — a pattern worth preserving.

## 3. Validation steps

Walk the artefact through this checklist:

1. **Headline is a single metric.** If it is a weighted combination, the weights are in the artefact and were reviewed.
2. **Every floor has a direction and a number.** "Calibration is good" is not a floor.
3. **Every slice has a predicate, a reason, a minimum N, and a class.** No vibe slices.
4. **Noise bands are measured, not guessed.** The method (bootstrap, seed variance) is named.
5. **Guardrail policy has a version.** The harness output pins policy version alongside model version.
6. **Harness has a single-point output.** Running the harness produces a `HarnessDecision`; a dashboard-only harness is not sufficient.
7. **Waiver process is defined.** Waivers have expiries; expired waivers trip the gate.
8. **Stale-set behaviour is defined.** Past the freshness window, the harness reports advisory, not gating.
9. **Adversarial / stress slice exists.** For any model with a non-trivial safety floor, an adversarial slice is present.
10. **Last-reviewed date within the declared cadence.**

Static checks to drop into CI:

```bash
grep -c "^G[0-9]" offline-harness.md                                           # expect at least 8 guardrail rules
grep -ci "slice registry" offline-harness.md                                    # must exist
grep -ci "noise band\|bootstrap\|seed variance" offline-harness.md              # noise handling named
grep -ci "last reviewed" offline-harness.md                                     # must exist
grep -ci "NIST AI RMF\|OWASP\|MITRE ATLAS" offline-harness.md                   # standard cited
grep -c "report-only\|hard" offline-harness.md                                  # slice classes used
```

For the runnable artefact, a smoke test on a synthetic `HarnessResult` should exercise at least one case per guardrail rule.

## 4. Rubric / review checklist

- [ ] Harness frame is six lines, names model under test, baseline (pinned), eval datasets with provenance, scoring protocol, sign-off role, and last-reviewed date.
- [ ] Single headline metric declared; secondary metrics declared and used for *explanation*, not gating.
- [ ] At least three floor metrics declared with directions, numbers, and noise bands: calibration, latency, slice-worst, safety, stability — pick the ones that apply.
- [ ] Slice registry has at least one slice per family where applicable: demographic / high-value / temporal / input-shape / adversarial / regression-defence.
- [ ] Every slice names a predicate, a reason-for-existing sentence, a minimum N, and a class (`hard` / `report-only`).
- [ ] Guardrail policy is an ordered rule list (G1..Gn), first match wins, default `fail`, numeric tolerances.
- [ ] Noise bands measured per metric per eval set; method named.
- [ ] Harness emits a single `HarnessDecision` (pass / pass-with-note / pass-but-hold / fail / inconclusive). A dashboard alone is insufficient.
- [ ] Waiver process defined (dated, named role, expiry); expired waivers trip the gate.
- [ ] Freshness plan defined for every eval set; stale-set behaviour is advisory, not silently gating.
- [ ] Harness audit cadence (quarterly typical) with named artefacts to re-review.
- [ ] NIST AI RMF MEASURE function cited where fairness / safety / stability floors are justified.
- [ ] If the system has a generation or classification-with-consequence mode, OWASP ML Top 10 or MITRE ATLAS tactics are referenced in the adversarial / safety slice design.
- [ ] Last-reviewed date within the declared cadence.

Review-weighted points for grading:

- Harness frame specificity and sign-off clarity: 10%
- Metric panel (headline / secondary / floors with directions and noise bands): 20%
- Slice registry (coverage, classes, predicates, minimum N): 20%
- Guardrail policy (ordered, numeric, first-match): 20%
- Freshness and waiver process (dated, named, expiring): 10%
- Runnable artefact (contract plus a gate function covering all rules): 10%
- Standards alignment (NIST AI RMF MEASURE, OWASP ML Top 10 or MITRE ATLAS where applicable) and hygiene: 10%

## 5. Common mistakes

- **Reporting only the mean.** Headline-only harnesses ship models that improve the average and regress the slice that matters. Slice-worst is a floor, not a secondary.
- **Floors that read like goals.** "Calibration should be good" is a wish. "ECE ≤ 0.03 on the golden set, bootstrap 95% CI reported" is a floor.
- **Slice sprawl.** A slice for every attribute makes every launch look worse without informing a decision. Each slice has a reason sentence, or it goes.
- **Protected-attribute slices without a policy.** If attributes cannot be stored, the registry documents the proxy / sampled-audit protocol explicitly. Silent omission is worse than acknowledged proxy.
- **Vibe noise bands.** "A 1% move is probably noise" is not a band. Bootstrap or seed-variance the metric on the eval set, and publish the number.
- **Guardrails as prose.** "We generally require improvement without regression" is not a rule set. An ordered first-match policy with numeric tolerances is.
- **Waivers that live forever.** A waiver without an expiry is a policy change. Name it as one or kill the regression.
- **Dashboard-only harness.** A harness that renders beautifully and does not emit a decision pushes the gate onto meeting energy. The harness returns `pass` / `fail`; humans override, with a waiver.
- **Headline-flat = ship.** G5 (pass-but-hold) is deliberate. Flat on the eval set is not a ship signal; it is a product decision, and the harness must mark it that way.
- **Missing adversarial slice.** For any model with a safety or policy surface, the harness must stress-test it. OWASP ML Top 10 and MITRE ATLAS name the relevant tactics; translate at least one into an eval slice.
- **No freshness plan.** An eval set treated as immutable eventually measures a world the model is no longer deployed in. Freshness is scheduled work, not an aspiration.
- **Changing the policy to fit the candidate.** If a tolerance is widened to pass a specific launch, the change is logged, dated, and reviewed. The next launch should not benefit from a quietly-expanded tolerance.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE function underpins the floor-metric and slice-reporting discipline; MEASURE 2.x subcategories ground the fairness and safety floors.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source for the adversarial-slice tactics (input manipulation, data poisoning) that the harness stresses.
- MITRE ATLAS — <https://atlas.mitre.org/>. Source for named adversarial tactics used in the adversarial / stress slice design.

Local context (authoritative for the harness taxonomy and the guardrail ordering):

- Paired lesson: `lessons/mod-305-advanced-evaluation/exercises/exercise-01-offline-harness-with-slices-and-guardrails.md`.
- Module README: `modules/mod-305-advanced-evaluation/README.md`.
- Related exercise (how the harness feeds online tests): `modules/mod-305-advanced-evaluation/exercise-02-shadow-and-canary-deploy-design/SOLUTION.md`.
- Related exercise (ML Test Score review packet, which audits the harness itself): `modules/mod-305-advanced-evaluation/exercise-03-ml-test-score-review-packet/SOLUTION.md`.
