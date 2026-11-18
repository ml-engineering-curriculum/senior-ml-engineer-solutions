# exercise-04 — Causal Inference When A/B Is Impossible (Reference Solution)

Paired exercise: `lessons/mod-306-experimentation-at-scale/exercises/exercise-04-causal-inference-when-ab-is-impossible.md`.

## 1. Solution overview

This exercise produces `causal-design.md` — a written design that names, for one intervention a team cannot cleanly A/B test, the quasi-experimental method chosen, the assumptions that method requires, the diagnostics that stress those assumptions, the sensitivity analysis that bounds the conclusion when assumptions bend, and the reporting contract that keeps a causal claim from leaking into the organisation as if it were from a trusted A/B. The artefact has six parts:

1. A **problem frame** that names why A/B is impossible or inappropriate for this case (network effects, one-sided launch, ethics / safety, policy change, pre-existing switch flip, two-sided marketplaces).
2. A **method selection rubric** — which quasi-experimental methods are candidates (Difference-in-Differences, Synthetic Control, Regression Discontinuity, Instrumental Variables, Switchback / interleaved, Propensity-score matching), the data each requires, and how the team picks one.
3. A **design** — the specific estimator, the data window, the unit of analysis, the comparison group construction, and the statistical model.
4. An **assumption and diagnostic suite** — for each method, the identifying assumptions plus the tests that would reject them, and what the design does if they are rejected.
5. A **sensitivity and robustness plan** — the bounds on the effect under plausible deviations, so the readout survives an audit that disputes a single assumption.
6. A **reporting contract** — how the result is communicated inside the organisation, with the identifying-assumption language and the sensitivity bounds surfaced so downstream readers do not treat the number as an A/B readout.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric.

The exercise exists because the first lever when A/B is unavailable is not "run a worse A/B anyway"; it is "commit to a quasi-experimental method whose assumptions you can defend." A causal claim with no stated identifying assumption is a correlational story being shipped as a decision. The NIST AI RMF MEASURE function asks for measurements that are valid for their intended use; the MANAGE function asks that risk be documented with its uncertainty. This design packet is both, scoped to one intervention.

## 2. Worked answer / implementation

### 2.1 Problem frame

Six lines:

- **Intervention being studied.** The policy, feature, model, or system change. Pinned to a version / effective date. "The pricing update" is not an intervention; "the 2026-10-15 shift of the free-tier rate limit from 100 to 50 requests/day for new signups in the US locale" is.
- **Why a clean A/B is unavailable.** One of:
  - *Network / spillover* — treatment units affect control units (marketplace liquidity, search ranking, social-graph exposure). Randomisation at the user level is contaminated.
  - *One-sided launch* — the intervention was shipped before an experiment was planned, or is not reversible.
  - *Ethics or safety* — withholding the treatment from a control arm is not acceptable (safety intervention, legally-required change).
  - *Policy / platform change* — a single-time step change affecting everyone (price change, deprecation, compliance update).
  - *Economic infeasibility* — the sample size required for a per-user A/B would be larger than the available population.
  - *Operational infeasibility* — the system cannot route per-user differently (shared index, global rate-limiter).
- **Decision the study feeds.** "Keep the policy," "roll it back," "extend it," "regulator attestation." The decision determines the bar for identifying assumptions and sensitivity width.
- **What a wrong answer costs.** Named explicitly. If overclaiming a null costs revenue and overclaiming a positive costs trust, the design leans toward whichever error is cheaper to make.
- **Available data.** Pre-period length, granularity of outcomes, availability of comparison units, instrument availability if any. The method rubric (§2.2) depends on this.
- **Last reviewed.** A date within the design's review cadence.

### 2.2 Method selection rubric

Score the problem on the axes each method cares about. Each method has a short description, the data it requires, and the headline identifying assumption.

**M1. Difference-in-Differences (DID).** Compare the pre-to-post change in the treated unit(s) against the pre-to-post change in an untreated comparison group. Headline assumption: **parallel trends** — absent treatment, the outcomes in treated and comparison groups would have moved in parallel. Data required: a long enough pre-period to show parallel trends, a comparison group that did not receive the treatment and is not spilled-over.

