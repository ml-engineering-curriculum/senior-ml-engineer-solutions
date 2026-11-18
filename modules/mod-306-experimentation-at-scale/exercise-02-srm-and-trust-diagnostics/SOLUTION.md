# exercise-02 — SRM And Trust Diagnostics (Reference Solution)

Paired exercise: `lessons/mod-306-experimentation-at-scale/exercises/exercise-02-srm-and-trust-diagnostics.md`.

## 1. Solution overview

This exercise produces `trust-diagnostics.md` plus a small runnable artefact (`srm.py`) — a written diagnostic protocol and the code that implements it, so that for any A/B test the team can answer, before anyone looks at the primary metric: *is this experiment trustworthy?* The artefact has six parts:

1. A **diagnostic frame** that pins down what an SRM (Sample Ratio Mismatch) is, what "trust" means in this protocol, and where in the ramp plan the diagnostics run.
2. A **suite of checks** — SRM at every ramp step, pre-period balance on key covariates, A/A sanity on a null slice, segment-SRM, exposure-triggered exposure consistency, and metric null-distribution checks.
3. A **decision policy** — the pre-registered rules that convert diagnostic results into pause / continue / invalidate decisions, with roles and expiries.
4. A **root-cause playbook** — the ranked list of common causes when a check fires (bucketing bug, filtering asymmetry, caching, bot traffic, triggered-analysis bias, redirect loss) and the diagnostic each cause produces.
5. A **reporting contract** — the structured output every experiment carries alongside its primary readout, so a reviewer can tell "trusted result" from "effect size without a trust claim."
6. A **re-run and recovery plan** — what happens after a diagnostic trip: what is salvageable, what is not, and how the team re-enters the ramp from exercise-01.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric.

The exercise exists because an experiment with an untrusted assignment or triggered-analysis bias is not a slower experiment; it is a wrong one. A 2% SRM at 50/50 is small-looking and statistically catastrophic — the asymmetry in which users got the treatment is almost always correlated with the metric the team is reading. NIST AI RMF MEASURE 2.x asks that measurements themselves be assessed for reliability; these diagnostics are that assessment for one online test, run before the test is allowed to speak.

## 2. Worked answer / implementation

### 2.1 Diagnostic frame

Six lines, in this order:

- **What a Sample Ratio Mismatch is.** The actual counts of randomised units per arm differ from the designed ratio by more than chance. At a designed 50/50 ratio with 100,000 users, "48,500 / 51,500" is a 3-point mismatch whose chi-square p-value is effectively zero; it is not noise, it is a bug.
- **What "trust" means in this protocol.** Trust is a binary attached to a readout. A trusted readout has: SRM clean at every ramp step, A/A clean on the pre-period, exposure consistency confirmed, and no post-exposure asymmetry. An untrusted readout is not a readout.
- **When diagnostics run.** At every ramp step before the exit gate (see `exercise-01`), at the final-decision readout, and on a cadence during long experiments to catch late-arriving drift.
- **Where diagnostics live.** In the experimentation platform, not in the ad-hoc notebook of whoever runs the readout. Diagnostics are platform artefacts; their absence is a platform gap, not a user error.
- **What a diagnostic trip does.** Pauses the ramp or the readout, routes the owner, and withholds the result from the next sign-off until root cause is named.
- **Last reviewed.** A date, within the protocol's review cadence.

### 2.2 The diagnostic suite

**D1. SRM chi-square check.**

For a designed assignment ratio `(r_1, r_2, ..., r_k)` and observed counts `(n_1, n_2, ..., n_k)`, compute the chi-square statistic against expected `E_i = N · r_i / Σ r_j` where `N = Σ n_i`:

```
χ² = Σ_i (n_i − E_i)² / E_i
p  = 1 − CDF(χ²; k − 1)
```

Pre-registered thresholds:

- `p < 0.001`: hard trip. Pause ramp. Do not report results. Root-cause mandated.
- `0.001 ≤ p < 0.01`: soft trip. Pause ramp. Rerun with the next hour of data; if the soft trip repeats, treat as hard.
- `p ≥ 0.01`: clean. Continue.

The threshold is tight because SRM is a bug signature, not a comparison; a p-value floating near 0.05 at scale means the asymmetry is not random. A one-sided threshold is not used here — asymmetry in either direction is a problem.

**D2. Pre-period balance.**

Before exposure begins, randomised units should look alike on key pre-period covariates: tenure, historical value of the primary metric, locale, platform, sign-up cohort. "Look alike" means the two-sample difference in means is within the sampling-noise band for each covariate.

A pre-period imbalance on a covariate that correlates with the primary is a leading indicator of an SRM that will show up later, and also feeds the CUPED variance-reduction choice (exercise-03). The protocol names the covariate list, the test used per covariate (t-test or chi-square depending on type), and the per-covariate tolerance.

**D3. A/A sanity.**

On a historical window with no treatment, randomly split users by the same hash and salt that the experiment will use, and compute the primary metric in each arm. The expected result: the difference is centred at zero and the empirical false-positive rate over many random splits is close to α. If A/A shows a persistent non-null effect, the issue is the metric, the pipeline, or the unit of analysis — not the treatment.

A/A sanity is run once per metric-and-salt combination, not once per experiment; the result is stored on the metric definition.

**D4. Segment-SRM.**

Segment-SRM is SRM within a slice. A design that is 50/50 overall but 60/40 in `locale=en` and 40/60 in `locale=non-en` has a segment-SRM even though the global ratio is clean. The protocol names the segment list (locale, platform, tenure bucket, protected-attribute proxies where available) and runs D1's chi-square per segment.

Segment-SRM is especially diagnostic of bucketing-collision bugs: a cohabiting experiment holding one segment back, a legacy A/B infrastructure routing on device class, a cache serving stale assignments to a specific locale.

**D5. Exposure consistency.**

For triggered experiments (users only enter the analysis once they hit a surface the experiment affects), check that trigger rates are equal across arms. If treatment users trigger more often than control users — because the surface itself is different — the analysis is already biased before the metric is computed. Expected trigger-rate equality is itself a chi-square check; the threshold matches D1.

Exposure consistency also covers: assignment-cache TTL skew, logged-in-vs-logged-out leakage, cross-device identity bleed, bot filter asymmetry.

**D6. Metric null-distribution check.**

Re-run the primary metric on randomly-generated A/A splits of the actual experiment's enrolled users (not a historical window). Over many such splits, the empirical distribution of the metric difference should be centred at zero and have the variance the sizing used. If the empirical variance is larger than the sizing variance, the experiment is under-powered relative to plan; if it is smaller, the sizing was conservative. In either case the plan's MDE changes, which changes the readout.

This check is run at the end-of-experiment readout, not per step.

### 2.3 Decision policy

Pre-registered, first-match-wins. Default is "do not report."

```
T1 (missing diagnostic):
  If any required diagnostic for this ramp step was not run or is
  missing output → BLOCK. The result is not reported. Root cause:
  the platform.

T2 (hard SRM, D1 or D4 or D5 with p < 0.001):
  → INVALIDATE this step. Pause the ramp. Root-cause before any
  re-enter. The result on this sample is not salvageable.

T3 (soft SRM):
  → PAUSE. Re-run D1 with the next data window. If the soft trip
  recurs, treat as T2.

T4 (pre-period imbalance beyond tolerance):
  → PAUSE. Decide, with the experimentation owner, whether CUPED
  adjustment (exercise-03) rescues the readout or whether re-
  randomisation is required.

T5 (A/A persistent non-null):
  → BLOCK the metric from being used in decisions. Open a metric-
  stability work item. The experiment can continue with a different
  primary; the broken metric does not become the decision.

T6 (D6 variance larger than plan):
  → RE-PLAN. The sizing from exercise-01 no longer achieves the
  declared power. Either extend runtime or widen the MDE.

T7 (all diagnostics clean):
  → RELEASE the readout for the next sign-off in the exercise-01
  pre-registered decision rule.

T8 (unknown):
  → BLOCK. New cases get rules, not improvised judgements.
```

