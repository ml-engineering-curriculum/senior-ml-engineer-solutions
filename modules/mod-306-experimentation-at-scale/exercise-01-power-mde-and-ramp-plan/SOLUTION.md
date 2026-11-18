# exercise-01 — Power, MDE, And Ramp Plan (Reference Solution)

Paired exercise: `lessons/mod-306-experimentation-at-scale/exercises/exercise-01-power-mde-and-ramp-plan.md`.

## 1. Solution overview

This exercise produces `power-and-ramp.md` plus a small runnable artefact (`power.py`) — a written experiment design that names, for one proposed A/B test, the primary metric, the Minimum Detectable Effect (MDE) the team is willing to miss, the statistical power the design targets, the sample size and runtime those jointly imply, and the ramp plan that takes the experiment from 1% exposure to a decision without betting the service on a single step. The artefact has six parts:

1. An **experiment frame** that pins down what is being tested, what decision the result feeds, and which population is in scope.
2. A **metric panel** — the single primary metric, a small set of secondary metrics that explain movement in the primary, and the guardrail metrics that gate a rollback regardless of primary.
3. A **power and sample-size calculation** — baseline, variance assumption, MDE, significance level, power, two-sided default, sample size and the runtime that sample size implies at current traffic.
4. A **ramp plan** — the exposure schedule (1% → 5% → 25% → 50% / full), the dwell time per step, the pre-registered gates that must hold to advance, and the pre-registered rollback triggers.
5. A **pre-registered decision rule** — what statistical result (and what guardrail state) is required to declare the experiment a win, a loss, or inconclusive, and who has sign-off at each gate.
6. A **re-planning and failure plan** — what the team does when the effect is smaller than the MDE, when a guardrail trips, or when the required runtime makes the experiment impractical.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric that calibrates every learner's plan to the same bar.

The exercise exists because an A/B test that starts without a pre-committed MDE, power target, and ramp plan is not an experiment; it is a search for a conclusion. A plan written in advance makes the result auditable — you can tell whether a null result is "treatment is flat" or "we did not power the study to detect this" — and it keeps the ramp from being decided in whichever meeting the launch lead attends. NIST AI RMF MEASURE calls for measurements whose reliability and validity are themselves documented; a power and ramp plan is that documentation for one online test, and NIST AI RMF MANAGE grounds the ramp as a staged risk control.

## 2. Worked answer / implementation

### 2.1 Experiment frame

Six lines, in this order:

- **What is being tested.** The intervention: a new model version, a prompt change, a ranking policy, an incentive. Pinned to a version. "The new ranker" is not an experiment; "`reco-v12` replacing `reco-v11` on the homepage carousel for logged-in users" is.
- **Decision the experiment feeds.** Promote to default, roll back, extend runtime, enter a follow-up. The decision determines how tight the MDE and how strict the guardrails need to be — launch-gate experiments have a lower tolerance for ambiguity than directional reads.
- **Population and unit of analysis.** Who is in the experiment (eligibility rules) and the randomization unit (user, session, device, account). The unit of analysis should match the randomization unit; a per-session metric analyzed per user is a variance mistake the plan should pre-empt.
- **Assignment mechanism.** Hash function, salt, bucketing service version. Named here so later SRM diagnostics (exercise-02) are grounded.
- **Hypothesis.** One sentence, directional or two-sided, in the shape of "intervention X will change primary metric M by at least Δ compared to baseline." If the team cannot write this sentence, the experiment is not ready to start.
- **Last reviewed.** A date, within the plan's review cadence. Plans that sit unread for months get started on assumptions no one remembers endorsing.

### 2.2 Metric panel

Three tiers. All three are mandatory. The panel mirrors the offline-harness shape from `mod-305/exercise-01` so an offline promotion decision and an online decision use the same vocabulary.

**Primary (one).** The single metric the experiment moves or does not move. If the team cannot pick one, the hypothesis is not scoped; collapse the hypothesis before sizing the test. For an engagement experiment the primary might be "sessions per weekly active user"; for a reliability experiment it might be "p95 end-to-end latency"; for a safety experiment it might be "rate of policy-flagged responses per 1000 requests." The primary determines the variance model used in sizing.