**M2. Synthetic Control.** Construct a weighted combination of comparison units that reproduces the treated unit's pre-period trajectory, then compare the actual post-period outcome against the synthetic counterfactual. Headline assumption: **the convex combination is a good pre-period match** and the comparison pool is not treated by a similar intervention. Data required: multiple comparison units with a rich pre-period. Especially useful when the treated unit is a single market, region, or product line.

**M3. Regression Discontinuity (RDD).** When an intervention is applied above or below a sharp cutoff on a running variable (e.g., accounts with > 50 sessions get the new feature), compare units just above vs. just below the cutoff. Headline assumption: **units near the cutoff are comparable except for treatment**, and the cutoff was not manipulated. Data required: a sharp (or fuzzy, with instrument) rule and enough mass near the cutoff.

**M4. Instrumental Variables (IV).** When an instrument `Z` affects treatment take-up `D` but has no direct effect on outcome `Y` except through `D`, estimate the local average treatment effect via two-stage least squares. Headline assumptions: **relevance** (`Z → D` is strong), **exclusion** (`Z → Y` only through `D`), **monotonicity** (`Z` moves `D` in the same direction for everyone). Data required: a defensible instrument — rare and precious.

**M5. Switchback / interleaved rollouts.** For marketplaces or systems with network effects, alternate the intervention across time windows (hour, day) across the whole population. Cross-time contamination is still possible but is bounded by the switch cadence. Headline assumption: **no between-period carryover beyond a declared bound**. Data required: the ability to toggle treatment at a cadence shorter than the outcome-attribution window.

**M6. Propensity-Score Matching / Weighting.** Match or reweight treated and untreated units to balance observed covariates and estimate the average treatment effect on the treated (ATT) under **unconfoundedness** (conditional on observed covariates, treatment is as-if random). Weakest of these methods on a hostile assumption; useful when nothing else applies.

Pre-registered collapse rules (first match wins):

```
C1 (network / spillover):
  → Switchback (M5) if cadence is feasible; otherwise Synthetic Control (M2)
  at market-level.

C2 (one-sided shipped change, single market or product):
  → Synthetic Control (M2) if a plausible donor pool exists; otherwise DID
  (M1) if a clean comparison group exists.

C3 (sharp eligibility cutoff):
  → RDD (M3). Fuzzy-RDD if take-up is partial at the cutoff.

C4 (encouragement / assignment with incomplete compliance):
  → IV (M4) if the instrument is defensible; otherwise ITT-only via DID on
  the encouragement signal.

C5 (ethics / safety withhold):
  → Prefer Synthetic Control or DID at aggregate level; propensity matching
  (M6) only as supporting evidence.

C6 (nothing above applies):
  → Propensity matching (M6) with pre-registered sensitivity bounds, labeled
  as weakest-evidence.

C7 (none of the above, no defensible method):
  → DO NOT publish a causal claim. Publish a descriptive readout and name
  what is missing. A story is not a causal readout.
```

### 2.3 The design

For the chosen method the design records, before the analysis runs:

- **Estimator and specification.** The equation, in terms of the outcome, the treatment indicator, the fixed effects (where applicable), the covariates, and the error structure. Pre-registered; not fit-to-taste.
- **Unit of analysis.** Market, user, account, cohort — match the level at which treatment varies. A spillover-contaminated user-level analysis on a market-level treatment is the common error here.
- **Data window.** Pre-period length (long enough to demonstrate the identifying assumption), post-period length (long enough to see the outcome move but short enough to avoid confounding from later launches).
- **Comparison group / donor pool construction.** Which units are eligible, which are excluded, and why. Spillover-contaminated units are excluded explicitly.
- **Covariates.** The observed variables adjusted for, with the decision rule for their inclusion (pre-committed or data-driven; if data-driven, the procedure is pre-registered).
- **Standard errors.** Clustered at the right level — unit for DID, placebo-permutation inference for Synthetic Control, robust / HC-3 for RDD, cluster-robust for IV, cluster or weighted-bootstrap for propensity methods.
- **Dependent analyses.** Any additional analyses (subgroups, heterogeneity) that will be reported, with their multiple-testing corrections.

