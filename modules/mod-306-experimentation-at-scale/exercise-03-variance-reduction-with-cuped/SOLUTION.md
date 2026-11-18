# exercise-03 — Variance Reduction With CUPED (Reference Solution)

Paired exercise: `lessons/mod-306-experimentation-at-scale/exercises/exercise-03-variance-reduction-with-cuped.md`.

## 1. Solution overview

This exercise produces `cuped-design.md` plus a small runnable artefact (`cuped.py`) — a written design and the code that applies CUPED (Controlled-experiment Using Pre-Experiment Data) as the variance-reduction step for one experiment, so a readout achieves the pre-registered power without requiring a longer run. The artefact has six parts:

1. A **method frame** that pins down what CUPED is, why variance reduction matters, and the precondition the method needs (a pre-experiment covariate correlated with the primary metric).
2. A **covariate selection rubric** — which pre-experiment variables are eligible, which are disqualified, and how the chosen covariate is validated.
3. A **CUPED adjustment** — the estimator, the point at which θ is estimated, the resulting adjusted metric, and the re-stated sample-size / runtime numbers.
4. A **assumption and diagnostic suite** — the checks that confirm CUPED is valid on this experiment (pre-treatment covariate, θ stability, residual balance) and the trip rules when a check fails.
5. A **reporting contract** — how the readout reports both unadjusted and adjusted estimates, the reduction achieved, and what the reader must still inspect.
6. A **failure and fallback plan** — what to do when CUPED does not help (weak correlation), when the covariate drifts post-launch, or when the method would mask a real effect.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric.

The exercise exists because the first lever to pull when a plan's runtime exceeds the ceiling (exercise-01 §2.3) is not "widen the MDE." It is "reduce the variance." CUPED is the standard pre-experiment-covariate adjustment used widely in industry since Deng et al. (2013), and when its preconditions hold it reduces required sample size materially without changing α or power. The failure modes are quiet — a covariate that looks pre-treatment but is contaminated, a θ fit on post-exposure data, a residual balance check skipped — and each one turns a variance-reduction win into a biased estimate. NIST AI RMF MEASURE 2.x asks that measurement adjustments are themselves validated; this packet is that validation.

## 2. Worked answer / implementation

### 2.1 Method frame

Six lines, in this order:

- **What CUPED is.** A linear adjustment to the primary metric that subtracts the component explained by a pre-experiment covariate, leaving an outcome with the same expectation under treatment but strictly smaller variance when the covariate is correlated with the outcome. Published in Deng et al., "Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data" (WSDM 2013). Textbook since; not an exotic technique.
- **Why it is used.** Equal or better power at the same α for a smaller sample, or equivalently, a smaller detectable effect at the same sample. The plan's runtime (exercise-01 §2.3) was the input; CUPED is the compress.
- **Precondition.** A pre-experiment covariate `X` that correlates with the primary metric `Y` on the unit of analysis. Correlation sign does not matter; magnitude does. A near-zero correlation means CUPED is a no-op; a strong correlation is where the gain lives.
- **Scope.** One primary metric, one covariate, one experiment. Multi-covariate generalisations (CUPAC, regression adjustment) are called out but out of scope; design one lever at a time.
- **Where it lives.** In the experimentation platform's metric pipeline, applied uniformly to both arms. Not computed in the analyst's notebook on a per-readout basis.
- **Last reviewed.** A date, within the design's review cadence.

### 2.2 Covariate selection rubric

**Eligible covariates.**

- The same metric `Y`, measured on the same unit of analysis, computed on a pre-experiment window of similar length (e.g., 28 days before enrolment for a per-user 28-day metric). This is the *usual* CUPED choice and the strongest covariate for stable, user-level metrics: engagement, spend, retention proxy.
- A closely-related metric with higher pre-period stability (e.g., sessions-per-week as a covariate for engagement-per-week).
- A pre-experiment segment indicator that correlates with `Y` (tenure bucket, prior-plan tier).