**Secondary (two to four).** Metrics that *explain* a primary move. For a click-through rate primary: pages per session, scroll depth, revisit rate. Secondaries are not gates; they prevent the launch write-up from saying "clicks went up" without also saying what happened to the things clicks are supposed to cause.

**Guardrails (two to five).** Metrics that must not regress regardless of primary movement. Typical guardrails:

- **Latency.** p95 or p99 user-facing latency. A quality lift paid for in latency is a different trade.
- **Error rate.** 5xx rate, model inference error rate, downstream failure rate.
- **Business floors.** Revenue per user, retention proxy, support contact rate. The ones a product leader would roll back on even if primary went up.
- **Safety / policy.** For generative systems, OWASP ML Top 10-aligned prompt-injection success rate and MITRE ATLAS-aligned adversarial-input failure rate on a stress slice. NIST AI RMF MEASURE 2.x treats these as first-class measurements, not side checks.
- **Equity guardrails.** Worst-slice primary movement for named protected or high-value segments. A primary that moves on average and regresses a protected slice is a loss.

**Rule of the panel:** a win requires the primary to move beyond the MDE in the intended direction AND no guardrail to trip. A primary move with a tripped guardrail is a loss until either the guardrail regression is mitigated or a dated, named-role waiver is logged. Guardrails are declared with directions and numeric tolerances before the experiment starts; they are not negotiated at the readout.

### 2.3 Power, MDE, and sample size

**Inputs to pin down before computing anything.**

- **Baseline value of the primary**, from a recent, stable window (e.g., trailing 28 days), by the same unit of analysis the experiment uses. For a rate, this is the proportion; for a continuous metric, the mean. Pull the number from the same source the readout will use.
- **Variance of the primary**, by the same unit. For a proportion `p`, variance is `p(1-p)` on the user level; for a continuous metric, estimate from the baseline window. For unit-of-analysis = user with a per-session metric, the variance is higher than the per-session variance and must be estimated at the user level (see §5).
- **MDE** — the smallest effect the team *cares to detect*. Not the smallest effect statistically detectable. The MDE is a product-and-risk judgement: if primary moves by Δ, is that worth the launch? Does Δ exceed the measurement noise of the metric over the chosen window? Does Δ exceed the cost of the intervention?
- **Significance level α.** Two-sided 0.05 is the default; argue the deviation if you change it. Larger α raises false-positive risk and should be paired with stricter guardrail evidence.
- **Power 1-β.** 0.80 is the common default; launch-gate experiments where a missed real effect is costly often target 0.90.
- **Assignment ratio.** 50/50 minimizes sample-size cost for a two-arm test; unequal ratios are chosen for safety (e.g., 90/10 during initial ramp) at a sample-size penalty.

**Formulae.**

For a difference in proportions (e.g., CTR, conversion), with equal-sized arms:

```
n_per_arm ≈ ((z_{α/2} + z_β)² · (p1(1 - p1) + p2(1 - p2))) / (p2 - p1)²
```

For a difference in means with equal-sized arms and variance σ²:

```
n_per_arm ≈ 2 · ((z_{α/2} + z_β)² · σ²) / Δ²
```

Where `z_{α/2}` is the normal quantile for two-sided significance (≈1.96 at α = 0.05) and `z_β` is the quantile for the chosen power (≈0.84 at 0.80, ≈1.28 at 0.90). The plan must record which formula was used and the exact z-values; a reader must be able to reproduce the number.

The MDE and the sample size are reciprocal views of the same design. Fix any two of (MDE, n, power, α) and the third follows. The plan writes down all four — not just the number that made the design fit.

**Runtime.**

Runtime = sample-size requirement divided by expected qualified traffic per day. If runtime exceeds a ceiling (two weeks is a common upper bound, longer invites weekday / seasonal confounding), the design loop runs again: widen the MDE, accept less power, enlarge the eligible population, or pick a less noisy metric (see exercise-03 on variance reduction). A design that requires a six-week run to detect an effect the team would ship on week two is not a design; it is a shelf.

### 2.4 Ramp plan

A ramp is the time-ordered sequence of exposure percentages between "experiment starts" and "decision is made." Its job is to make a failure cheap: catch SRM, infrastructure skew, and guardrail regressions at 1% rather than 50%. The ramp is pre-registered; moving faster than the plan requires a logged override.