### 2.4 Assumption and diagnostic suite (per method)

Each method has a small set of tests that either support or reject its identifying assumption. The design pre-registers which diagnostics will run and the rule each one implies.

**DID — Parallel trends.**

- Pre-period graphical check and formal joint test that the treated-vs-comparison trends have equal slopes. A rejected pre-trend test invalidates the DID; the design names the fallback.
- Event-study (dynamic DID) if the treatment timing varies across units, showing the coefficient trajectory across lags and leads. Non-zero pre-period leads reject parallel trends.
- Placebo tests on a pre-treatment "fake" cutoff.

**Synthetic Control — Pre-period fit and placebo.**

- RMSPE in the pre-period must be small relative to the post-period gap; a design with a bad pre-fit cannot claim a post-period effect.
- Placebo tests: run the same synthetic-control procedure for every donor unit and compare the treated unit's post-period gap to the distribution of placebo gaps.
- Leave-one-out: drop each donor; the estimate should not depend on any single donor.

**RDD — Continuity and no manipulation.**

- Density test (McCrary) for a jump in the running-variable density at the cutoff; manipulation of the running variable invalidates the design.
- Covariate smoothness: pre-determined covariates should not jump at the cutoff.
- Bandwidth sensitivity: the estimate should be stable across a plausible range of bandwidths.
- Donut-hole (exclude units in a small window around the cutoff) to check for heaping.

**IV — Relevance, exclusion, monotonicity.**

- First-stage F-statistic on `Z → D`; weak-instrument region is a hard fail (rule-of-thumb F > 10 is commonly cited but domain-dependent; the design names its own numeric bar).
- Reduced-form check: `Z → Y` should move in the direction the structural model predicts.
- Overidentification test when multiple instruments are available.
- Narrative defence of exclusion (code-level or subject-matter reasoning that `Z` does not affect `Y` by any path other than through `D`). Exclusion is an assumption, not a test; the design names who defended it and how.

**Switchback — No between-period carryover.**

- Within-period stabilisation: outcomes within a period should stabilise before period end; carryover from the previous period's treatment manifests as an early-period trend.
- Washout windows between periods if needed.
- Varying switch cadence as a sensitivity: the estimated effect should not change with the cadence.

**Propensity matching — Covariate balance.**

- Standardised mean differences across treated and matched-comparison on observed covariates; a declared tolerance (e.g., SMD < 0.1) gates the match quality.
- Overlap / common-support check: units with extreme propensity scores are trimmed or flagged.
- Love plot of balance before vs after weighting.
- Rosenbaum bounds or E-value as a sensitivity to the un-measured confounder.

**Shared diagnostics (all methods).**

- **Negative controls.** Outcomes that should not be affected by the treatment are checked for an effect; if they move, confounding is present.
- **Falsification tests.** The analysis re-run on a pre-treatment "fake" intervention date; a non-null estimate is a red flag.
- **Heterogeneity.** Pre-committed subgroups; results surface heterogeneity without making it a primary without correction.

### 2.5 Sensitivity and robustness plan

Every causal readout carries bounds on the effect under plausible deviations from the identifying assumptions. The bounds are reported, not hidden.

- **DID:** a worst-case parallel-trend deviation (e.g., assume pre-period trend extrapolates into the post-period) widens the confidence interval by the trend's magnitude; report the adjusted interval.
- **Synthetic Control:** the placebo-distribution-based p-value is reported alongside the point estimate.
- **RDD:** range of estimates across a declared bandwidth grid.
- **IV:** Anderson–Rubin confidence interval for weak-instrument-robust inference.
- **Propensity methods:** Rosenbaum bound or E-value — the confounder strength that would need to exist to explain away the result. If the required confounder is implausible, the result is robust; if plausible, the readout says so.

The decision threshold is set with the sensitivity bounds in mind. "The intervention raised outcomes by Δ" is a claim; "the intervention raised outcomes by Δ ± sensitivity width, with the width driven by the parallel-trends assumption" is the claim a reader can act on.

