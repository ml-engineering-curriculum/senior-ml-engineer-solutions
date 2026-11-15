# exercise-02 — Shadow And Canary Deploy Design (Reference Solution)

Paired exercise: `lessons/mod-305-advanced-evaluation/exercises/exercise-02-shadow-and-canary-deploy-design.md`.

## 1. Solution overview

This exercise produces `online-rollout.md` — a written plan that says, for one ML model promotion, how the team will move from a passing offline harness result to a full production rollout *without* betting the service on a single step. The artefact has six parts:

1. A **promotion frame** that names the model under promotion, the baseline it replaces, the decision the rollout feeds (shadow → canary → ramp → full), and the user / business surface affected.
2. A **shadow stage design** — what traffic is mirrored, how predictions are captured and compared, which divergence metrics gate the exit from shadow, and what in shadow does *not* run (writes, side-effects, user-visible calls).
3. A **canary stage design** — the traffic split policy, the health metrics that gate each step up, the automatic-rollback triggers, and the manual override.
4. A **rollback and holdback plan** — what gets reverted on what signal in how many minutes, what holdback fraction remains after full rollout, and for how long.
5. A **monitoring plan** — which metrics are watched at which cadence by which owner, and how the plan integrates with the offline harness (exercise-01).
6. A **decision log** — the dated, named-role sign-offs for each stage, so the rollout is auditable after the fact.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric.

The exercise exists because ML rollouts fail in ways that unit tests and offline evals do not catch: traffic distribution drift between offline and online, infrastructure skew between training and serving, long-tail inputs the harness undersampled, and interaction effects with downstream systems. Shadow and canary stages are the controls the NIST AI RMF MEASURE and MANAGE functions expect between "the harness says pass" and "100% of users see the new model."

## 2. Worked answer / implementation

### 2.1 Promotion frame

Six lines:

- **Model under promotion.** Name, version, task. Pinned.
- **Baseline it replaces.** The currently-serving model, pinned to a version — not an alias.
- **Harness result it is promoted from.** A reference to the offline harness output: version, decision, date. If the offline harness returned `pass-with-note` or `pass-but-hold` (see exercise-01), the promotion plan explicitly addresses the note.
- **User / business surface.** Who is affected if the rollout goes wrong — and in what way. Latency, quality, safety, cost.
- **Decision this plan feeds.** "Promote through shadow → canary → full," with the gate at each step. Not "ship to prod."
- **Last reviewed.** A date, within the plan's review cadence.

### 2.2 Shadow stage design

Shadow is production traffic *mirrored* to the candidate: the candidate sees the same inputs the production model sees, produces predictions, and those predictions are *discarded* (nothing writes, nothing user-visible). Shadow catches: serving-infra skew (features that look fine offline and are missing / stale online), latency behaviour under real load, and divergence from the production model on real inputs.

**What shadow is for:**

- Validating that the candidate produces an output at all for real traffic.
- Measuring online latency distribution, not the offline-batch distribution.
- Measuring *divergence* between candidate and baseline on the real input stream — a divergence the offline eval cannot capture because offline inputs were sampled differently.
- Verifying feature freshness and completeness in production (no silently-missing feature that was present in the offline table).

**What shadow is not for:**

- Measuring quality with respect to ground truth, when ground truth is delayed (e.g., conversion, click, resolution outcomes). Shadow only sees divergence, not correctness. Ground-truth-based decisions belong in canary.
- Any downstream side-effect — no writes to databases, no outbound API calls, no notifications. The canonical mistake here is a "shadow" setup that still fires a downstream message because the candidate path did not fully branch.

**Shadow design checklist:**