**Disqualified covariates.**

- Any feature measured during or after exposure. "Pre-experiment" is a hard constraint; a covariate built from data that includes enrolment is not a covariate, it is the treatment effect in disguise.
- Treatment-affected features (page-view count on the surface the experiment touches, if the surface is visible before randomisation and still materially altered by the treatment path). If the surface itself changes exposure, the pre-exposure measure may still be contaminated by anticipatory selection; the plan names the test that rules this out.
- Features with high missingness in a slice of the population. CUPED on a covariate missing for 20% of users creates a sample split where adjusted results are reported on 80% and unadjusted on the rest — a readout pathology.
- Features that will change definition during the experiment window. A covariate that depends on a product surface that itself ships mid-experiment is not stationary.

**Validation of the chosen covariate.**

Before the experiment starts:

- **Correlation estimate.** On a pre-experiment window, compute the empirical correlation ρ(X, Y) on the randomisation-unit level. If `|ρ| < 0.1`, the method's gain is negligible and the plan declines CUPED for this metric; the reason is logged.
- **Stationarity check.** Split the pre-experiment window in half; verify that ρ in each half is comparable. A covariate whose correlation with `Y` swings materially week-to-week is not a good candidate.
- **Pre-treatment confirmation.** Confirm the covariate's definition sources no post-exposure data. A review of the SQL (or the feature-store spec) by a second engineer is the check; the owner-and-reviewer are both logged.
- **Balance expectation.** Confirm that across many simulated A/A splits on the pre-period, the covariate's two-arm difference is centred at zero. This is the A/A sanity (exercise-02 D3) applied to the covariate.

### 2.3 The CUPED adjustment

**Estimator.**

Let `Y_i` be the primary metric for unit `i`, `X_i` the pre-experiment covariate, `X̄` the overall mean of `X`, and θ the regression coefficient:

```
θ = Cov(Y, X) / Var(X)
```

Define the adjusted outcome:

```
Y_i^{cuped} = Y_i − θ · (X_i − X̄)
```

The adjusted-metric arm difference `Ȳ_T^{cuped} − Ȳ_C^{cuped}` has the same expectation as the unadjusted difference and smaller variance by approximately a factor of `1 − ρ²`, where ρ is the correlation between `Y` and `X`. The required sample size for the same power and MDE scales approximately by the same factor.

**Where θ is estimated.**

- θ is estimated once, on data that does not depend on the arm — either the pre-experiment window or the pooled post-exposure data of both arms. Fitting θ on the treatment arm alone would smuggle the treatment effect into the adjustment.
- θ is fixed for the duration of the experiment; it does not update per interim readout. A θ that moves with the readout is a p-hacking surface.
- θ is stored with the experiment configuration, alongside the covariate definition and the window.

**Standard-error correction.**

The adjusted-metric variance estimator must use the correct degrees of freedom; a plug-in that ignores the θ-estimation step overstates precision by a tiny amount at experiment scale (a few thousand units or more) and by a material amount at small scale. The plan names whether θ is re-fit with a holdout, or whether the standard-error formula corrects explicitly.

**Re-stated sample size.**

From exercise-01, the per-arm sample size for a means-design was:

```
n_per_arm ≈ 2 · (z_{α/2} + z_β)² · σ² / Δ²
```

With CUPED the variance drops to `σ² · (1 − ρ²)` so:

```
n_per_arm^{cuped} ≈ n_per_arm · (1 − ρ²)
```

With ρ = 0.5, `1 − ρ² = 0.75` → 25% sample-size reduction. With ρ = 0.7, `1 − ρ² = 0.51` → ~49% reduction. These are the method's accounting; actual gain is bounded by how stationary the covariate–outcome relationship is on the live data.

### 2.4 Assumption and diagnostic suite

Pre-registered, blocking. A trip pauses the readout.