### 2.6 Reporting contract

Every causal readout is labeled as such and carries the identifying assumption up front. Minimum fields:

```yaml
causal_readout:
  method: synthetic_control
  intervention: free_tier_rate_limit_reduction_2026_10_15
  unit_of_analysis: country_market
  pre_period: 2026-04-15 to 2026-10-14
  post_period: 2026-10-15 to 2026-11-14
  donor_pool_size: 24
  identifying_assumption: >
    "The weighted combination of donor markets reproduces the treated market's
    pre-period trajectory, and no donor market received a comparable rate-limit
    change during the study window."
  pre_period_rmspe: 0.012
  post_period_gap: -0.047                  # on normalised scale
  placebo_p_value: 0.018
  leave_one_out_range: [-0.055, -0.040]
  sensitivity:
    rosenbaum_or_similar: >
      "An unobserved confounder would need to shift the pre-period trajectory
      by at least X to explain the post-period gap."
  robustness:
    negative_controls: { status: clean }
    falsification_pre_date: { status: clean, placebo_date: 2026-08-15 }
  caveats: >
    "This is a quasi-experimental estimate, not an A/B result. The pre-period
    RMSPE indicates a good fit; interpret the effect as the market-level change
    under the stated identifying assumption."
  reviewed_by_role: experimentation_owner
  last_reviewed: 2026-11-18
```

Rules the contract enforces:

- The word **"experiment"** is reserved for randomised studies. A causal readout is a **"quasi-experimental estimate"** — the distinction is load-bearing for the downstream reader.
- The identifying assumption is written in plain language at the top of the readout, not footnoted.
- The sensitivity bounds are the headline, not an appendix. A point estimate without a bound is not a shippable causal readout.
- A failed diagnostic (parallel trends rejected, weak instrument, bad pre-fit) kills the readout; the design must name the fallback, not override.

### 2.7 Illustrative worked example (not a real system)

> **Illustrative feature** — a hypothetical free-tier rate-limit reduction shipped on 2026-10-15 in the US market, with the UK, DE, FR, JP, AU, CA markets untreated during the study window. This example is a composite for calibration; do not treat it as a real system or incident.

Problem frame:

- Intervention: free-tier rate limit cut from 100 to 50 requests/day for new US signups, effective 2026-10-15.
- A/B unavailable: policy change applied uniformly to the US market; no per-user withholding was operationally feasible and legal review required all US users receive the same policy during the trial.
- Decision fed: whether to keep the lower limit in the US and whether to extend to other markets.
- Outcome: paid-tier conversion rate per new signup within 30 days.
- Available data: 6 months of pre-period daily conversion by market; 6 untreated donor markets.

Method selection: C2 applies (one-sided shipped change in a single market with a plausible donor pool). Synthetic Control (M2) is selected; DID is noted as the sensitivity fallback if the synthetic pre-fit is poor.

Design:

- Estimator: Synthetic Control weighting six donor markets to reproduce US pre-period trajectory.
- Unit of analysis: market (country).
- Pre-period: 2026-04-15 to 2026-10-14 (26 weeks). Post-period: 2026-10-15 to 2026-11-14 (4 weeks).
- Donor pool: UK, DE, FR, JP, AU, CA. Excluded: IN (concurrent pricing change on 2026-09-10; spillover-contaminated).
- Covariates in pre-period match: weekly conversion rate, weekly new-signup volume, weekly 7-day active rate.
- Inference: placebo-permutation across all donor markets.

Diagnostics:

- Pre-period RMSPE: 0.012 (small relative to post-period gap of 0.047).
- Placebo: US post-period gap is in the top 2 of 7 markets by absolute size; one-sided placebo p ≈ 0.018 (`2/7` is the smallest achievable with this donor pool — a limitation named explicitly in the readout).
- Leave-one-out: removing any single donor, the estimate stays in [-0.055, -0.040].
- Negative controls: non-pricing-sensitive engagement metrics (daily session minutes) do not move at the intervention date.
- Falsification: synthetic control estimated for the same markets with a "fake" intervention date of 2026-08-15 yields a gap of 0.004 (clean).

