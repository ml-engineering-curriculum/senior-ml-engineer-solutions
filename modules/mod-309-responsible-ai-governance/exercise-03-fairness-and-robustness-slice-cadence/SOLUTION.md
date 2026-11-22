# exercise-03 — Fairness, Robustness, And Safety Slice Cadence (Reference Solution)

Paired exercise: `lessons/mod-309-responsible-ai-governance/exercises/exercise-03-fairness-and-robustness-slice-cadence.md`.

## 1. Solution overview

This exercise produces `slice-cadence.md` — a written cadence for the fairness, robustness, and safety slices that one production ML surface is evaluated against on every release, plus the thresholds, review workflow, and escalation path when a slice regresses. The artefact has six parts:

1. A **slice inventory** — the list of fairness slices, robustness slices, and safety slices the surface is evaluated against, each with a defined population, size, labelling methodology, metric, and launch-gate threshold.
2. A **metric-choice rationale** — for each slice, the fairness / robustness / safety metric chosen, the harm model it is defending against, and the choice's trade-offs (why this metric, not another).
3. A **cadence** — when each slice is recomputed (release gate, nightly canary, weekly review, quarterly recalibration), who owns it, and how the cadence is tied to retraining and release.
4. A **gate / regression policy** — the threshold for each slice that gates release, the regression tolerance for already-shipping slices, and the error-budget bookkeeping that ties slice health to ship/freeze decisions.
5. An **escalation / waiver policy** — what happens when a slice fails: launch-block, time-boxed waiver with named approvers, dataset-expansion backlog, or safety-net mitigation. Waivers are dated, signed, and reviewed.
6. A **review cadence for the cadence itself** — when the slice inventory is re-visited (quarterly default), who signs off, and what inputs (incidents, new population, new regulation, new model class) force an out-of-cycle re-review.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric that calibrates every learner's cadence document to the same bar.

The exercise exists because a Model Card §4 metrics table (exercise-01) is a snapshot; a slice cadence is the schedule that keeps the snapshot honest. "We measured fairness once at launch" is not a cadence — it is a release artefact that decays within weeks. NIST AI RMF MEASURE function grounds the cadence as the continuous-measurement surface on trustworthiness characteristics; MANAGE function grounds the gate and waiver policy as the risk-control surface that connects slice measurements to launch / freeze decisions. NIST AI 600-1's "harmful bias and homogenization" and "information security" GAI risk categories ground the fairness and adversarial-robustness slice families. MITRE ATLAS tactics ground the adversarial-input slice family; OWASP ML Top 10 ML01 (Input Manipulation Attack) and ML09 (Output Integrity Attack) ground specific slice content.

## 2. Worked answer / implementation

### 2.1 Why three slice families, not one

**Fairness slices** measure whether the model's performance (or its errors) differs across populations the surface serves. They answer: "Is this model worse, in a way the user feels, for one group than another?"

**Robustness slices** measure whether the model's performance holds under distribution shift that a user could reasonably produce — noisier inputs, different session shapes, longer context, out-of-vocabulary terms, adjacent locales. They answer: "Does the model degrade gracefully, or does it cliff when the input is slightly different from training?"

**Safety slices** (sometimes called "adversarial" or "misuse" slices — this exercise uses "safety" as the umbrella) measure whether the model refuses or correctly handles inputs an adversarial user could construct — prompt injections, jailbreaks, out-of-policy requests, abuse categories. They answer: "When a user tries to use this model badly, does it fail safely?"

The three families are intentionally separate because they have different owners, different data, different metrics, and different gate thresholds. Rolling them together into "evaluation" produces a cadence that cannot be prioritized — a fairness regression on a small locale needs a different response from a safety regression on prompt-injection resistance. This exercise's cadence holds them separate.

### 2.2 Slice inventory

Each slice is one row in the inventory, with these columns (the `slice-cadence.md` document can render the inventory as one table per family or as a flat table with a `family` column — the content matters, not the layout):

| Column | Purpose |
|---|---|
| Slice id | Short, stable identifier (`fair.locale.pt-BR`, `robust.session.long`, `safety.injection.direct`). Used in CI logs and alerts. |
| Family | `fairness` / `robustness` / `safety`. |
| Population / construction | How the slice is sampled or constructed. For fairness: which users (locale, device, cohort). For robustness: which transformation (added noise, longer context, OOV substitution). For safety: which attack type (direct prompt injection, policy evasion, jailbreak). |
| Dataset ref | The evaluation dataset(s) the slice is drawn from, citing the Data Statement from exercise-01. For safety slices, the red-team dataset and the dated version. |
| N | Slice sample size. Must be above the pre-registered minimum (e.g., 500) for the metric to be reportable, otherwise the row is marked low-power. |
| Metric | The metric evaluated, defined as "good-events / valid-events" where possible (see exercise-01 §4). |
| Launch-gate threshold | The value the metric must exceed on the slice to ship a release. |
| Regression tolerance | The maximum regression from the previous release the slice is allowed before an alert fires. |
| Cadence | How often the slice is recomputed: `release` / `nightly` / `weekly` / `quarterly`. |
| Owner | The team (role) that owns the slice. Fairness slices typically owned by the surface team + governance; safety slices by surface team + security; robustness by surface team + eval platform. |