**A1. Pre-treatment property.** The covariate is deterministically pre-exposure. Evidence: SQL / feature-spec review, named reviewer, dated.

**A2. Covariate–outcome correlation on live pre-period.** `|ρ| ≥ 0.1` on the enrolled population's pre-experiment window. If below, CUPED is declined for this experiment (not failed — declined); the readout reports unadjusted. Rationale: the method's approximate gain is `1 − ρ²`; for `|ρ| < 0.1` the gain is ≤ 1%, within noise of the overhead.

**A3. Residual balance.** After adjustment, the arm difference on `X` itself must be within the pre-period's imbalance tolerance (exercise-02 D2). An arm difference on `X` after adjustment indicates either a randomisation problem (SRM — see exercise-02) or a covariate drift mid-experiment.

**A4. θ stability.** If the readout happens at multiple interim points (early-stopping design), θ re-estimated on each interim is compared against the registered θ; a swing beyond a pre-declared tolerance invalidates the interim. The declared tolerance is a numeric band on θ, not a feeling.

**A5. No reporting on partial cohorts.** Units with missing `X` are not adjusted. If the missing-covariate cohort exceeds a declared share (e.g., 5%), the plan either imputes with a declared method or declines CUPED for that run. Mixed reporting is not allowed.

**A6. Monotonicity sanity.** On a held-out historical A/A split, the CUPED-adjusted false-positive rate should match the nominal α. If not, θ estimation or the SE correction is wrong; fix before using CUPED for a decision.

### 2.5 Reporting contract

Every CUPED-adjusted readout carries a side-by-side block:

```yaml
cuped:
  covariate: user_attach_rate_pre_28d
  rho: 0.62
  theta: 0.73                         # fit on pre-period
  theta_source: pre_experiment_window # not treatment arm
  reduction_factor: 0.616             # 1 - rho^2
  readout:
    unadjusted: { delta: 0.0097, se: 0.0031, p: 0.0017 }
    adjusted:   { delta: 0.0101, se: 0.0019, p: 0.000006 }
  missing_covariate_share: 0.019
  residual_balance_delta_on_X: 0.0004
  assumptions:
    a1_pre_treatment: { status: clean, reviewer_role: experimentation_owner }
    a2_rho_threshold: { status: clean, value: 0.62 }
    a3_residual_balance: { status: clean }
    a4_theta_stability: { status: not_applicable_no_interim }
    a5_missing_share: { status: clean, value: 0.019 }
    a6_monotonicity: { status: clean, aa_fpr: 0.049 }
  overall: clean
  last_reviewed: 2026-11-10
```

The reporting contract requires that:

- Both unadjusted and adjusted estimates are reported. A reader must be able to see what the raw data said and what the adjustment did.
- The reduction factor is reported, not rounded away. If the adjusted estimate crosses the decision threshold and the unadjusted one does not, the reader must see the gap and the justification.
- A `cuped: not_used` block, with the reason, is required when CUPED is declined. The absence of adjustment is a positive statement, not a silent omission.

### 2.6 Illustrative worked example (not a real system)

> **Illustrative feature** — the `reco-v12` checkout recommendation experiment from `exercise-01` (per-user attach rate). This example is a composite for calibration; do not treat it as a real system or incident.

Covariate selection:

- Candidate: `user_attach_rate_pre_28d` — per-user attach rate over the 28 days immediately before enrolment. Same metric, same unit, lagged.
- Pre-experiment ρ estimated on a 28-day window: 0.62. Above the 0.1 threshold.
- Stationarity: split the window into two 14-day halves; ρ values 0.60 and 0.63. Within tolerance.
- Pre-treatment confirmation: SQL review by a second engineer, 2026-11-05. Source: cart-upsell event log, no post-exposure inputs.
- Missing-share: 1.9% of enrolled users have no pre-period activity. Below the 5% ceiling; those units are not adjusted and are reported in `missing_covariate_share`.