- **Traffic selection.** All traffic, or a sample. A sample is fine if the sample is unbiased on the slices the offline harness cares about (see exercise-01).
- **Prediction capture.** Candidate predictions are logged with request id, baseline prediction, candidate prediction, latency, and feature snapshot for divergence analysis.
- **Divergence metrics.** For classifiers: label-agreement rate, class-distribution shift (PSI or KL), confidence histogram comparison. For regressors: delta distribution (p50, p95, max), correlation. For rankers: top-k overlap, Kendall tau. For generation: deterministic structural checks (schema conformance, length, refusal rate); quality comparison requires labels or a judge (exercise-04), which is a canary concern.
- **Serving health.** Candidate p50/p95/p99 latency, error rate, saturation. Candidate must meet the serving floor declared in exercise-01 §2.2 under *real* load, not under offline-batch load.
- **Gate to exit shadow.** A written condition. Example: "candidate served ≥ N hours, p95 latency within budget, error rate < X%, divergence metrics within tolerance on all hard-gated slices, no incident attributable to the candidate path." N is set so the shadow period covers at least one full business cycle (day-of-week / peak hour).
- **Side-effect isolation.** A single-sentence declaration: "the candidate path does not write, does not notify, does not call downstream services." If it does any of those, it is a pre-canary A/B, not shadow.

Senior-engineer notes:

- **Shadow is not optional for serving-infra changes.** Any time features, feature store, inference server, or hardware changes, shadow must run even if the model weights are unchanged.
- **Shadow catches outages, not quality.** A shadow that reports "divergence 3%" tells you the models disagree, not which one is right. Correctness belongs in canary.
- **Logging cost is real.** For high-QPS services, log a sampled divergence table at a rate that preserves slice coverage. The sample plan is in the design.

### 2.3 Canary stage design

Canary is real traffic *served* by the candidate, to a limited fraction of users or requests. Unlike shadow, canary users see the candidate's output. Canary measures quality against business metrics in production.

**Ramp policy.** A sequence of (traffic %, dwell time, exit gate) steps. First-match-wins gates, numeric thresholds, no vibes. A conventional ramp:

```
Step C1:   1% of traffic.    Dwell ≥ 24 h or 1 full business cycle (whichever longer).
Step C2:   5% of traffic.    Dwell ≥ 48 h.
Step C3:  25% of traffic.    Dwell ≥ 72 h.
Step C4:  50% of traffic.    Dwell ≥ 72 h.
Step C5: 100% of traffic.    Plan transitions to §2.4 holdback mode.
```

Dwell times accommodate: delayed labels, business-cycle variation, and the time needed for anomaly detectors to accumulate signal. They are a floor, not a target.

**Traffic assignment.** Deterministic (hash of user id or request id) so a user does not flip between models within a session. The hash function and salt are pinned; changing them mid-rollout invalidates comparability.

**Health metrics gating each step up.** Three tiers, same shape as the offline harness:

- **Guardrail health (hard gates).** p95 / p99 latency within SLO, error rate below budget, cost per request within budget, no safety / policy-violation rate regression. Any regression beyond tolerance → auto-rollback.
- **Quality health.** Business metric(s) that correspond to the offline headline — click-through rate, conversion, task-completion rate, agent-accept rate, whatever the team ships. Candidate must not regress the business metric beyond its noise band. Ties go to "do not promote further."
- **Slice health.** The hard-gated slices from the offline harness (exercise-01 §2.3) are tracked online. A slice regression beyond tolerance halts the ramp.

**Rollback triggers.** Explicit, numeric, first-match. Example:

```
R1 (SLO breach):    error rate > X% over 5 min OR p99 > Y ms over 5 min
                      → IMMEDIATE auto-rollback. Page on-call.
R2 (safety breach): policy-violation rate > baseline + Z over 15 min
                      → IMMEDIATE auto-rollback. Page on-call.
R3 (cost breach):   cost per request > budget * 1.5 over 15 min
                      → auto-reduce traffic to previous step; page on-call.
R4 (quality drop):  headline business metric below baseline - Δ for 2
                      consecutive cadence windows
                      → halt ramp; human review required to continue or rollback.
R5 (slice drop):    any hard-gated slice business metric below baseline - δ
                      for 2 consecutive cadence windows
                      → halt ramp; human review required.
R6 (manual):        on-call initiated
                      → immediate rollback or step-down; log reason.
```