**Default ramp shape.**

```
R1  Shadow (if feasible):         0% exposure; control sees production, treatment runs
                                  in parallel and its predictions / effects are discarded.
                                  Dwell: 1–3 days.
                                  Exit gate: zero crash rate on treatment path, feature
                                  parity, no latency pathology at full traffic volume.

R2  1% ramp:                      1% exposure to treatment, 99% to control.
                                  Dwell: 1–3 days, enough to observe steady state.
                                  Exit gate: no SRM at ≥ 0.01 significance on the chi-square
                                  test (see exercise-02); no guardrail trip; no error-rate
                                  anomaly.

R3  5% ramp:                      5% / 95%.
                                  Dwell: ≥ 3 days, spanning at least one full weekly cycle
                                  if the metric has day-of-week structure.
                                  Exit gate: SRM clean; guardrails within tolerance;
                                  primary directional sign matches hypothesis or is flat.

R4  Full-power split (e.g., 50/50): expose at the ratio the sizing was computed for.
                                  Dwell: until the sample size from §2.3 is reached, OR
                                  the pre-registered early-stopping criterion fires.
                                  Exit gate: pre-registered decision rule (§2.5).

R5  Rollout:                      If win, ramp to 100% in steps (e.g., 50% → 75% → 100%);
                                  if loss or inconclusive, revert and record.
```

**Dwell time.** Each step holds long enough to observe the metric at steady state — novelty effects on an engagement metric often decay over several days, and weekly-seasonal metrics need a full week per step to avoid mistaking Thursday for a win. The plan records the minimum dwell per step and names the rationale ("one weekly seasonal cycle," "two days of stable guardrails," not "overnight").

**Advance gates.** Each step has a pre-registered gate that must hold to advance: SRM clean, guardrails within tolerance, no error-rate anomaly. Gates are evaluated at the end of the dwell window, not peeked at hourly.

**Rollback triggers.** Each step has pre-registered rollback triggers that fire *without* a readout: a hard SRM at α = 0.001, error-rate spike beyond tolerance, safety-guardrail breach, p95 latency beyond the budget. Rollback is automatic within an SLO — `t_rollback` is named (e.g., "within 5 minutes of alert fire") and is a function of the on-call rotation, not a human in a meeting.

**Manual override.** A human can always slow the ramp. A human should rarely speed it up; if they do, the decision is logged with the role, the date, the reason, and the counterfactual (what the plan said should happen).

### 2.5 Pre-registered decision rule

Pre-registration is the discipline that separates an experiment from a story. Before ramp begins, the plan records:

- **Win condition.** Primary moves by at least the MDE in the intended direction with p < α AND all guardrails within tolerance.
- **Loss condition.** Primary moves against the hypothesis beyond the MDE with p < α OR a guardrail trips beyond tolerance.
- **Inconclusive.** Sample size reached, neither win nor loss conditions met. Default action: do not ship; file for a follow-up with improved power (exercise-03) or a redesigned intervention.
- **Early-stopping policy.** Either no peeking (the honest default; look only at the sample-size end) or a declared sequential procedure (e.g., group-sequential bounds, always-valid confidence intervals). Peeking without a declared correction inflates false-positive rate; call it out or do not do it.
- **Multiple-comparisons policy.** If the plan tracks more than one primary (avoid this; secondary metrics are not primaries), the family-wise error correction is named up front (Bonferroni, Benjamini–Hochberg). If any subgroup result will be reported as a conclusion, the correction extends to subgroups.
- **Sign-off.** The named roles that sign at each gate: SRE / on-call for R1–R2 advance, ML / experimentation owner for R3 advance, product / business owner for R4 decision. Not people — roles — so the gate survives rotation.

### 2.6 Illustrative worked example (not a real system)

> **Illustrative feature** — a checkout-recommendation ranker on a tabular retail product. Logged-in users on web, US locale. This example is a composite for calibration; do not treat it as a real system or incident.

Experiment frame:

- Intervention: `reco-v12` replacing `reco-v11` on the cart-upsell slot.
- Decision fed: promote to 100% default if win; otherwise revert.
- Population: logged-in US web users who reach the cart page.
- Unit of analysis: user.
- Assignment: hash of `user_id` with salt `reco-v12-exp`, 50/50 at R4.
- Hypothesis: `reco-v12` increases per-user attach rate (share of cart sessions that add an upsell item) by ≥ 1 pp absolute.
- Last reviewed: 2026-11-10.

Metric panel:

- Primary: per-user attach rate.
- Secondary: average order value, items per order, cart abandonment rate.
- Guardrails: p95 cart-page latency ≤ 350 ms; checkout error rate within ±0.1 pp of baseline; revenue per user flat within ±0.5%; worst-slice (new-user cohort, past 7 days) attach rate not regressing by more than 0.5 pp; policy-violation rate (recommended-item flag) ≤ baseline.

Power and sample size:

- Baseline attach rate: 18.5% (28-day trailing, per-user).
- MDE: 1 pp absolute (primary changes from 18.5% → 19.5% or better to count).
- α: 0.05 two-sided. Power: 0.80 (z_β ≈ 0.84, z_{α/2} ≈ 1.96).
- Proportion-difference formula with full-precision z-values (`z_{α/2} ≈ 1.95996`, `z_β ≈ 0.84162`): `n_per_arm ≈ (2.80159² · (0.150775 + 0.156975)) / 0.0001 ≈ 7.84888 · 0.30775 / 0.0001 ≈ 24,154`.
- Qualified traffic: ~40,000 eligible users/day.
- Runtime at 50/50: ≈ 24,154 × 2 / 40,000 ≈ 1.21 days. Add dwell time per ramp step to get total runtime.

Ramp plan:

- R1 shadow: 2 days. Exit gate: feature parity confirmed, no crash on treatment, p95 within budget at shadow load.
- R2 1%: 2 days. Exit gate: SRM p ≥ 0.01 on chi-square, no guardrail trip.
- R3 5%: 7 days (full weekly cycle). Exit gate: SRM p ≥ 0.01, guardrails within tolerance, primary sign ≥ 0 or within noise.
- R4 50/50: until n ≥ 24,131 per arm. Dwell ≥ 7 days to include a weekly cycle.
- R5: step to 100% in 25% increments on win; revert on loss or inconclusive.

Sign-off roles: SRE on-call (R1–R2), experimentation owner (R3), product owner (R4 decision).

### 2.7 Runnable sketch: `power.py`

```python
# power.py
"""Reference power / MDE / sample-size calculator.

Design goal: pin all four of {MDE, n, power, alpha} to numbers so a plan is
reproducible. A design where any of the four is "about" something is not a design.
"""

from dataclasses import dataclass
from math import ceil
from statistics import NormalDist


@dataclass(frozen=True)
class ProportionsDesign:
    baseline: float
    mde_abs: float
    alpha: float = 0.05
    power: float = 0.80
    two_sided: bool = True

    def required_n_per_arm(self) -> int:
        z_alpha = NormalDist().inv_cdf(1 - self.alpha / (2 if self.two_sided else 1))
        z_beta = NormalDist().inv_cdf(self.power)
        p1 = self.baseline
        p2 = self.baseline + self.mde_abs
        if not 0 < p1 < 1 or not 0 < p2 < 1:
            raise ValueError("baseline and baseline + mde must lie in (0, 1)")
        num = (z_alpha + z_beta) ** 2 * (p1 * (1 - p1) + p2 * (1 - p2))
        return ceil(num / (self.mde_abs ** 2))


@dataclass(frozen=True)
class MeansDesign:
    sigma: float
    mde_abs: float
    alpha: float = 0.05
    power: float = 0.80
    two_sided: bool = True

    def required_n_per_arm(self) -> int:
        z_alpha = NormalDist().inv_cdf(1 - self.alpha / (2 if self.two_sided else 1))
        z_beta = NormalDist().inv_cdf(self.power)
        num = 2 * (z_alpha + z_beta) ** 2 * (self.sigma ** 2)
        return ceil(num / (self.mde_abs ** 2))


def runtime_days(n_per_arm: int, qualified_users_per_day: int, ratio: tuple[int, int] = (1, 1)) -> float:
    r_t, r_c = ratio
    total_n = n_per_arm * (r_t + r_c) / max(r_t, r_c)
    return total_n / qualified_users_per_day


if __name__ == "__main__":
    design = ProportionsDesign(baseline=0.185, mde_abs=0.01, alpha=0.05, power=0.80)
    n = design.required_n_per_arm()
    print(f"n per arm: {n}")
    print(f"runtime at 40k/day, 50/50: {runtime_days(n, 40_000):.1f} days")
```