Re-planned sample size:

- Original (exercise-01 §2.3): n = 24,154 per arm.
- Reduction factor `1 − 0.62² = 0.6156`.
- CUPED-sized n: `24,154 · 0.6156 ≈ 14,869` per arm → ~38% reduction. Runtime at 40,000 qualified users/day, 50/50: `≈ 14,869 · 2 / 40,000 ≈ 0.74` days plus dwell.

θ:

- `θ = Cov(Y, X) / Var(X)` fit on the pre-experiment window: 0.73. Stored with the experiment config.

Readout:

- Unadjusted: `Δ = 0.0097` (0.97 pp), SE 0.0031, p = 0.0017.
- Adjusted: `Δ = 0.0101` (1.01 pp), SE 0.0019, p ≈ 6×10⁻⁶.
- Both arms sign-identical. Adjusted estimate is above the pre-registered 1 pp MDE; unadjusted is marginal.
- Decision: the plan's pre-registration (exercise-01 §2.5) accepted CUPED-adjusted estimates as primary. Win declared. Both unadjusted and adjusted shown in the readout; the launch note records that the adjustment moved the readout from marginal to decisive and names the ρ and θ.

### 2.7 Runnable sketch: `cuped.py`

```python
# cuped.py
"""CUPED adjustment on a per-unit pandas-style dataframe.

The function is deliberately a single estimator. The design document decides
when to use it, which covariate, and what guardrails to attach; the code just
implements the arithmetic.
"""

from dataclasses import dataclass


@dataclass(frozen=True)
class CupedFit:
    theta: float
    mean_x: float
    rho: float
    reduction_factor: float   # 1 - rho^2


def fit_theta(y, x) -> CupedFit:
    """Estimate theta, mean_x, and rho on a pre-experiment series.

    y, x are parallel sequences of floats on the same randomisation unit.
    """
    y = list(y); x = list(x)
    if len(y) != len(x) or len(y) < 2:
        raise ValueError("y and x must be aligned and non-trivial in length")
    n = len(y)
    mean_y = sum(y) / n
    mean_x = sum(x) / n
    cov = sum((yi - mean_y) * (xi - mean_x) for yi, xi in zip(y, x)) / (n - 1)
    var_x = sum((xi - mean_x) ** 2 for xi in x) / (n - 1)
    var_y = sum((yi - mean_y) ** 2 for yi in y) / (n - 1)
    if var_x <= 0:
        raise ValueError("covariate has zero variance; CUPED would be a no-op")
    theta = cov / var_x
    rho = cov / ((var_x * var_y) ** 0.5) if var_y > 0 else 0.0
    return CupedFit(theta=theta, mean_x=mean_x, rho=rho, reduction_factor=1 - rho * rho)


def apply_cuped(y, x, fit: CupedFit) -> list[float]:
    """Return CUPED-adjusted outcome Y - theta * (X - mean_x). Units with X=None
    are returned as the raw Y with no adjustment; the caller is responsible for
    deciding whether to report them.
    """
    y = list(y); x = list(x)
    if len(y) != len(x):
        raise ValueError("y and x must be aligned")
    return [yi if xi is None else (yi - fit.theta * (xi - fit.mean_x))
            for yi, xi in zip(y, x)]


if __name__ == "__main__":
    # Toy, two users, clean correlation.
    y_pre = [0.1, 0.9, 0.2, 0.7, 0.3, 0.8]
    x_pre = [0.1, 0.8, 0.2, 0.75, 0.25, 0.85]
    fit = fit_theta(y_pre, x_pre)
    print(f"theta={fit.theta:.3f} rho={fit.rho:.3f} reduction={fit.reduction_factor:.3f}")
    y_exp = [0.2, 0.95, 0.1, 0.78, 0.35, 0.88]
    x_exp = [0.15, 0.82, 0.18, 0.76, 0.28, 0.86]
    print(apply_cuped(y_exp, x_exp, fit))
```