Sensitivity: the Abadie-style placebo-based p-value is the primary uncertainty report; a one-period-shift parallel-trend robustness check gives a worst-case post-period gap of -0.038.

Readout: the US market saw a quasi-experimental estimate of a 4.7-percentage-point reduction in paid-tier conversion at 30 days under the identifying assumption that no donor market received a comparable policy change during the study window. The placebo-based p-value is 0.018, bounded by the donor-pool size. The read is: do not extend the lower rate limit to other markets without a decision from product leadership on a 3–5 pp conversion cost.

## 3. Validation steps

Walk the design through this checklist:

1. **Problem frame names the reason A/B is unavailable** with a specific category (network, one-sided, ethics, policy, infeasibility).
2. **Method is chosen by the pre-registered rubric** (C1–C7), not by convenience. The choice is dated and reviewed.
3. **Identifying assumption is named in one plain-language sentence.**
4. **Pre-period is long enough** to stress the identifying assumption (e.g., ≥ 6 months for market-level SC).
5. **Spillover-contaminated units are excluded** from the comparison group and the exclusion is logged.
6. **Diagnostics run for the chosen method** are the pre-registered ones from §2.4.
7. **Sensitivity bound is reported in the headline**, not in an appendix.
8. **Readout labels the result "quasi-experimental estimate," not "experiment."**
9. **Failed diagnostic triggers the pre-registered fallback**, not a workaround.
10. **Negative controls and falsification tests run** even if they would not have been required by a stricter interpretation of the method.
11. **Last-reviewed date within the design's declared cadence.**

Static checks on `causal-design.md`:

```bash
grep -ciE "parallel trends|pre-period fit|exclusion|unconfoundedness" causal-design.md  # identifying assumption stated
grep -ciE "placebo|falsification|negative control"                    causal-design.md  # diagnostics present
grep -ciE "sensitivity|rosenbaum|e-value|anderson-rubin"              causal-design.md  # sensitivity named
grep -ciE "quasi-experimental"                                         causal-design.md  # labeling discipline
grep -ciE "did|synthetic control|rdd|iv|switchback|propensity"         causal-design.md  # method named
grep -ciE "nist ai rmf"                                                causal-design.md  # standard cited
grep -ciE "last reviewed"                                              causal-design.md  # freshness
```

## 4. Rubric / review checklist

- [ ] Problem frame identifies a specific reason A/B is unavailable (not a generic "cannot A/B").
- [ ] Method chosen via the rubric; alternatives and the reason this one won are documented.
- [ ] Estimator, unit of analysis, and data windows are pre-registered.
- [ ] Comparison group / donor pool construction excludes spillover-contaminated units, with reasons.
- [ ] Identifying assumption stated in one plain-language sentence at the top of the readout.
- [ ] Diagnostic suite covers at least three method-appropriate tests (e.g., pre-trend + placebo + negative control for DID; pre-fit + placebo + leave-one-out for SC).
- [ ] Sensitivity bound is the headline number alongside the point estimate.
- [ ] Reporting contract enforces "quasi-experimental estimate" language and surfaces the identifying assumption.
- [ ] Failure-fallback path is explicit: if parallel trends reject, what runs next; if weak instrument, what runs next.
- [ ] Negative controls and falsification tests run and are reported.
- [ ] NIST AI RMF MEASURE cited for measurement-validity grounding; MANAGE cited for risk-with-uncertainty framing.
- [ ] For systems with a safety or policy surface, OWASP ML Top 10 / MITRE ATLAS referenced where the intervention's security context matters.
- [ ] Last-reviewed date within the declared cadence.

Review-weighted points for grading:

- Problem frame and justification for abandoning A/B: 10%
- Method selection rubric and the chosen method's justification: 15%
- Design specification (estimator, unit, window, comparisons, covariates, SE structure): 20%
- Assumption-and-diagnostic suite (depth and method appropriateness): 20%
- Sensitivity and robustness plan (quantified bounds): 15%
- Reporting contract (language discipline, assumption surfacing, fallback path): 15%
- Standards alignment (NIST AI RMF MEASURE / MANAGE) and hygiene: 5%