**Slice families in detail:**

**Fairness slices.**

- **Language / locale.** One slice per supported language. For surfaces that serve multiple locales, one slice per locale. The threshold is a relative-gap bound against the primary locale (e.g., "pt-BR acceptance rate must be within 20% relative of en-US acceptance rate"). The threshold is justified in §2.3.
- **Device / platform.** Mobile vs. web, iOS vs. Android where behaviour diverges. Slice when the UX differs enough that the model's output quality is consumed differently.
- **Session shape.** New users vs. returning users, short-session vs. long-session. A model whose quality cliffs on new users is a model with a cold-start problem.
- **Protected-class proxies** (where consent and policy allow). Age-range cohorts from self-reported profile, accessibility-flag indicators from UI state. Proxy slices are labelled as proxies, never as ground-truth demographics; the risk of confounding is noted inline.
- **Business-tier cohorts.** Enterprise vs. SMB vs. free-tier. A model whose quality cliffs on free-tier users produces a fairness concern even if the cohort has no protected-class correlate.

**Robustness slices.**

- **Input length.** Short, medium, long context. Each slice is a bucket of the input-length distribution; thresholds are often tighter at the extremes where training data is sparser.
- **Distribution shift.** Weekly rolling slice against the training-cutoff window (does the model's performance on the current week's traffic hold, or is it drifting?).
- **Noise / typo robustness.** For text surfaces, a slice where inputs are corrupted with injected typos or synonym substitutions at a declared rate; the metric is the relative-gap against the clean slice.
- **OOV / long-tail.** A slice drawn from the tail of the input distribution — rare terms, rare queue types, rare feature combinations. Models tend to cliff here; the cadence makes the cliff visible.
- **Latency robustness.** Not just mean latency — p95 under slice-specific load (long inputs, cold-cache). The slice ties back to the SLO (mod-307/exercise-01).

**Safety slices.**

- **Direct prompt injection.** MITRE ATLAS AML.T0051. Inputs constructed to override the system prompt. The slice is built from a red-team dataset of injection patterns, versioned and updated quarterly.
- **Indirect prompt injection.** AML.T0051 extension — content in retrieved documents or tool outputs attempting to influence model behaviour. For RAG surfaces, this slice is load-bearing.
- **Jailbreak / policy evasion.** Inputs attempting to bypass safety guidance (role-play, prefix attacks, hypothetical framings). Dataset is maintained by a security review team; thresholds are refusal-rate-based.
- **Harmful content elicitation.** NIST AI 600-1 categories: CBRN, dangerous content, obscene content. Slice is built from a small, access-controlled probe set; metric is refusal rate.
- **Membership inference / training-data extraction.** OWASP ML Top 10 ML03 / ML04. Slice evaluates whether an attacker can extract training examples from the deployed model; metric is attack success rate against a random-baseline.
- **Abuse-case slices.** Task-specific: for a support-assistant, "customer-impersonation prompts"; for a code-assistant, "exfiltration-prompt" patterns. Each slice is grounded in a documented misuse case.

A slice inventory for one surface is typically 10–25 slices total across the three families. Fewer than 10 is probably under-covered; more than 25 is probably diluting on-call and release-review attention.

### 2.3 Metric-choice rationale

Every slice carries a one- to three-sentence rationale for the chosen metric. The rationale answers three questions:

- **What is the harm model?** What does a user feel when this slice degrades? "Agent spends more edit time when pt-BR drafts are worse than en-US drafts; customer waits longer for a reply." The harm is operational, not abstract.
- **Why this metric?** Why "acceptance rate parity" and not "equal opportunity" or "demographic parity"? For fairness, the choice is driven by the harm model: if false negatives on a safety slice are what cause harm, the metric is equal-opportunity on the positive class, not overall accuracy parity.
- **What are the trade-offs?** What does this metric NOT measure? "Acceptance-rate parity does not catch calibration gaps; a quarterly recalibration check is added to cover that gap."

Fairness metrics carry known trade-offs (demographic parity vs. equal-opportunity vs. calibration are generally not simultaneously satisfiable; this is well-established in the fairness literature). The exercise does not require the learner to re-derive the result but does require them to name which trade-off they are making and why it is acceptable for the surface.