The code's job is to make the design's claim ("adjusted outcome equals `Y − θ (X − X̄)`, θ fit on pre-period") reproducible. The platform wraps this with the missing-cohort handling (A5) and the standard-error correction.

## 3. Validation steps

Walk the design through this checklist:

1. **Covariate is pre-exposure.** The SQL or feature-store spec was reviewed by a second named role.
2. **ρ is estimated on a pre-experiment window** of adequate length and reported as a number, not a range.
3. **θ is fit on pre-experiment data or pooled post-exposure data**, never on treatment-only post-exposure.
4. **Sample-size recomputation is derived** from `1 − ρ²`, with the before and after both shown.
5. **Missing-covariate share is measured and reported.** Above 5%, CUPED is declined or imputation is explicit.
6. **Reporting shows unadjusted and adjusted side by side.** A reader can see what the adjustment did.
7. **θ is fixed for the experiment** and stored with config, not re-fit per interim.
8. **A/A monotonicity has been run** on a historical window to confirm the adjusted estimator's nominal α.
9. **Residual balance on `X`** is clean post-adjustment; a non-zero residual flags an upstream SRM.
10. **Declined-CUPED is a positive statement** in the readout, not a silent omission.
11. **Last-reviewed date within cadence.**

Static checks on `cuped-design.md`:

```bash
grep -ciE "theta|θ"                                   cuped-design.md   # estimator named
grep -ciE "rho|correlation"                           cuped-design.md   # precondition named
grep -ciE "pre-experiment|pre-treatment"              cuped-design.md   # window specified
grep -ciE "deng|cuped"                                cuped-design.md   # method cited
grep -ciE "unadjusted.*adjusted|side[- ]by[- ]side"   cuped-design.md   # reporting contract
grep -ciE "declined|not used"                         cuped-design.md   # decline path exists
grep -ciE "nist ai rmf"                               cuped-design.md   # standard cited
grep -ciE "last reviewed"                             cuped-design.md   # freshness
```

For `cuped.py`:

- Smoke test: on `y = x` input, `fit.theta == 1.0` and `apply_cuped` returns the mean-centred sequence.
- Property: `reduction_factor ∈ [0, 1]`.
- Property: `apply_cuped` leaves `y_i` unchanged when `x_i is None`.

## 4. Rubric / review checklist

- [ ] Method frame names CUPED as a pre-experiment-covariate adjustment and cites Deng et al. (2013) as the canonical reference.
- [ ] Precondition stated: covariate is pre-exposure and correlates with the primary metric.
- [ ] Covariate selection rubric names eligible choices, disqualified choices, and validation steps (correlation, stationarity, pre-treatment confirmation).
- [ ] Estimator formulae reproduced (θ, adjusted outcome, variance reduction `1 − ρ²`).
- [ ] θ is fit on pre-period or pooled data; the plan forbids treatment-arm-only θ.
- [ ] Sample-size recomputation uses `1 − ρ²` and is reported as the new runtime.
- [ ] Assumption suite covers A1 pre-treatment, A2 ρ threshold, A3 residual balance, A4 θ stability, A5 missing share, A6 monotonicity.
- [ ] Reporting contract requires side-by-side unadjusted and adjusted estimates with ρ and θ visible.
- [ ] Declined-CUPED path is explicit (`cuped: not_used` with reason).
- [ ] Integration with `exercise-02` D2 (pre-period balance) via residual-balance check is called out.
- [ ] Integration with `exercise-01` sizing via the recomputed runtime is called out.
- [ ] NIST AI RMF MEASURE 2.x cited for measurement-adjustment validation.
- [ ] Runnable `cuped.py` implements fit and apply with the documented behaviour for missing covariate.
- [ ] Last-reviewed date within the design's cadence.

Review-weighted points for grading:

- Method frame and precondition statement: 10%
- Covariate selection rubric (eligibility, disqualification, validation): 20%
- Estimator and recomputed sizing (correct formulae, numeric reduction): 20%
- Assumption suite A1–A6 with numeric thresholds: 20%
- Reporting contract (unadjusted / adjusted / declined): 15%
- Runnable artefact (`cuped.py`): 10%
- Standards alignment (NIST AI RMF MEASURE) and hygiene: 5%

## 5. Common mistakes

- **θ fit on treatment-only data.** The adjustment inherits the treatment effect and bias is baked in. θ is fit on pre-period or pooled.
- **Covariate that is not strictly pre-exposure.** A feature built from events that straddle enrolment is contaminated; a reviewer must have confirmed the window boundary.
- **Covariate with 20% missingness.** Mixed reporting across the adjusted and unadjusted cohort is not a readout. Impute with a declared method or decline.
- **ρ ≈ 0, CUPED applied anyway.** The method is a no-op; the overhead and extra moving parts buy nothing and create a surface for errors. Decline, log the reason.
- **θ re-fit at each interim.** A θ that moves is a free parameter at readout time — a p-hacking surface. θ is fixed with the experiment config.
- **Standard error reported without correction.** The adjusted-metric SE must account for θ estimation; the magnitude is small at experiment scale but non-zero. Name the correction or the holdout.
- **Only the adjusted estimate shown.** Hiding the unadjusted estimate loses the sanity check that the adjustment did not flip a sign. Report both.
- **Residual-balance check on `X` skipped.** A non-zero residual on `X` after adjustment indicates an upstream SRM; skipping this check makes CUPED a cover for a broken randomisation.
- **CUPED used to rescue a stalled launch.** If unadjusted is p = 0.6 and adjusted is p = 0.03, the adjustment is doing more than reducing variance — check A3 residual balance and A6 monotonicity, hard.
- **Multi-covariate generalisations introduced without a plan.** CUPAC, regression adjustment, and ML-based variance reduction (MLRATE) are out of scope for a single CUPED exercise; adding them silently hides which lever moved the readout.
- **"Not used" omitted silently.** The absence of CUPED is a statement; a readout that silently drops the adjustment hides a choice.
- **Covariate definition changes mid-experiment.** If the feature-store spec ships an update to the covariate during the run, θ no longer corresponds to the live data. Freeze the covariate for the duration.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE function 2.x underpins the validation discipline: a measurement adjustment is a measurement choice and must itself be validated for reliability.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source for the framing when the covariate pipeline becomes a data-integrity surface an adversary could exploit (e.g., feature-store poisoning that shifts θ).
- MITRE ATLAS — <https://atlas.mitre.org/>. Source for named tactics (data provenance, feature manipulation) relevant to the integrity of the pre-experiment covariate.

Method reference (public, canonical):

- Deng, Xu, Kohavi, Walker, "Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data" (WSDM 2013). Canonical description of CUPED; cited as a method reference, not a performance claim.

Local context (authoritative for the covariate selection, assumption suite, and reporting contract):

- Paired lesson: `lessons/mod-306-experimentation-at-scale/exercises/exercise-03-variance-reduction-with-cuped.md`.
- Module README: `modules/mod-306-experimentation-at-scale/README.md`.
- Related exercise (sizing that drives the runtime pressure CUPED is meant to relieve): `modules/mod-306-experimentation-at-scale/exercise-01-power-mde-and-ramp-plan/SOLUTION.md`.
- Related exercise (pre-period balance D2 is the same check CUPED's residual-balance assumption reuses): `modules/mod-306-experimentation-at-scale/exercise-02-srm-and-trust-diagnostics/SOLUTION.md`.
- Related exercise (methods used when neither A/B nor CUPED-A/B is feasible): `modules/mod-306-experimentation-at-scale/exercise-04-causal-inference-when-ab-is-impossible/SOLUTION.md`.