**Manual override.** A named role can halt or reverse the ramp at any step; the override is logged with reason. The role is named, not a person, so the gate survives turnover.

Senior-engineer notes:

- **1% is not "no users."** For a service at 10 M requests/day, 1% is 100 k requests. Safety gates must hold from step C1.
- **Dwell beats percentage.** The temptation to ramp faster when metrics look good is where most canary regressions ship. The dwell floor is there to accumulate rare events, not to average metrics.
- **Assignment hash is the point of ratchet.** If the hash salt is rotated mid-rollout, every user is re-bucketed and the comparability of before/after breaks. Pin the salt per rollout.
- **Delayed-label gaps.** Business metrics with a lag (e.g., retention, revenue) cannot be read inside a 24 h dwell. The plan declares which metrics are read live and which are read in the post-rollout review.

### 2.4 Rollback and holdback plan

**Rollback.** Rollback is a single action, not a procedure. The plan names:

- The exact command, flag, or feature-flag flip that reverts traffic to the baseline.
- The time budget from signal to 100% baseline (typical target: < 5 min for SLO rollback, < 30 min for quality rollback).
- Who is paged and on what signal.
- The post-rollback artefacts: the request logs preserved, the divergence tables preserved, the incident record opened.
- What state is dirty after rollback (predictions already written, notifications already sent) and how it is reconciled.

**Holdback.** After full rollout, a holdback is a persistent minority slice (commonly 1–5%) that continues to be served by the baseline for a declared period (commonly 2–4 weeks). The holdback is what lets the team measure the delayed-label metrics that canary could not resolve. The plan names:

- The holdback fraction and duration.
- The metrics the holdback resolves (retention, revenue, long-term quality).
- The decision at the end of the holdback window: retire the holdback, extend it, or declare the rollout a regression and roll back.

Senior-engineer notes:

- **A rollback with no preserved state is not a rollback.** If predictions were used to populate downstream caches, the caches are either invalidated on rollback or the rollback is incomplete. Name the cache invalidation in the plan.
- **"Monitor for a week" is not a holdback.** A holdback is a *traffic split*, not a monitoring commitment. Without it, the delayed-label comparison has no counterfactual.
- **Holdback overlap with experimentation platforms.** If the team runs a product A/B framework, the holdback should ride that framework's assignment, not an orthogonal split — otherwise users see two rollouts at once and attribution breaks.

### 2.5 Monitoring plan

The monitoring plan is the operational face of the harness. It is where each gate from §2.2–2.4 becomes a dashboard query and an alert rule.

Per gate, the plan names: the metric, the query (or query id), the cadence it is read, the owner who gets paged, and the dashboard it lives on. A single-table layout:

| Gate | Metric | Query | Cadence | Owner | Dashboard |
|---|---|---|---|---|---|
| Shadow exit | p95 latency, divergence rate on `enterprise` slice | `qid:shadow-latency`, `qid:shadow-div-enterprise` | every 5 min, rolled-up hourly | ML platform on-call | `shadow-rollout` |
| R1 SLO | error rate | `qid:canary-err` | 1 min | serving on-call | `canary-health` |
| R4 quality | business headline | `qid:canary-headline` | hourly | product on-call | `canary-quality` |

The plan integrates with the offline harness: the slices in exercise-01 §2.3 are the slices tracked here; the floor metrics in exercise-01 §2.2 are the hard gates here. Slice drift online is a signal to re-run the offline harness, not to ignore.

### 2.6 Decision log

A table, appended once per stage transition:

| Stage | Date | Decision | Signed off by (role) | Harness ref | Notes |
|---|---|---|---|---|---|

The log is the audit trail. It is not optional; it is the thing a future reviewer reads to understand why a rollout proceeded despite a borderline signal.

### 2.7 Illustrative worked example (not a real system)

> **Illustrative feature** — promoting `router-v12` (from exercise-01 §2.6) from `pass-with-note` offline to full production. This example is a composite for calibration; do not treat it as a real system or incident.