Each diagnostic trip has a named role that owns the root-cause work: the experimentation platform owner for T1, T2, T3, T5; the metric owner for T5, T6; the experiment owner for T4, T6. Roles, not people.

Waivers are an escape hatch, not a policy. A waiver on a T2 trip would be an "we are reporting an untrusted readout" waiver — which is almost always wrong. If one is granted, it is dated, named-role, expiring, and loud in the readout.

### 2.4 Root-cause playbook

When a diagnostic fires, the playbook names the likely causes in rank order. The playbook is not a guess tree; each cause has an owned test that confirms or refutes it.

1. **Bucketing bug.** Hash collision across experiments, mutually-exclusive-set misconfiguration, salt reuse. Test: re-run assignment for a sampled user cohort through the bucketing service in isolation and compare to logged assignment.
2. **Filter asymmetry.** One arm includes users another arm excludes, because the filter uses post-exposure state. Test: for the enrolled population, verify the filter is applied identically and on pre-exposure state.
3. **Caching / stale assignment.** A CDN or client cache serves the pre-experiment version to a subset. Test: cache-key audit, correlate ratio drift with cache fleet rollouts.
4. **Redirect loss.** A redirect in the treatment path drops a fraction of users before analysis. Test: edge-log delta between arms.
5. **Bot or automation traffic.** Automated clients hashed into one arm more than the other because of header asymmetry. Test: filter by `user_agent` class and recompute D1.
6. **Triggered-analysis bias.** The surface that triggers inclusion is itself affected by treatment, so treatment users trigger at a different rate. Test: D5 at both the pre-trigger and post-trigger population; if the pre-trigger ratio is clean, this is the cause.
7. **Logged-in / logged-out leakage.** A session re-authenticates mid-experiment and is reassigned. Test: assignment-stability check per user over the experiment window.
8. **Clock skew in enrolment windows.** Treatment and control enrol on slightly different clocks, so the day-of-week composition differs. Test: enrolment timestamp histogram per arm.
9. **Simultaneous experiments.** A sibling experiment holding one segment back creates a segment-SRM in this one. Test: cross-reference active-experiment manifest.

The playbook is adapted per team; the discipline is that every entry has an *owned test*, not a vibe.

### 2.5 Reporting contract

Every readout carries a trust block. Minimum fields:

```yaml
trust:
  srm_d1: { p_value: 0.47, status: clean }
  srm_d4_segments:
    - { segment: locale, p_value: 0.31, status: clean }
    - { segment: platform, p_value: 0.08, status: clean }
  balance_d2:
    - { covariate: tenure_days, delta: 0.3, tolerance: 1.0, status: clean }
    - { covariate: baseline_primary_7d, delta: 0.002, tolerance: 0.005, status: clean }
  aa_d3: { run_id: aa-metric-attach-2026-10, status: clean }
  exposure_d5: { p_value: 0.22, status: clean }
  null_dist_d6: { empirical_var_ratio: 1.03, status: clean }
  overall: clean
  last_reviewed: 2026-11-10
  reviewed_by_role: experimentation_owner
```

An `overall: clean` is a precondition for the readout to be read. A readout without a trust block is not promoted past R1.

### 2.6 Illustrative worked example (not a real system)

> **Illustrative feature** — the `reco-v12` checkout recommendation experiment from `exercise-01`. This example is a composite for calibration; do not treat it as a real system or incident.

At the R2 (1%) exit:

- D1 SRM: expected 1,200 / 118,800, observed 1,185 / 118,815. `χ² ≈ 0.19`, `p ≈ 0.66`. Clean.
- D2 pre-period balance: `tenure_days` arm delta 0.4 days (tolerance 1.0), `baseline_attach_7d` delta 0.001 (tolerance 0.005). Clean.
- D3 A/A: stored result for `attach_rate` with salt `reco-v12-exp`, last re-run 2026-10-30, empirical FPR 0.049 against nominal 0.05. Clean.
- D4 segment-SRM: `locale`, `platform`, `tenure_bucket`. Smallest p-value 0.11. Clean.
- D5 exposure consistency: trigger rate 100% at R2 (experiment uses full-page assignment, not triggered); not applicable, documented.
- Overall: clean. Ramp exits R2.

At the R3 (5%) exit:

- D1 SRM: observed 5,912 / 94,088 vs. expected 5,000 / 95,000 at 5/95. `χ² ≈ 174`, `p ≈ 10⁻³⁹`. **Hard trip.**
- Playbook, in rank: bucketing bug first. Re-run assignment for 10k sampled users: logged assignments agree with service. Not the bucketing.
- Next: filter asymmetry. Audit the inclusion filter. Finding: a `country=US` filter was added in the last deploy and is applied *after* exposure for treatment and *before* exposure for control; treatment users outside the US are logged as enrolled while control users outside the US are excluded. SRM is the symptom.
- T2 fires: INVALIDATE this step. The R3 data is not salvageable. Deploy a fix that applies the filter identically; restart from R2.
- Readout at R3 is not written; the trust block is: `overall: invalidated`, owner `experimentation_owner`, cause `filter asymmetry post-exposure`.

### 2.7 Runnable sketch: `srm.py`

```python
# srm.py
"""SRM check with pre-registered thresholds.

The function is small on purpose. SRM is diagnostic, not interpretive; the plan
decides what to do on a trip, not this code.

Two-arm case uses the exact normal / chi-square (df=1) survival via erfc; the
k-arm case delegates to scipy.stats.chi2 when scipy is available, and otherwise
raises — because an in-repo multi-arm chi-square implementation is more
surface-area than this teaching artefact needs.
"""

from dataclasses import dataclass
from math import erfc, sqrt
from typing import Iterable, Literal

SrmStatus = Literal["clean", "soft_trip", "hard_trip"]


@dataclass(frozen=True)
class SrmResult:
    chi_square: float
    p_value: float
    status: SrmStatus


def _chi2_sf_df1(x: float) -> float:
    """P(X > x) for chi-square with 1 df. X = Z^2 where Z ~ N(0,1), so
    P(X > x) = 2 * P(Z > sqrt(x)) = erfc(sqrt(x / 2)).
    """
    return 1.0 if x <= 0 else erfc(sqrt(x / 2.0))


def _chi2_sf(x: float, df: int) -> float:
    if df == 1:
        return _chi2_sf_df1(x)
    try:
        from scipy.stats import chi2  # type: ignore
    except ImportError as exc:
        raise RuntimeError(
            "multi-arm SRM (df >= 2) requires scipy; install scipy or use a"
            " pairwise two-arm check"
        ) from exc
    return float(chi2.sf(x, df))


def srm_check(observed: Iterable[int], expected_ratios: Iterable[float],
              soft: float = 0.01, hard: float = 0.001) -> SrmResult:
    observed = list(observed)
    ratios = list(expected_ratios)
    if len(observed) != len(ratios) or len(observed) < 2:
        raise ValueError("observed and expected_ratios must align, length >= 2")
    total = sum(observed)
    denom = sum(ratios)
    expected = [total * r / denom for r in ratios]
    if any(e <= 0 for e in expected):
        raise ValueError("all expected counts must be positive")
    chi2 = sum((o - e) ** 2 / e for o, e in zip(observed, expected))
    p = _chi2_sf(chi2, len(observed) - 1)
    if p < hard:
        status: SrmStatus = "hard_trip"
    elif p < soft:
        status = "soft_trip"
    else:
        status = "clean"
    return SrmResult(chi_square=chi2, p_value=p, status=status)


if __name__ == "__main__":
    # Clean case
    print(srm_check([1185, 118815], [0.01, 0.99]))
    # Hard trip case
    print(srm_check([5912, 94088], [0.05, 0.95]))
```