### 2.4 Cadence

Cadence has four buckets, matched to how fast the measurement decays:

- **Release gate (every release).** Fairness slices gate release. Safety slices gate release for any user-facing surface. Robustness-cliff slices (OOV, long context) gate for generative surfaces and classifiers whose inputs vary materially.
- **Nightly canary.** A fast subset of slices runs against the previous day's production traffic or a shadow replay. Catches a fairness regression that only shows up on live traffic. Pages on the regression-tolerance miss, tickets on the launch-gate miss on canary.
- **Weekly review.** The surface team reviews slice health in the weekly meeting. Trends over the past 4 weeks, open regressions, waiver status. The weekly review is the forcing function for "no slice is forgotten."
- **Quarterly recalibration.** The slice inventory itself is reviewed: are these the right slices? Are the thresholds still appropriate? Are there slices to add, retire, or widen? New populations onboarded in the quarter must appear in the next quarter's inventory.

Cadence mapping per slice family:

- Fairness: release gate + weekly review + quarterly recalibration.
- Robustness: release gate (cliff slices only) + nightly canary (distribution-shift slice) + weekly review + quarterly recalibration.
- Safety: release gate + nightly canary (safety probes) + weekly review + quarterly red-team refresh + ad-hoc on vulnerability disclosure.

Each cadence tier names: who runs it, where the output lands (dashboard / report / ticket), who the alert goes to on failure, and the SLA for triage.

### 2.5 Gate / regression policy

**Launch gate.** A slice below its launch-gate threshold blocks release. "Below" is strict — the CI check is a boolean gate, not a reviewer's judgement. If the launch would ship a slice below gate, the launch is blocked and requires a waiver.

**Regression tolerance.** For slices already above the launch gate, a release is allowed to degrade the slice up to a declared tolerance (e.g., "no more than 2% absolute regression on the acceptance-rate SLI"). A regression beyond tolerance also blocks release.

**Error-budget bookkeeping.** Each slice has an error budget per quarter (`1 - threshold` for availability-shaped slices; a similar notion for quality-shaped slices). Slice budget burn is tracked alongside the SLO error budget from mod-307/exercise-01; a slice whose budget is exhausted is treated as a release blocker until the budget resets or a waiver is granted.

**Low-power rows.** A slice whose N is below the pre-registered minimum is not suppressed; it is marked `low-power`, reported with the CI, and the action is to backlog an N-expansion (dataset-expansion backlog). A low-power slice may still gate release if its lower-CI bound is below the threshold — the test is on the CI, not the point estimate.

**Composite gates.** For multi-slice regressions (e.g., 3 fairness slices regress 1% each), the policy declares whether the composite blocks release. The default is: a single-slice regression within tolerance passes; a composite regression that materially moves the aggregate blocks.

### 2.6 Escalation / waiver policy

When a slice fails:

- **Launch-block (default).** Release is blocked; the surface team triages, reproduces, fixes, re-runs the slice, and ships.
- **Time-boxed waiver (exception).** The surface team may request a waiver when the slice miss is understood and the release is time-sensitive. A waiver has:
  - A **named approver**: `team-agent-assist` tech-lead *and* a governance-reviewer signature. Never a single individual, never undated.
  - An **expiry date**: 90 days maximum. Longer waivers require a VP-level approver and a quarterly review.
  - A **mitigation plan**: the specific actions (dataset expansion, model retrain, feature-engineering fix, safety-net rule) that resolve the slice gap.
  - A **monitoring requirement**: additional telemetry or probe added while the waiver is active, so the slice miss does not get worse unnoticed.
  - A **public entry** in the surface's governance registry.
- **Dataset-expansion backlog.** For slices that fail because of low power or low coverage (rather than true regressions), the response is to expand the slice's dataset. The backlog is tracked, dated, and prioritized; a slice that has sat in the backlog for more than one quarter is escalated to VP-level.
- **Safety-net mitigation.** For safety slices that fail, a model-side fix may not be available before release. Safety-net mitigations are input filters, policy classifiers, or guardrails deployed around the model. Safety-net mitigations have their own slice evaluations and gates; a safety-net filter with no evaluation is itself a liability.

A waiver is not a skip — the waiver *is* the governance trail. Learners who write "we'll ship with a waiver" without the five waiver elements have not written a waiver; they have written a skip.

### 2.7 Review cadence for the cadence

The slice inventory itself is reviewed quarterly:

- **Who signs.** Surface team L30 owner, governance reviewer, security reviewer (for safety slices), product owner (for fairness slices that affect user-visible surfaces).
- **Inputs.** Prior quarter's slice-failure report, incidents attributed to a slice gap, new user populations onboarded, new regulations in scope, new attack patterns observed (feeds exercise-04), new paved-road probe sets (feeds exercise-02).
- **Outputs.** The next quarter's slice inventory, with slices added (new populations, new attack families), retired (populations removed, attack families consolidated), re-thresholded (thresholds tightened or loosened, with a reason), and re-sized (datasets expanded, low-power rows resolved).
- **Trigger-based.** Any of: an incident with a slice attribution, a new population onboarded mid-quarter, a new regulatory guidance (e.g., a new EU AI Act conformity requirement), a model class change (e.g., swapping the base model), or a new attack disclosure (e.g., a new MITRE ATLAS technique added).

### 2.8 Illustrative worked example (not a real system)

> **Illustrative feature** — `agent-assist/draft`, the internal LLM-backed support-agent assistant from exercise-01 and exercise-02. The slice-cadence excerpt below uses the same composite surface; do not treat it as a real system or incident.

**Slice inventory excerpt (fairness):**

| Slice id | Population | Dataset | N | Metric | Launch gate | Regression tol. | Cadence | Owner |
|---|---|---|---|---|---|---|---|---|
| `fair.locale.en-US` | en-US tickets, billing/shipping/account-mgmt queues | `eval-ds-v4` | 4,812 | Agent-acceptance rate | ≥ 0.65 | 2% abs | release + weekly + nightly-canary | `team-agent-assist` |
| `fair.locale.pt-BR` | pt-BR tickets | `eval-ds-v4` | 711 | Agent-acceptance rate, relative-gap vs. en-US | relative gap ≤ 25% | 2% abs | release + weekly | `team-agent-assist` + `team-governance` |
| `fair.locale.es-MX` | es-MX tickets | `eval-ds-v4` | 420 | Agent-acceptance rate, relative-gap vs. en-US | relative gap ≤ 25% | — (low-power; CI check only) | release + weekly | `team-agent-assist` + `team-governance` |
| `fair.queue.billing` | billing queue | `eval-ds-v4` | 5,100 | Agent-acceptance rate | ≥ 0.65 | 2% abs | release + weekly | `team-agent-assist` |
| `fair.queue.shipping` | shipping queue | `eval-ds-v4` | 4,230 | Agent-acceptance rate | ≥ 0.65 | 2% abs | release + weekly | `team-agent-assist` |
| `fair.queue.account-mgmt` | account-management queue | `eval-ds-v4` | 2,990 | Agent-acceptance rate | ≥ 0.60 | 2% abs | release + weekly | `team-agent-assist` |
| `fair.business-tier.enterprise` | enterprise-tier tickets | `eval-ds-v4` | 1,840 | Agent-acceptance rate, relative-gap vs. overall | relative gap ≤ 15% | 2% abs | weekly + quarterly | `team-agent-assist` + `team-support-ops` |
| `fair.accessibility-flag` | tickets with accessibility-flag indicators | `eval-ds-v4` | 220 | Agent-acceptance rate | low-power; CI lower-bound not materially below overall | — | release (low-power row) | `team-agent-assist` + `team-governance` |

**Slice inventory excerpt (robustness):**

| Slice id | Transformation | Dataset | N | Metric | Launch gate | Regression tol. | Cadence | Owner |
|---|---|---|---|---|---|---|---|---|
| `robust.length.long` | input length > 20 turns | `eval-ds-v4` | 980 | Agent-acceptance rate | ≥ 0.55 | 3% abs | release + weekly | `team-agent-assist` |
| `robust.length.short` | input length < 5 turns | `eval-ds-v4` | 3,110 | Agent-acceptance rate | ≥ 0.65 | 2% abs | release + weekly | `team-agent-assist` |
| `robust.oov.long-tail` | tickets with OOV terms / rare templates | `eval-ds-v4` | 430 | Agent-acceptance rate | relative gap ≤ 30% vs. overall | 3% abs | weekly | `team-agent-assist` |
| `robust.drift.weekly` | last-7-day production replay | shadow replay | ~18k/day | Agent-acceptance rate | ≥ 0.60 | 2% abs | nightly canary | `team-agent-assist` |
| `robust.typo.injected` | 10% token substitution rate | `eval-ds-v4` synthetic | 2,000 | Relative gap vs. clean | ≤ 10% | 3% abs | release | `team-agent-assist` + `team-model-eval` |

**Slice inventory excerpt (safety):**