Promotion frame:

- Model under promotion: `router-v12`. Baseline: `router-v11`.
- Harness result: `pass-with-note`, 2026-10-20, note on `short-body` slice (report-only regression of 0.02 PR-AUC).
- Surface: ticket intake form; auto-routes to tier-2 engineering vs. tier-1 support.
- Decision: promote through shadow → canary 1% → 5% → 25% → 50% → 100% → holdback 2% for 3 weeks.
- Last reviewed: 2026-10-22.

Shadow stage:

- Traffic selection: 100% of production traffic mirrored.
- Prediction capture: request id, baseline label + score, candidate label + score, latencies, feature snapshot.
- Divergence metrics: label-agreement rate (overall and per hard-gated slice), confidence histograms, class-distribution PSI, per-slice divergence table.
- Serving health: candidate p95 latency, error rate, feature-staleness rate.
- Exit gate: ≥ 72 h shadow (covers a full week minus weekend asymmetry), candidate p95 ≤ 50 ms, error rate ≤ 0.1%, no slice divergence > 10% on any hard-gated slice, zero candidate-side incidents.
- Side-effect isolation: candidate path does not write to ticket store; does not emit routing events to downstream systems; does not post to the audit log beyond the shadow log.

Canary stage (ramp policy per §2.3):

Assignment: hash by ticket id, salt pinned for this rollout.

- C1 1% / dwell 24 h. Hard gates: R1, R2, R3. Business metrics: agent-accept rate within noise band of baseline. Slices: `enterprise`, `locale:non-en`, `do-not-auto-route`, `cold-start-user`.
- C2 5% / dwell 48 h. Additional check: `last-7-days` slice agent-accept rate.
- C3 25% / dwell 72 h. Additional check: tier-2 team load — do not shift more volume than the team can absorb; this is a product gate, not a model gate.
- C4 50% / dwell 72 h.
- C5 100%.

Rollback triggers:

- R1: error rate > 0.5% over 5 min → auto-rollback.
- R2: false-positive rate on `do-not-auto-route` > 0.5% over 15 min → auto-rollback.
- R3: cost per prediction > $0.0015 over 15 min → step down one level; page.
- R4: agent-accept rate below baseline by ≥ 2 percentage points for 2 hourly cadences → halt ramp; product sign-off required.
- R5: `enterprise` slice agent-accept rate below baseline by ≥ 2 pp for 2 hourly cadences → halt ramp.
- R6: manual override by ML platform on-call or routing product owner at any time.

Rollback plan:

- Command: feature-flag flip `router.candidate_enabled=false`; takes effect within 60 s.
- Dirty state: predictions already made have been routed; routing is additive, not reversible. The plan accepts this and declares that affected tickets are flagged in the post-rollback audit.
- Post-rollback artefacts: shadow and canary logs preserved for 30 days; incident record opened.

Holdback:

- 2% of traffic continues on `router-v11` for 3 weeks post-100%.
- Resolves: 3-week retained-escalation-accuracy rate (delayed label).
- End-of-window decision: retire holdback if retained accuracy within noise; extend for 3 more weeks if borderline; roll back if regressed.

Monitoring plan: as §2.5, table instantiated with the above metrics and queries.

Decision log: appended per stage transition; sign-offs by `ml-platform-on-call` and `routing-product-owner`.

### 2.8 Runnable sketch: a ramp controller

The ramp is a state machine. A minimal shape (Python-like pseudocode):