The script is a single function. The protocol's complexity lives in the decision policy (§2.3) and the playbook (§2.4), not in the chi-square.

## 3. Validation steps

Walk the protocol through this checklist:

1. **D1 threshold is numeric and tight.** `p < 0.001` for hard trip. Not "very small."
2. **Every diagnostic has an owner role.** T1–T8 name the role that leads the root-cause work.
3. **Segment-SRM list is named.** Locale, platform, tenure, protected proxies where available. Not "important segments."
4. **A/A results are stored per metric-and-salt.** Not re-run per experiment.
5. **Exposure-consistency check is specified for triggered experiments** and documented as "not applicable" for full-page assignment.
6. **Null-distribution check at readout.** The end-of-experiment variance is checked against sizing variance (ties back to exercise-01 §2.3).
7. **Trust block schema is explicit.** Every readout carries structured trust fields.
8. **Decision policy is first-match.** Default: do not report.
9. **Playbook entries each own a confirmation test.** No vibe causes.
10. **Last-reviewed date within cadence.**

Static checks on `trust-diagnostics.md`:

```bash
grep -ciE "chi-square|chi²"                       trust-diagnostics.md   # SRM method named
grep -ciE "p *< *0\.001|hard trip"                trust-diagnostics.md   # hard threshold numeric
grep -ciE "segment[- ]srm"                        trust-diagnostics.md   # segment check present
grep -ciE "a/a|null distribution"                 trust-diagnostics.md   # sanity checks present
grep -ciE "trust block|overall: *(clean|invalid)" trust-diagnostics.md   # reporting contract
grep -ciE "nist ai rmf"                           trust-diagnostics.md   # standard cited
grep -ciE "last reviewed"                         trust-diagnostics.md   # freshness
```

For `srm.py`:

- Smoke test: assert `srm_check([5000, 5000], [1, 1]).status == "clean"`.
- Smoke test: assert `srm_check([5912, 94088], [0.05, 0.95]).status == "hard_trip"`.
- Property: p-value in [0, 1] for valid inputs; raises on length mismatch or zero expected.

## 4. Rubric / review checklist

- [ ] Diagnostic frame names what SRM is, when checks run, where they live, and what a trip does.
- [ ] Six diagnostics (D1 SRM, D2 pre-period balance, D3 A/A, D4 segment-SRM, D5 exposure consistency, D6 null-distribution) each with an explicit test and threshold.
- [ ] SRM thresholds are numeric (`p < 0.001` hard, `0.001 ≤ p < 0.01` soft, `p ≥ 0.01` clean).
- [ ] Decision policy is an ordered rule list (T1..T8), first-match-wins, default "do not report," each rule names an owner role.
- [ ] Root-cause playbook ranks at least six causes (bucketing, filter asymmetry, caching, redirect loss, bot, triggered-bias) and each entry owns a confirmation test.
- [ ] Reporting contract defines a trust block with structured fields and an `overall` status.
- [ ] Trust block is a precondition for any readout to advance past R1 in the exercise-01 ramp.
- [ ] A/A sanity is stored per metric-and-salt, not per experiment.
- [ ] Null-distribution check closes the loop with the sizing variance from exercise-01.
- [ ] NIST AI RMF MEASURE 2.x cited for measurement-reliability grounding; MANAGE cited for trip-as-risk-control framing.
- [ ] Runnable `srm.py` implements the chi-square check with the pre-registered thresholds and passes smoke tests.
- [ ] Last-reviewed date within the protocol's declared cadence.

Review-weighted points for grading:

- Diagnostic suite (D1–D6 completeness and numeric thresholds): 25%
- Decision policy (ordered, numeric, role-owned, first-match): 20%
- Root-cause playbook (owned tests per entry, covers bucketing / filter / cache / bot / triggered): 15%
- Reporting contract (structured trust block, enforced as precondition): 15%
- Integration with `exercise-01` ramp and sizing (null-distribution feedback, trust block at each gate): 10%
- Runnable artefact (`srm.py` with smoke tests): 10%
- Standards alignment (NIST AI RMF MEASURE / MANAGE) and hygiene: 5%

## 5. Common mistakes

- **SRM threshold at α = 0.05.** A test at scale will trip or fail to trip at 0.05 for reasons unrelated to actual asymmetry. Hard SRM threshold is `p < 0.001`; the policy survives multiple-step ramps.
- **One global SRM, no segment-SRM.** The global ratio can be clean while every meaningful segment is skewed. The 50/50 that is 60/40 inside the segment that drives the metric is a loss, not a win.
- **A/A re-run per experiment.** The A/A property belongs to the metric, not the test. A metric that fails A/A fails for every experiment it is a primary of; store the result on the metric definition.
- **Trust check after the readout is drafted.** A trust block built after the headline moved is cover, not a check. The trust block gates the readout, not the other way around.
- **Triggered-analysis bias confused with real lift.** If the treatment makes the trigger more likely, triggered-only analysis sees a lift that is not causal. Check pre-trigger exposure (D5) first.
- **Caching and redirect-loss causes ignored.** The modal SRM in web experiments is infrastructure, not statistics. The playbook must name these causes explicitly.
- **Waivers used to ship an untrusted readout.** A T2 waiver is almost always wrong. If granted, it is dated, named-role, expiring, and loud in the readout — and the next experiment on the same path starts behind on trust.
- **Null-distribution check skipped at readout.** A real experiment often has larger empirical variance than the sizing assumed (cross-arm cohabitation, heavier-tailed users). If the check is skipped, the declared power is a story, not a property.
- **Bot and automation traffic not segmented.** A bot filter asymmetry across arms is a classic cause of SRM; the playbook must treat the bot class as a segment.
- **Diagnostics in the readout owner's notebook.** Diagnostics owned by one engineer are diagnostics that will be skipped under launch pressure. The platform owns the diagnostics; the owner reads the trust block.
- **"Soft" trips allowed to pass without re-check.** A soft trip that is not re-run in the next window is a hard trip the policy let through. The policy must be explicit that soft trips retry and escalate.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE function underpins the measurement-reliability discipline that these diagnostics implement; MANAGE function grounds the pause-on-trip default as a risk control.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source for threat-model framing when bot or adversarial traffic creates asymmetry across arms (e.g., automated-input / abuse scenarios).
- MITRE ATLAS — <https://atlas.mitre.org/>. Source for named tactics (e.g., adversarial automation, identity spoofing) that can produce assignment asymmetry and must be covered in the playbook for systems with an abuse surface.

Local context (authoritative for the diagnostic suite and the decision policy):

- Paired lesson: `lessons/mod-306-experimentation-at-scale/exercises/exercise-02-srm-and-trust-diagnostics.md`.
- Module README: `modules/mod-306-experimentation-at-scale/README.md`.
- Related exercise (ramp whose gates require this trust block): `modules/mod-306-experimentation-at-scale/exercise-01-power-mde-and-ramp-plan/SOLUTION.md`.
- Related exercise (variance-reduction method that re-uses D2's balance covariates): `modules/mod-306-experimentation-at-scale/exercise-03-variance-reduction-with-cuped/SOLUTION.md`.
- Related exercise (causal methods used when the diagnostic verdict is "A/B cannot be trusted here"): `modules/mod-306-experimentation-at-scale/exercise-04-causal-inference-when-ab-is-impossible/SOLUTION.md`.