| Slice id | Attack type | Dataset | N | Metric | Launch gate | Regression tol. | Cadence | Owner |
|---|---|---|---|---|---|---|---|---|
| `safety.injection.direct` | Direct prompt injection (ATLAS AML.T0051) | `red-team-ds-v4` | 1,200 | Injection-success rate | ≤ 0.05 | 1% abs | release + nightly | `team-agent-assist` + `team-security` |
| `safety.injection.indirect` | Indirect injection via ticket content | `red-team-ds-v4` | 800 | Injection-success rate | ≤ 0.05 | 1% abs | release + nightly | `team-agent-assist` + `team-security` |
| `safety.jailbreak.policy-evasion` | Policy-evasion attempts (role-play, hypothetical) | `red-team-ds-v4` | 600 | Refusal rate | ≥ 0.97 | 1% abs | release + nightly | `team-agent-assist` + `team-security` |
| `safety.harm.CBRN` | CBRN-adjacent requests (NIST AI 600-1) | `red-team-ds-v4` | 300 | Refusal rate | ≥ 0.98 | — (no tolerance; any regression blocks) | release + nightly + quarterly refresh | `team-security` |
| `safety.harm.dangerous` | Violent / self-harm content | `red-team-ds-v4` | 400 | Refusal rate | ≥ 0.98 | — | release + nightly + quarterly refresh | `team-security` |
| `safety.membership-inference` | Membership-inference probe set (OWASP ML03) | `red-team-ds-v4` | 500 | Attack success vs. baseline | success ≤ baseline + 1% | — | quarterly | `team-security` + `team-model-eval` |
| `safety.abuse.impersonation` | Customer-impersonation prompts | `red-team-ds-v4` | 300 | Refusal rate | ≥ 0.95 | 1% abs | release + nightly | `team-agent-assist` |

**Metric-choice rationale excerpts:**

- `fair.locale.pt-BR` uses relative-gap-vs-en-US because the harm model is "agents spend more edit time on worse drafts and customers wait longer"; the metric is a user-feelable gap in operational quality. Equal-opportunity on a positive class is not applied here because the training label is implicit (agent sent the draft) and there is no clean false-negative signal. The 25% relative-gap threshold is calibrated against a 2026-Q1 study that measured a 30% gap producing a measurable customer-wait increase.
- `safety.injection.direct` uses injection-success rate (lower is better) rather than refusal rate because the attack succeeds silently — the model does not refuse; it does the wrong thing. The metric measures whether the attacker's injected instruction was followed in the output, judged by an LLM-as-judge (see mod-305/exercise-04). The ≤ 5% threshold is calibrated against the paved-road safety-review baseline and MITRE ATLAS threat-severity guidance.
- `fair.accessibility-flag` is a proxy slice (accessibility UI indicators infer accessibility needs but do not confirm them). The row is labelled as a proxy, the threshold is a CI-lower-bound-not-below-overall rather than a strict relative-gap, and the quarterly recalibration reviews whether the proxy is stable enough to keep.

**Gate / regression policy excerpt:**

- Launch gate is boolean. A release with any `fair.locale.*` or `safety.*` slice below gate is blocked. A release with `fair.business-tier.*` or `robust.*` slice miss is reviewed by the surface L30 and may ship with a waiver.
- Regression tolerance is per-slice; a composite regression (≥ 3 slices regressing simultaneously, each within single-slice tolerance) blocks release and triggers an investigation.
- Error budget is reported per quarter. In 2026-Q3, `fair.locale.pt-BR` burned 60% of its quarterly budget on three consecutive weekly regressions; the quarterly recalibration retightened the threshold and scheduled a dataset expansion for 2026-Q4.

**Waiver example (illustrative):**

> **Waiver W-2026-10-14-pt-br-acceptance.** `fair.locale.pt-BR` is at 0.58 acceptance rate vs. 0.72 en-US (19% relative gap vs. launch gate of 25%); within threshold. Release is approved. A regression to 20% relative gap in the next cycle would re-review this waiver. Approvers: `team-agent-assist` tech-lead + `team-governance` reviewer, dated 2026-10-14. Mitigation plan: pt-BR dataset expansion by 2027-Q1 (owner: `team-data-platform` + `team-agent-assist`). Monitoring: weekly nightly canary on pt-BR acceptance rate, pager if the rolling 7-day acceptance rate drops below 0.55. Expiry: 2027-01-14. Registry entry: `governance-registry/waivers/W-2026-10-14-pt-br-acceptance`.

### 2.9 Optional runnable sketch: `slice_gate.py`

The cadence is enforced by a small gate script that reads a slice-result file (produced by the offline evaluation harness from mod-305/exercise-01) and the cadence's threshold file, and returns an exit code CI reads.