The script is deliberately small. The discipline the plan enforces is that every number in the design document can be re-derived from the inputs, not that the calculation is elaborate.

## 3. Validation steps

Walk the plan through this checklist:

1. **Hypothesis is one sentence** with a directional claim, a named primary metric, and a quantified effect.
2. **Primary metric is one metric.** Weighted composites are not primaries; if a composite is used, the weights are in the plan and were reviewed.
3. **Baseline, variance, MDE, α, power, and sample size are all numeric** and derivable from the inputs via the script.
4. **Unit of analysis matches randomization unit.** A per-session metric analyzed per user has its variance estimated at the user level, not the session level.
5. **Runtime is calculated** from sample size and qualified traffic. Runtime exceeds no declared ceiling without a documented re-plan.
6. **Guardrails have directions and numeric tolerances.** "Latency should be good" is not a guardrail.
7. **Ramp plan has at least four steps** (shadow → 1% → 5% → full) with named dwell times and gates.
8. **Dwell on at least one step spans a full weekly cycle** for metrics with weekly seasonality.
9. **Decision rule is pre-registered.** Win / loss / inconclusive and early-stopping policy are named before ramp starts.
10. **Sign-off roles are named per gate**, not people.
11. **Rollback triggers are pre-registered** and bind on-call, not a meeting.
12. **Plan references the SRM diagnostic** (exercise-02) as a gate between ramp steps.

Static checks to drop into CI on `power-and-ramp.md`:

```bash
grep -ciE "baseline|variance|mde|alpha|power|sample size" power-and-ramp.md   # sizing inputs named
grep -ciE "ramp|dwell|exit gate|rollback trigger"         power-and-ramp.md   # ramp present
grep -ciE "pre-registered|decision rule"                  power-and-ramp.md   # decision rule named
grep -ciE "guardrail"                                     power-and-ramp.md   # guardrails present
grep -ciE "nist ai rmf|owasp|mitre atlas"                 power-and-ramp.md   # standards cited
grep -ciE "last reviewed"                                 power-and-ramp.md   # freshness
```

For `power.py`, a smoke test runs the CLI example in §2.7 and compares the printed `n` to a stored reference.

## 4. Rubric / review checklist

- [ ] Experiment frame names intervention (pinned version), decision fed, population, unit of analysis, assignment mechanism, hypothesis, and last-reviewed date.
- [ ] Single primary metric declared; secondaries are explanatory, not gates.
- [ ] At least three guardrail metrics declared with directions and numeric tolerances, covering latency, error rate, business floor, safety, and equity as applicable.
- [ ] Baseline value and variance pulled from a named recent window at the correct unit of analysis.
- [ ] MDE is a product/risk judgement, not a statistically-convenient value, and the plan explains why the team cares about an effect of that size.
- [ ] α, power, two-sided default, and assignment ratio are explicit numbers.
- [ ] Sample size is computed via a named formula; the formula is reproduced in the plan.
- [ ] Runtime is derived from sample size and qualified traffic and compared against a declared ceiling.
- [ ] Ramp has at least four steps (shadow → 1% → 5% → full, then rollout), with dwell times and exit gates per step.
- [ ] At least one step dwells through a full weekly cycle (for metrics with day-of-week structure).
- [ ] Pre-registered decision rule: win, loss, inconclusive; early-stopping policy; multiple-comparisons policy.
- [ ] Sign-off roles named per gate (SRE / on-call, experimentation owner, product owner), not people.
- [ ] Pre-registered rollback triggers (hard SRM, error spike, guardrail breach) with time-bound automation.
- [ ] Re-planning path named for "effect smaller than MDE" and "runtime too long" cases, including variance reduction via CUPED (exercise-03).
- [ ] NIST AI RMF MEASURE function cited for measurement-validity grounding; MANAGE function cited for ramp-as-risk-control framing.
- [ ] For systems with a safety or policy surface, OWASP ML Top 10 and MITRE ATLAS tactics are reflected in the safety guardrail choice.
- [ ] Last-reviewed date within the plan's declared cadence.