## 5. Common mistakes

- **Running a user-level A/B on a network-effect intervention.** Spillover contaminates the control arm and the estimate is biased toward the null. Switchback or market-level synthetic control is the method; a bad A/B is worse than a good quasi-experiment.
- **No identifying assumption stated.** A result reported without "the assumption this leans on is X" is a correlational story. The assumption is the deliverable's anchor, not a footnote.
- **Parallel trends assumed but not shown.** DID without a pre-period plot or formal test is a two-point comparison that happens to involve a difference operation. Show the trend.
- **Spillover-contaminated comparison units kept in the pool.** A donor market that received a similar intervention mid-study contaminates synthetic control. Exclude explicitly and log the reason.
- **IV with a weak instrument.** First-stage F below the declared bar; the estimator becomes unstable and the inference is unreliable. Weak instruments are not a mild issue — they invalidate the method.
- **Propensity matching with no overlap check.** If treated and comparison units do not share a common support on covariates, the match is extrapolation disguised as adjustment. Trim or flag.
- **Sensitivity in the appendix.** A point estimate shipped without bounds becomes a precise number at the executive readout, and the method's uncertainty evaporates. The bound is the headline.
- **"Experiment" used for a quasi-experimental study.** Downstream readers treat A/B readouts and causal estimates differently; language discipline is not pedantry. Reserve "experiment" for randomised studies.
- **Diagnostic trip rationalised.** A failed pre-trend test survives because a secondary analysis "mostly" supports the conclusion. The design pre-registers the fallback; it does not pre-register exceptions.
- **Falsification and negative controls skipped.** These are the cheap sanity checks that catch the quiet confounder; if they fail, the method is probably wrong.
- **One method, no fallback.** Any design that cannot name what runs when its headline diagnostic fails is a design with a single point of failure. The fallback is named up front.
- **Reading quasi-experimental results as causal with A/B-level confidence.** A quasi-experimental estimate is a causal claim under an assumption. The organisation needs to see both parts or it will act on neither.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE function grounds the requirement that a measurement adjustment (and the causal method behind it) be validated for its intended use; MANAGE function grounds the reporting of risk under uncertainty, which is exactly what a sensitivity bound communicates.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source for the framing when the intervention is a security or abuse-mitigation change and the measurement must cover an adversary that adapts to it.
- MITRE ATLAS — <https://atlas.mitre.org/>. Source for named tactics relevant to interventions whose causal effect must be measured on adversarial as well as benign populations (e.g., data provenance, feature manipulation).

Method references (public, canonical textbook-level):

- Angrist and Pischke, *Mostly Harmless Econometrics* (Princeton, 2009). Standard reference for DID, IV, and RDD design and assumptions. Cited as a method reference.
- Abadie, "Using Synthetic Controls: Feasibility, Data Requirements, and Methodological Aspects" (Journal of Economic Literature, 2021). Canonical description of Synthetic Control; cited as a method reference.
- Imbens and Rubin, *Causal Inference for Statistics, Social, and Biomedical Sciences* (Cambridge, 2015). Standard reference for propensity-score methods and sensitivity analysis (Rosenbaum bounds); cited as a method reference.

Local context (authoritative for the method rubric, reporting contract, and fallback discipline):

- Paired lesson: `lessons/mod-306-experimentation-at-scale/exercises/exercise-04-causal-inference-when-ab-is-impossible.md`.
- Module README: `modules/mod-306-experimentation-at-scale/README.md`.
- Related exercise (A/B sizing and ramp that this design replaces when A/B is unavailable): `modules/mod-306-experimentation-at-scale/exercise-01-power-mde-and-ramp-plan/SOLUTION.md`.
- Related exercise (trust diagnostics; several ideas carry over, especially segment balance and A/A sanity): `modules/mod-306-experimentation-at-scale/exercise-02-srm-and-trust-diagnostics/SOLUTION.md`.
- Related exercise (variance reduction; CUPED-style pre-period covariates are a common input to propensity-weighted causal designs): `modules/mod-306-experimentation-at-scale/exercise-03-variance-reduction-with-cuped/SOLUTION.md`.