```python
# slice_gate.py
"""Gate slice results against the cadence thresholds.

Inputs:
  - slice_results.json   : output of the offline evaluation harness
  - cadence.yaml         : the slice-cadence thresholds

Exit:
  - 0 on all gates passed
  - 1 on any launch-gate miss
  - 2 on any regression-tolerance miss (allow a reviewer to force with --allow-regression-waiver=W-...)
"""

from __future__ import annotations
import json
import sys
import yaml
from dataclasses import dataclass


@dataclass
class SliceResult:
    slice_id: str
    n: int
    estimate: float
    ci_low: float
    ci_high: float
    prior_release_estimate: float | None


def load_results(path: str) -> list[SliceResult]:
    with open(path) as fh:
        data = json.load(fh)
    return [SliceResult(**row) for row in data]


def load_thresholds(path: str) -> dict:
    with open(path) as fh:
        return yaml.safe_load(fh)


def evaluate(results: list[SliceResult], cadence: dict) -> tuple[int, list[str]]:
    msgs: list[str] = []
    gate_miss = False
    regression_miss = False

    thresholds = {s["slice_id"]: s for s in cadence["slices"]}
    for r in results:
        t = thresholds.get(r.slice_id)
        if t is None:
            msgs.append(f"WARN unknown slice: {r.slice_id}")
            continue

        min_n = t.get("min_n", 500)
        low_power = r.n < min_n

        direction = t.get("direction", "higher-is-better")
        gate = t.get("launch_gate")
        if gate is not None:
            # Test on CI bound, not point estimate, so low-power rows can still gate.
            if direction == "higher-is-better":
                bound = r.ci_low
                passes = bound >= gate
            else:
                bound = r.ci_high
                passes = bound <= gate
            if not passes:
                gate_miss = True
                msgs.append(
                    f"FAIL launch-gate {r.slice_id}: {bound:.3f} vs gate {gate}"
                    f" (N={r.n}{'; low-power' if low_power else ''})"
                )
            elif low_power:
                msgs.append(
                    f"WARN low-power {r.slice_id}: N={r.n} < min_n={min_n}; passed on CI bound"
                )

        tol = t.get("regression_tolerance_abs")
        if tol is not None and r.prior_release_estimate is not None:
            if direction == "higher-is-better":
                regression = r.prior_release_estimate - r.estimate
            else:
                regression = r.estimate - r.prior_release_estimate
            if regression > tol:
                regression_miss = True
                msgs.append(
                    f"FAIL regression-tolerance {r.slice_id}: regressed {regression:.3f} > {tol}"
                )

    code = 1 if gate_miss else (2 if regression_miss else 0)
    return code, msgs


if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("usage: slice_gate.py <slice_results.json> <cadence.yaml>")
        sys.exit(3)
    results = load_results(sys.argv[1])
    cadence = load_thresholds(sys.argv[2])
    code, msgs = evaluate(results, cadence)
    for m in msgs:
        print(m)
    sys.exit(code)
```

`cadence.yaml` for the script mirrors the inventory table:

```yaml
slices:
  - slice_id: fair.locale.en-US
    direction: higher-is-better
    launch_gate: 0.65
    regression_tolerance_abs: 0.02
    min_n: 500
  - slice_id: fair.locale.pt-BR
    direction: higher-is-better
    launch_gate: null                 # relative-gap gate handled by a separate check
    regression_tolerance_abs: 0.02
    min_n: 500
  - slice_id: safety.injection.direct
    direction: lower-is-better
    launch_gate: 0.05
    regression_tolerance_abs: 0.01
    min_n: 500
  # ...
```

## 3. Validation steps

Walk the cadence document through this checklist before considering it done:

1. **Three slice families represented.** Fairness, robustness, safety — each with at least one slice.
2. **Slice inventory covers at least 10 slices** across the three families, with clear ids, defined populations, cited datasets, N, metric, launch gate, regression tolerance, cadence, owner.
3. **Dataset refs cite exercise-01 Data Statements and exercise-02 lineage.** A slice without a dataset provenance is unaudit-able.
4. **Every metric has a rationale** naming the harm model, the trade-off, and why this metric and not another.
5. **Launch-gate thresholds are justified** — not just a number, a number with a reason (calibrated against a user-study, a prior release, a standards guideline).
6. **Low-power rows are marked and routed** to a dataset-expansion backlog, not suppressed.
7. **Cadence tiers are named** (release gate / nightly canary / weekly review / quarterly recalibration), with owner, destination, pager rules, and SLAs.
8. **Gate policy is boolean at CI** and names the composite-regression rule.
9. **Waiver policy has five elements** (named approvers, expiry ≤ 90 d default, mitigation plan, monitoring requirement, governance-registry entry).
10. **Review cadence for the cadence is named** (quarterly default) with trigger conditions.
11. **Safety family cites MITRE ATLAS** techniques (AML.T0051, AML.T0043) on at least prompt-injection and adversarial-input slices; NIST AI 600-1 categories on harmful-content slices; OWASP ML Top 10 ML01/ML03/ML09 where relevant.
12. **Fairness family declares the fairness metric choice and its trade-off** against alternative metrics (demographic parity, equalized odds, calibration within groups).
13. **Robustness family includes a distribution-shift slice** (e.g., nightly rolling-week replay) tied to the SLO from mod-307.
14. **The cadence document references `slice_gate.py` or equivalent** as the enforcement mechanism in CI.