```python
# rollout/controller.py
from dataclasses import dataclass
from typing import Literal

Stage = Literal["shadow", "C1", "C2", "C3", "C4", "C5", "holdback"]

@dataclass
class StageSpec:
    name: Stage
    traffic_pct: float
    min_dwell_hours: float
    exit_gate_query: str   # id of the query that must return pass

@dataclass
class RolloutState:
    stage: Stage
    entered_at_iso: str
    hash_salt: str
    paused: bool
    paused_reason: str | None

def can_advance(state: RolloutState, spec: StageSpec, now_iso: str, gate_result: dict) -> tuple[bool, str]:
    if state.paused:
        return False, f"paused: {state.paused_reason}"
    if _hours_since(state.entered_at_iso, now_iso) < spec.min_dwell_hours:
        return False, "dwell not satisfied"
    if not gate_result.get("pass", False):
        return False, f"gate not passed: {gate_result.get('reason', 'unknown')}"
    return True, "ok"

def should_rollback(rollback_signals: dict) -> tuple[bool, str]:
    # First-match ordered checks mirror §2.3 rollback triggers.
    for rule in ("R1", "R2", "R3", "R4", "R5", "R6"):
        if rollback_signals.get(rule):
            return True, rule
    return False, ""
```

The controller is intentionally boring. Interesting rollouts are usually the ones where the controller did not prevent a human from improvising past a gate; keep the controller mechanical and the humans accountable in the decision log.

## 3. Validation steps

Walk the artefact through this checklist:

1. **Frame pins versions, not aliases.** Candidate and baseline are both version-pinned.
2. **Shadow isolation is declared.** One sentence: no writes, no downstream calls, no notifications. If the design cannot assert that sentence truthfully, it is a pre-canary A/B.
3. **Shadow exit gate is numeric.** Dwell hours, latency floor, error floor, divergence tolerances per slice.
4. **Canary ramp is a table.** Steps, traffic %, dwell hours, exit gate — not prose.
5. **Assignment is deterministic.** Hash function and salt declared; salt pinned per rollout.
6. **Rollback triggers are ordered and numeric.** R1..Rn with explicit thresholds and windows; first match wins.
7. **Rollback command and time budget are named.** A rollback that takes "as long as it takes" is not a plan.
8. **Dirty-state behaviour is declared.** What happens to predictions already used downstream.
9. **Holdback is a traffic split.** Declared fraction and duration, not just a monitoring commitment.
10. **Monitoring plan integrates offline harness slices and floors.** Not a separate list.
11. **Decision log is in the artefact.** The table is present (even if empty at write time) with the stage / date / decision / role / notes columns.
12. **NIST AI RMF MANAGE / MEASURE** is cited where appropriate for risk treatment layering.
13. **Last reviewed within the declared cadence.**

Static checks:

```bash
grep -c "^R[0-9]" online-rollout.md                    # expect >= 5 rollback rules
grep -c "^Step C[0-9]\|^C[0-9] " online-rollout.md     # canary steps declared
grep -ci "shadow" online-rollout.md                     # shadow stage exists
grep -ci "holdback" online-rollout.md                   # holdback declared
grep -ci "assignment\|hash\|salt" online-rollout.md     # deterministic assignment
grep -ci "last reviewed" online-rollout.md              # must exist
grep -ci "NIST AI RMF" online-rollout.md                # standard cited
```

## 4. Rubric / review checklist

- [ ] Promotion frame: candidate version-pinned, baseline version-pinned, harness result referenced, surface named, last-reviewed date present.
- [ ] Shadow stage: side-effect isolation declared; traffic selection declared; divergence metrics named (overall and per hard-gated slice); numeric exit gate including dwell.
- [ ] Shadow exit gate includes real-load latency measurement under production traffic shape.
- [ ] Canary ramp: table with at least 4 steps (1% → 100% not-in-one-jump), each with traffic %, dwell floor, exit gate.
- [ ] Assignment is deterministic; hash + salt pinned; policy on re-bucketing declared.
- [ ] Rollback triggers: ordered, first-match, numeric thresholds and time windows, at minimum SLO, safety, cost, quality, slice, and manual.
- [ ] Rollback command / flag is a single action; time budget to 100% baseline declared; dirty-state reconciliation declared.
- [ ] Holdback: traffic fraction and duration declared; delayed-label metrics named; end-of-window decision criteria declared.
- [ ] Monitoring plan integrates the offline harness (exercise-01) — slices and floors are shared; divergence from the harness triggers a re-run, not an improvisation.
- [ ] Decision log table present in the artefact (even if rows will be filled during the rollout).
- [ ] NIST AI RMF MANAGE / MEASURE referenced where risk-treatment layering applies.
- [ ] OWASP ML Top 10 and MITRE ATLAS referenced for safety-slice and adversarial-signal gates where the model has a non-trivial safety surface.
- [ ] Last reviewed date within the declared cadence.
- [ ] Plan never advances on "metrics look good, let's skip dwell" — dwell is a floor.