Review-weighted points for grading:

- Experiment frame and hypothesis specificity: 10%
- Metric panel (primary / secondary / guardrails with directions and tolerances): 15%
- Power / MDE / sample-size calculation (named inputs, correct formula, reproducible number): 25%
- Ramp plan (steps, dwell, gates, rollback): 20%
- Pre-registered decision rule and sign-off roles: 15%
- Re-planning path and runnable artefact: 10%
- Standards alignment (NIST AI RMF MEASURE / MANAGE, OWASP / MITRE ATLAS where applicable) and hygiene: 5%

## 5. Common mistakes

- **MDE as the smallest statistically-detectable effect.** The MDE is "the smallest effect worth shipping on." A 0.1% MDE that requires six weeks of traffic is not a plan; it is a wait.
- **Unit-of-analysis / randomization-unit mismatch.** Randomizing on user and analyzing per session (or vice versa) misstates variance. The ratio of these two variances is not 1; the plan must name both.
- **Baseline variance estimated on treatment-contaminated data.** Variance for sizing is drawn from a pre-experiment window, not from a window that includes the treatment.
- **Peeking without a declared stopping rule.** Early stopping on an uncorrected p-value inflates the false-positive rate well beyond α. Either do not peek, or pre-register a sequential procedure.
- **Secondaries as primaries at readout time.** If a secondary metric is going to be reported as a win, it was a primary all along and should have been sized for.
- **Guardrails evaluated at the end.** A guardrail is a trigger, not a readout. A guardrail only checked after the decision has been drafted is rhetorical cover.
- **Novelty effects ignored.** A lift that appears in the first two days and disappears by day seven is not a lift. Dwell per step must cover at least one weekly cycle for engagement metrics.
- **Running the experiment on launch pressure.** Under time pressure, a ramp is often skipped and the experiment starts at 50/50. The ramp is the risk control; skipping it is not a shortcut, it is an incident waiting to happen (see `mod-307-ml-reliability-slos`).
- **SRM detected and ignored.** An SRM at 1% ramp invalidates the subsequent results on that split. The plan commits in advance to pausing on SRM, not to re-interpreting it.
- **Assignment unit colliding with a pre-existing experiment.** Two experiments on the same unit can confound each other. The plan names the mutual-exclusion list or the orthogonalization scheme used.
- **Plan written after start.** A "plan" dated after the ramp begins is not a pre-registration; it is a retrofit. The plan is dated before R1 ends.
- **No re-plan path.** A plan that does not say what happens when the MDE is not met reads as "we will decide later what counts as success." That is the shape of a p-hacked readout.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE function grounds the measurement-validity discipline that makes a pre-registered MDE, power, and decision rule a precondition to a trustworthy readout; MANAGE function grounds the ramp plan as a staged risk control.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source for the adversarial-input scenarios that inform safety guardrail choice for ML experiments where the treatment is a generative or classification-with-consequence system.
- MITRE ATLAS — <https://atlas.mitre.org/>. Source for named adversarial tactics (evasion, model extraction) relevant when a guardrail covers abuse-resistance of the treatment.

Local context (authoritative for the plan shape and the ramp ordering):

- Paired lesson: `lessons/mod-306-experimentation-at-scale/exercises/exercise-01-power-mde-and-ramp-plan.md`.
- Module README: `modules/mod-306-experimentation-at-scale/README.md`.
- Related exercise (SRM and trust diagnostics that gate each ramp step): `modules/mod-306-experimentation-at-scale/exercise-02-srm-and-trust-diagnostics/SOLUTION.md`.
- Related exercise (variance reduction when the plan's runtime is too long): `modules/mod-306-experimentation-at-scale/exercise-03-variance-reduction-with-cuped/SOLUTION.md`.
- Related exercise (causal design when a clean A/B cannot be run at all): `modules/mod-306-experimentation-at-scale/exercise-04-causal-inference-when-ab-is-impossible/SOLUTION.md`.
- Related exercise (offline harness that must pass before the online ramp begins): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