Static checks a CI step can enforce on `slice-cadence.md`:

```bash
grep -ciE "fairness|robustness|safety"                 slice-cadence.md
grep -ciE "launch gate|regression tolerance|threshold" slice-cadence.md
grep -ciE "release|nightly|weekly|quarterly"           slice-cadence.md
grep -ciE "waiver|approver|expiry|mitigation"          slice-cadence.md
grep -ciE "mitre atlas|aml\\.t|nist ai|owasp"          slice-cadence.md
grep -ciE "ci|95%|confidence|low.power"                slice-cadence.md
```

`slice_gate.py` from §2.9 enforces the numeric portion of the cadence.

## 4. Rubric / review checklist

- [ ] Three slice families represented (fairness, robustness, safety), with at least 10 total slices.
- [ ] Each slice has: id, family, population/construction, dataset ref (citing exercise-01 Data Statements), N, metric, launch gate, regression tolerance, cadence, owner.
- [ ] Each metric has a rationale naming the harm model and the trade-off against alternatives.
- [ ] Launch-gate thresholds are justified, not just asserted.
- [ ] Low-power rows are marked and routed to a dataset-expansion backlog; they are not suppressed.
- [ ] Cadence tiers (release gate / nightly canary / weekly review / quarterly recalibration) are each named with owner, destination, pager rules, and SLA.
- [ ] Gate policy is a boolean CI gate; composite-regression rule is declared.
- [ ] Waiver policy includes: named approvers (two-signature minimum), expiry (≤ 90 days default), mitigation plan, monitoring requirement, governance-registry entry.
- [ ] Review cadence for the cadence itself is named (quarterly default) with trigger conditions.
- [ ] Safety family cites MITRE ATLAS (AML.T0051 / AML.T0043) and NIST AI 600-1 categories.
- [ ] Fairness family declares metric choice with trade-off against demographic parity / equalized odds / calibration-within-groups.
- [ ] Robustness family includes a distribution-shift slice tied to the SLO (mod-307/exercise-01).
- [ ] `slice_gate.py` (or equivalent) referenced as the enforcement mechanism; cadence.yaml-shaped threshold file included.
- [ ] Cross-reference to exercise-01 (Model Card §4 metrics table and §7 fairness section populated from this cadence), exercise-02 (dataset provenance for every slice), exercise-04 (safety slices anchored in the threat-model's attack catalog).
- [ ] NIST AI RMF MEASURE / MANAGE cited; NIST AI 600-1 "harmful bias and homogenization" and "information security" GAI categories cited.

Review-weighted points for grading:

- Slice inventory completeness across three families: 20%
- Metric-choice rationale (harm model + trade-off): 15%
- Threshold justification and low-power discipline: 10%
- Cadence tiers and ownership clarity: 15%
- Gate / regression policy with composite rule: 10%
- Waiver policy with five required elements: 10%
- Standards alignment (NIST AI RMF, NIST AI 600-1, MITRE ATLAS, OWASP ML Top 10): 10%
- Cross-consistency with exercise-01 Model Card §4 / §7 and exercise-02 Data Statements: 10%

## 5. Common mistakes

- **One "fairness evaluation" at launch.** A single snapshot is not a cadence. The cadence is a *schedule*; without it, slices decay and incidents accrete silently.
- **Lumping families together.** Rolling fairness, robustness, and safety into "evaluation" produces a cadence that cannot be prioritized. A pt-BR acceptance regression needs a different response from a prompt-injection regression. Keep them separate.
- **Metric without rationale.** "We use demographic parity" is not a rationale; "demographic parity is chosen because the harm is operational quality parity, not calibration; equalized odds is not applied because the training label is implicit" is.
- **Threshold as an opinion.** A launch-gate threshold with no justification is an opinion; a reviewer cannot evaluate an opinion. Thresholds are calibrated against a user-study, a prior release, or a standards guideline, and the calibration is cited.
- **Suppressing low-power rows.** A slice with N=40 is not "too small to matter"; it is "low-power" and needs a dataset-expansion backlog. Suppressing it hides the fairness gap.
- **Waiver as a one-line note.** A waiver without named approvers, expiry, mitigation plan, monitoring requirement, and registry entry is a skip, not a waiver. The waiver *is* the governance trail; skipping the elements is skipping governance.
- **No regression tolerance.** A launch gate alone catches below-threshold slices; it does not catch a gradual degradation that stays above the threshold. Regression tolerance per slice closes the gap.
- **No composite rule.** Three slices each regressing 1% below tolerance individually can aggregate to a material user-feelable regression; the composite rule is the guard against death-by-a-thousand-cuts.
- **Safety slices without ATLAS / NIST anchors.** A safety slice without a named attack technique or risk category is un-reviewable; "we tested prompt injection" is not a slice, a specific AML.T0051 probe set with a dated version is.
- **Robustness without a drift slice.** A nightly rolling-week replay catches the distribution shift the release gates cannot see. Without it, the model may be shipping worse weekly and no one notices until a weekly review.
- **Pass/fail collapsing.** "We passed eval" is not a statement. The statement is "20 slices evaluated, 18 passed launch gate, 2 within regression tolerance; no composite regression; waiver W-... refreshed."
- **No review of the cadence itself.** A quarterly recalibration of the slice inventory is what keeps the cadence from decaying into history. Without it, the slice set calcifies around the launch populations and misses new ones.
- **Owner as a person.** "Jane will own the pt-BR slice" is not an owner. `team-agent-assist` + `team-governance` is.
- **No link to the Model Card.** The Model Card §4 metrics table (exercise-01) and the slice cadence are the same data at two different granularities. A cadence that does not populate the Model Card is a cadence running alone.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE function grounds the cadence as the continuous-measurement surface on trustworthiness characteristics (accuracy, reliability, safety). MANAGE function grounds the gate / waiver policy as the risk-control surface that connects slice measurements to launch / freeze decisions. GOVERN function grounds the review cadence for the cadence itself as the governance instrument that keeps the slice inventory legible to leadership.
- NIST AI 600-1 (Generative AI Profile) — <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>. The GAI risk categories "harmful bias and homogenization" ground the fairness family; "information security" and "obscene, degrading, or abusive content," "dangerous content," and "CBRN information" ground the safety family. Each is a required slice (or an explicit N/A with rationale) for a generative surface.
- MITRE ATLAS — <https://atlas.mitre.org/>. Cited on the safety family for adversarial-tactic taxonomy. Key techniques cited by default: AML.T0051 LLM Prompt Injection (direct and indirect), AML.T0043 Craft Adversarial Data, AML.T0024 Exfiltration via ML Inference API, AML.T0019 Publish Poisoned Datasets (ties to exercise-02), AML.T0010 ML Supply Chain Compromise. The slice-cadence's safety family inventory is derived from the subset of ATLAS techniques applicable to the surface.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. ML01 Input Manipulation Attack anchors the robustness.typo.injected and safety.injection.direct slices. ML03 Model Inversion Attack / ML04 Membership Inference Attack anchor the safety.membership-inference slice. ML09 Output Integrity Attack anchors any slice that evaluates output integrity under adversarial inputs.

Practitioner implementation examples:

- VeriSwarm Trust Center — <https://veriswarm.ai/trust>. Example of a practitioner publishing slice-level fairness and safety reports to external stakeholders; useful only as a shape-of-artefact reference, not as a standard.

Local context (authoritative for the cadence shape and cross-references):

- Paired lesson: `lessons/mod-309-responsible-ai-governance/exercises/exercise-03-fairness-and-robustness-slice-cadence.md`.
- Module README: `modules/mod-309-responsible-ai-governance/README.md`.
- Related exercise (Model Card §4 / §7 table that the cadence populates): `modules/mod-309-responsible-ai-governance/exercise-01-model-card-authoring/SOLUTION.md`.
- Related exercise (lineage + Data Statements that provide the dataset provenance for every slice): `modules/mod-309-responsible-ai-governance/exercise-02-data-lineage-and-pii-audit/SOLUTION.md`.
- Related exercise (threat model whose attack catalog anchors the safety slice family): `modules/mod-309-responsible-ai-governance/exercise-04-misuse-and-adversarial-threat-model/SOLUTION.md`.
- Related exercise (offline evaluation harness whose slice-dataset definitions this cadence consumes): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (shadow / canary design whose drift slice shares the cadence's nightly tier): `modules/mod-305-advanced-evaluation/exercise-02-shadow-and-canary-deploy-design/SOLUTION.md`.
- Related exercise (LLM-as-judge harness that implements the judge for refusal-rate and injection-success slices): `modules/mod-305-advanced-evaluation/exercise-04-llm-as-judge-when-and-how/SOLUTION.md`.
- Related exercise (SLO document whose quality SLIs the cadence's release-gate thresholds align with): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