Review-weighted points:

- Frame specificity and integration with the offline harness result: 10%
- Shadow design (isolation, divergence metrics, exit gate): 20%
- Canary ramp (step table, assignment determinism, dwell discipline): 20%
- Rollback + holdback (ordered triggers, single-action rollback, dirty-state, delayed-label plan): 25%
- Monitoring plan (integrates harness slices/floors, named owners, cadence): 15%
- Decision log, review cadence, and standards alignment (NIST AI RMF, OWASP ML, MITRE ATLAS where applicable): 10%

## 5. Common mistakes

- **"Shadow" with side effects.** A shadow deployment that still writes to a store, calls a downstream API, or emits a notification is not shadow. Users (or systems) see the candidate's output and the rollout's safety posture collapses.
- **Reading quality from shadow.** Shadow produces divergence, not correctness. Business-metric decisions belong in canary with real users.
- **No dwell floor.** "Looks good, let's ramp" is how most canary regressions ship. Dwell exists to accumulate rare events; it is a floor, not an aspiration.
- **Rotating the assignment salt mid-rollout.** This re-buckets every user and destroys before/after comparability. Pin the salt per rollout.
- **Rollback by procedure, not by command.** A rollback that requires a meeting is not a rollback. It is a single action — flag, flip, revert — with a declared time budget.
- **No dirty-state plan.** Rollback that leaves inconsistent downstream state (already-sent emails, already-populated caches) is incomplete. Name the inconsistency; name the reconciliation.
- **Treating holdback as monitoring.** A holdback is a traffic split. Without it, delayed-label metrics have no counterfactual and the post-rollout review is gut-feel.
- **Ignoring the long tail.** 1% canary at moderate QPS still sees thousands of users — safety gates must hold there. The "it's only 1%" instinct is how safety incidents get framed as "early signal."
- **Orthogonal canary and product A/B frameworks.** If the team runs product experiments, canary that assigns independently from the experiment framework double-exposes users and breaks attribution. Use the shared framework or document the independence and its cost.
- **Divergence-only canary.** Canary that reports only "candidate vs. baseline divergence" without reading the business metric has not left shadow in any meaningful sense.
- **Hard-gated slices offline, soft-gated online.** If a slice is hard-gated in the offline harness, it must remain hard-gated online. Loosening gates for the canary is a quiet policy change.
- **Missing decision log.** Rollouts that proceed without a dated, named-role sign-off at each stage are unauditable; the first post-mortem exposes the gap and it is too late.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE and MANAGE functions underpin the ordered, documented risk-treatment layering in the ramp (canary gates) and rollback / holdback phases.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Referenced in the safety gates (R2) for models with policy surfaces, where online misuse and prompt-injection-adjacent patterns must be watched during canary.
- MITRE ATLAS — <https://atlas.mitre.org/>. Referenced in the shadow-stage divergence design when adversarial tactics (evasion, model inversion) are expected on the live stream and divergence between baseline and candidate may be the first signal.

Local context (authoritative for the staged-ramp taxonomy and the gate ordering):

- Paired lesson: `lessons/mod-305-advanced-evaluation/exercises/exercise-02-shadow-and-canary-deploy-design.md`.
- Module README: `modules/mod-305-advanced-evaluation/README.md`.
- Related exercise (offline harness that this plan is promoted from): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (production-readiness scoring that audits the rollout plan): `modules/mod-305-advanced-evaluation/exercise-03-ml-test-score-review-packet/SOLUTION.md`.
