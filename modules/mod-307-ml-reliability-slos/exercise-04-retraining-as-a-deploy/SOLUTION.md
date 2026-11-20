# exercise-04 — Retraining As A Deploy (Reference Solution)

Paired exercise: `lessons/mod-307-ml-reliability-slos/exercises/exercise-04-retraining-as-a-deploy.md`.

## 1. Solution overview

This exercise produces `retrain-deploy.md` plus a small runnable artefact (`state_machine.py`) — a written retraining-as-a-deploy specification backed by the state machine that a scheduler can execute. The artefact has seven parts:

1. A **retraining frame** that pins down what is being retrained, what triggers a retrain, what decision a successful retrain feeds, and who owns the surface the retrained model serves.
2. A **state machine** — the explicit phases a retrain traverses from "triggered" to "retired," with the precondition, invariants, and exit gates per phase. A retrain without a state machine is a cron job hoping for a canary.
3. A **canary policy** — the shadow and canary stages, the slice-level and overall thresholds, the dwell times, and the pre-registered advance and rollback rules.
4. A **rollback plan** — pre-registered, named-role rollback of a promoted model when the production slice regresses post-promotion, cross-referenced to `exercise-03`'s rollback classes.
5. An **artefact and lineage contract** — what gets stored for every retrain run (dataset snapshot, training config, metrics, SBOM-equivalent for ML: dataset version, feature version, library versions, random seed, hardware fingerprint), so that a retrain is reproducible and auditable.
6. A **guardrail suite** — the pre-registered checks a candidate model must pass before promotion, aligned with the offline harness from `mod-305/exercise-01`, the SLOs from `exercise-01`, the cost policy from `exercise-02`, and the safety probes informed by OWASP ML Top 10 and MITRE ATLAS.
7. A **review and cadence policy** — the trigger mix (scheduled, drift-driven, performance-driven, safety-driven), the throttle rules that prevent flapping, and the human sign-off that gates large model changes.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric.

The exercise exists because retraining a production model is a deploy. It changes what the service returns, it can regress on slices the canary did not cover, it can burn error budget silently, and it can introduce safety regressions that pass every automated check. A retrain run by a Monday-morning cron that promotes on "val-accuracy up" is a landmine; a retrain run through a state machine with canary gates, cost admission, and pre-registered rollback is a deploy. NIST AI RMF MAP grounds the pre-deploy categorization of the retrain (purpose, context, impact), MEASURE grounds the pre-promotion evaluation, and MANAGE grounds the canary and rollback as operational risk controls. OWASP ML Top 10 scenarios around data poisoning and training-data integrity are the reason the dataset-lineage contract in §2.5 is non-optional.

## 2. Worked answer / implementation

### 2.1 Retraining frame

Six lines, in this order:

- **What is retrained.** The model family, name, and surface it serves. Pinned. "The ranker" is not a target; "`reco-model` serving `reco-serve/homepage` for logged-in US web users" is. If multiple model versions share a surface (champion/challenger, per-segment models), the frame names which is being retrained.
- **Triggers.** The ordered list of events that start a retrain: scheduled (cron), drift-driven (drift SLI burns past threshold, per `exercise-01`), performance-driven (online quality SLI burns), data-driven (new labelled dataset accepted), safety-driven (new adversarial example class discovered). Each trigger has a "will start a retrain" guard (see §2.6).
- **Decision fed.** Promote to default after canary; keep as warm shadow for one week; retire. The decision determines the promotion threshold stringency.
- **Named owners.** Model owner (primary), data owner (dataset custody), SRE / on-call (serving surface), safety reviewer (for surfaces with policy surface area). Roles, not people.
- **Dependencies.** Feature store, training infrastructure, canary infrastructure, cost-policy quota (see `exercise-02`), rollback target retention (`exercise-03`). A retrain without each dependency's SLO is a retrain whose failure modes are unbounded.
- **Last reviewed.** A date, within the policy's declared review cadence.

### 2.2 State machine

Retraining as a deploy is a state machine with explicit transitions. The machine runs in a scheduler (not in an unversioned script) and emits events the on-call can see.

```
                       +---------+
                       | TRIGGERED |
                       +-----+-----+
                             | pre-admit (budget, lineage, lock)
                             v
                       +-----+-----+
                       | ADMITTED  |
                       +-----+-----+
                             | training started
                             v
                       +-----+-----+
                       |  TRAINING |
                       +-----+-----+
          training failed  /       \  training ok
                          /         \
                         v           v
                 +---------+     +----+------+
                 |  FAILED  |    | EVALUATED |
                 +---------+    +----+------+
                                     | offline guardrails ok
                                     v
                              +------+------+
                              | SHADOWING   |
                              +------+------+
                                     | shadow gates ok
                                     v
                              +------+------+
                              |  CANARYING  |
                              +------+------+
                 canary fail  /       \  canary ok
                             /         \
                            v           v
                   +--------+      +----+-----+
                   | ROLLED_BACK |  | PROMOTED |
                   +-------------+  +----+-----+
                                         | post-promotion monitor ok
                                         v
                                    +----+-----+
                                    |  STABLE  |
                                    +----+-----+
                                         | retention window elapsed
                                         v
                                    +----+-----+
                                    | RETIRED  |
                                    +----------+
```

The state machine is explicit because each state has a **precondition**, an **invariant**, and an **exit gate**:

- **TRIGGERED**: trigger recorded with cause (scheduled / drift / performance / data / safety) and dataset-snapshot pin.
- **ADMITTED** (precondition for TRAINING): cost admission passes (`exercise-02`); concurrency lock on this model acquired (no two concurrent retrains on the same model); dataset-lineage record created.
- **TRAINING**: resource budget consumed per plan; checkpoint cadence is on schedule; no training-loss NaNs; invariant that training logs include dataset version, feature version, library versions, random seed. Exit gate: training completed with final metrics recorded; checkpoint written to the artefact store.
- **EVALUATED**: offline harness (`mod-305/exercise-01`) runs. Exit gate: overall and per-slice offline metrics meet thresholds; no guardrail slice regressed beyond tolerance; safety probes pass; cost-per-request estimate within surface's cost SLO.
- **SHADOWING**: candidate runs in parallel with champion; predictions are logged but discarded. Exit gate: shadow runs for pre-registered dwell (e.g., 24h); prediction-drift vs. champion within tolerance; latency and resource usage within budget; no crash on real traffic shapes.
- **CANARYING**: candidate serves a small fraction of traffic (e.g., 1% → 5%). Exit gates: see §2.3.
- **PROMOTED**: candidate replaces champion; previous champion retained as warm rollback target for ≥ `R_rollback` days (see §2.4).
- **STABLE**: post-promotion monitoring passes for the dwell window. Exit gate: no quality SLI or safety probe regression beyond tolerance.
- **RETIRED**: surpassed by the next retrain; retained in cold storage for audit per the data-retention policy.
- **ROLLED_BACK**: previous champion restored; candidate archived with the failure cause; incident per `exercise-03` if the rollback was mid-canary.
- **FAILED**: training did not complete; artefacts of the partial run are retained for diagnosis; next trigger is re-admitted through the full state machine.

A retrain that is not representable in this state machine is a retrain whose failure modes are not pre-registered.

### 2.3 Canary policy

The canary is the ramp (`mod-306/exercise-01`) applied to a retrained model. The policy specifies the ramp steps, dwell times, exit gates, and rollback triggers.

**Default ramp shape.**

```
C0  Shadow:   0% user-facing exposure. Candidate scores every request in
              parallel; scores are logged, not served.
              Dwell: 24h (spans a full day of traffic shape).
              Exit gate:
                - Score-distribution drift between candidate and champion
                  within pre-registered tolerance.
                - Candidate latency at full traffic volume within SLO
                  (exercise-01 latency SLO).
                - Candidate cost per request within cost SLO (exercise-02 C1).
                - Zero crash rate on candidate path.
                - Prediction-coverage parity: candidate returns a response
                  for every input the champion does.

C1  1% canary: 1% of eligible users randomly assigned to candidate, 99%
               to champion. Assignment by hash of user_id with a salted
               canary salt; SRM-safe (see mod-306/exercise-02).
               Dwell: 2–3 days.
               Exit gate:
                 - No SRM at the 1% split (chi-square p ≥ 0.01).
                 - Overall online quality SLI within tolerance of champion.
                 - Per-slice quality SLI within tolerance for every named
                   guardrail slice (worst-slice protected-attribute cohort,
                   worst-slice revenue cohort, worst-slice new-user cohort).
                 - Latency SLO budget burn attributable to canary ≤ 10%.
                 - Safety probe set pass rate ≥ champion's pass rate.

C2  5% canary: 5% / 95%. Dwell: ≥ 1 week (at least one full weekly cycle).
               Exit gate:
                 - Same as C1 with tighter CIs given more exposure.
                 - Secondary metrics (per the paired online experiment plan)
                   consistent with hypothesis.

C3  Decision: Promote to 100% (candidate becomes champion) or roll back
              (candidate archived with failure cause). Decision is made at
              the end of C2 dwell by the model owner with safety-reviewer
              sign-off for surfaces with a policy surface.
```

**Pre-registered rollback triggers** (fire without a readout):

- **Hard SRM** (p < 0.001 chi-square on assignment) — pause + diagnose.
- **Safety probe regression** — any regression on a named OWASP ML Top 10-aligned or MITRE ATLAS-aligned probe set beyond the pre-registered tolerance. Rollback is immediate, not at the canary exit.
- **Worst-slice quality regression** beyond the pre-registered per-slice tolerance (e.g., protected-attribute slice drops by > 2 pp). Rollback is immediate.
- **Latency SLO burn** attributable to the canary exceeding a pre-registered share of the error budget.
- **Cost-cap breach** on the candidate's per-request cost exceeding the per-request cost SLO.

**Manual override.** A human can slow the canary. A human can only speed it with a logged role, date, reason, and counterfactual (what the policy said should happen).

**Post-promotion monitoring.** After the candidate becomes champion, the state machine remains in SHADOWING state for the (now-ex) champion — the previous champion is warm and shadowed so a post-promotion regression can be rolled back without a cold start. The warm shadow runs for the rollback-retention window `R_rollback` (see §2.4).

### 2.4 Rollback plan

Rollback aligns with `exercise-03`'s rollback classes, specialized to retraining.

**Rollback classes.**

- **In-canary rollback** (candidate is still at ≤ 5% traffic). RTO: ≤ 2 minutes (config change, serving infra already has both candidate and champion warm). Precondition: previous champion is still the primary route, no serving state has been mutated. Action: deassign canary traffic.
- **Post-promotion rollback** (candidate was promoted to 100%; regression surfaced post-promotion). RTO: ≤ 5 minutes. Precondition: previous champion was retained warm for `R_rollback` days. Action: re-pin serving to previous champion; archive post-promotion candidate with failure cause.
- **Dataset rollback.** When a retrain is rolled back because the dataset was bad (data-poisoning, labeling-pipeline bug, upstream data outage), the dataset snapshot is marked as bad in the dataset registry so future retrains do not re-use it. This is specific to retraining and does not have an equivalent in generic deploy rollback.
- **Pipeline rollback.** When the retrain *pipeline* itself was bad (training code bug, metric computation bug), the pipeline version is pinned back; in-flight candidates are discarded.

**Retention policy.**

- `R_rollback ≥ 28 days` for the previous champion of any surface with an SLO. The number is chosen so a post-promotion regression that surfaces after one monthly reporting cycle still has a rollback target.
- Warm-shadow serving of the previous champion is on for the first 7 days after promotion; cold-storage retrieval after that.
- Training-run artefacts (checkpoints, logs, metrics) retained per the organization's audit policy; for surfaces with regulatory exposure, the policy names the statute's retention requirement.

**Rollback decision authority.**

- In-canary rollback: on-call or model owner.
- Post-promotion rollback within 24h: model owner.
- Post-promotion rollback after 24h and before `R_rollback` expires: model owner with product-owner notification.
- Rollback after `R_rollback` (previous champion cold): requires reconstitution (promote from cold storage, re-run shadow and C1 canary) — effectively a new deploy.

### 2.5 Artefact and lineage contract

A retrain whose dataset, config, and run cannot be reproduced is a retrain whose failure cannot be investigated. The contract names what is stored and how it is linked.

**Dataset lineage.**

- **Dataset snapshot ID.** The immutable identifier of the training-time dataset. The identifier references not just the data but the pipeline that produced it (feature store version, label pipeline version, filtering query version).
- **Dataset ancestry.** The chain of upstream datasets. If the dataset is a filter of a larger pool, the filter predicate and the pool's version are both recorded.
- **Label provenance.** For supervised tasks, where the labels came from (human annotation pipeline, system-generated signals, implicit feedback) and the version of that pipeline.
- **Known-bad list.** Datasets flagged as bad from prior rollback events. The retrain pre-flight check refuses to train on a known-bad dataset.
- **Poisoning check.** For data sources that include user-supplied content (RAG indices, feedback-based labels), the policy names the content-filtering and anomaly-detection step that runs before the dataset is accepted (aligned with OWASP ML Top 10 data-poisoning scenario).

**Training config.**

- Model architecture version, loss function version, optimizer, hyperparameters, early-stopping config, random seed.
- Framework and library versions (torch, jax, transformers, etc.), down to patch.
- Hardware fingerprint (GPU type, driver version). Different hardware can produce bit-divergent models; the audit trail names the hardware class.

**Run metadata.**

- Start time, end time, compute cost in the surface's cost unit, exit status, operator (role; "scheduler" if automated).
- Training metrics per epoch; final metrics; validation and held-out test metrics; per-slice metrics for every guardrail slice declared in the offline harness.
- Shadow and canary metrics as the state machine progressed.

**Linkage.**

- Every state transition in §2.2 writes an event with the run ID, dataset snapshot ID, training config version, and operator. A reviewer can replay a retrain from the event log alone.

**Integrity.**

- Artefacts are content-addressed (hash-named) in the artefact store. A rename that collides is detected; a deletion that invalidates a reference is flagged.

### 2.6 Guardrail suite

The guardrails are the pre-registered checks a candidate must pass to leave EVALUATED and again to leave CANARYING. They are the point where this exercise stitches into the rest of the curriculum.

**Offline guardrails** (EVALUATED exit gate):

- Overall metric meets threshold (above champion or above absolute floor, per policy).
- Every guardrail slice within tolerance of champion or within policy floor.
- Safety probe set pass rate ≥ champion's; probe set reflects OWASP ML Top 10 scenarios (prompt injection, output integrity, data poisoning for retrained safety classifiers) and MITRE ATLAS tactics.
- Calibration within tolerance on canary slice.
- Cost-per-prediction estimate within surface's cost SLO (`exercise-02`).
- Latency profile within surface's latency SLO (`exercise-01`).

**Online guardrails** (CANARYING exit gate):

- SRM-clean at every canary step.
- Quality SLI unchanged or improved on the overall and per-slice.
- Latency SLO burn attributable to canary within the pre-registered allowance.
- Cost SLI unchanged.
- Safety signals unchanged or improved.
- No user-reported regression tagged against the candidate's exposure window.

**Post-promotion guardrails** (STABLE exit gate):

- All online guardrails hold for the post-promotion dwell window (default 7 days).
- Previous champion warm shadow has not been invoked (no rollback).

A candidate that passes all three layers is promoted. A candidate that passes offline but fails online is rolled back; the offline harness is reviewed for the slice or signal it missed (feedback into `mod-305/exercise-01`).

### 2.7 Review and cadence

**Trigger mix.**

- **Scheduled** — e.g., weekly or monthly, chosen from traffic volume and drift pace. A weekly scheduled retrain is overkill for a stable surface and under-powered for a volatile one; the number is argued.
- **Drift-driven** — drift SLI from `exercise-01` burns past an alerting threshold, scheduler queues a retrain. The threshold is tighter than the SLO target because the retrain is a leading indicator.
- **Performance-driven** — online quality SLI burns; scheduler queues a retrain. The retrain is a mitigation, not a fix; the incident still goes through `exercise-03`.
- **Data-driven** — a new labelled dataset accepted into the registry triggers a retrain IF the dataset meaningfully extends coverage. "Meaningfully" is a pre-registered threshold on dataset-size delta or slice coverage.
- **Safety-driven** — a new adversarial-example class is discovered or an OWASP ML Top 10-aligned probe set is updated; scheduler queues a retrain of the surfaces whose safety filter uses that class.

**Throttle rules** (prevent flapping):

- Minimum inter-retrain interval per surface: e.g., ≥ 24h unless a safety trigger overrides.
- Maximum concurrent retrains per surface: 1. The state machine's concurrency lock enforces this.
- Back-off on repeated failures: if the last two runs failed in EVALUATED, the next trigger pauses pending human review; the scheduler does not keep burning compute.

**Human sign-off.**

- Scheduled retrains: automated; promotion requires only the automated gates passing.
- Drift- or performance-driven retrains: automated; promotion requires automated gates + a model owner acknowledging the incident chain.
- Data-driven retrains: automated when the dataset extends coverage without changing label semantics; model-owner sign-off when label semantics change.
- Safety-driven retrains: automated training; promotion requires safety-reviewer sign-off plus model-owner sign-off.

The sign-off is a role, not a person. The state machine refuses to transition past CANARYING without the required role's logged approval.

**Review cadence.**

Quarterly review by the model owner and the SRE / on-call: trigger mix is still correct (too many false triggers → throttle; too few → add); retention policy is still correct; rollback-decision authority is still correct.

### 2.8 Illustrative worked example (not a real system)

> **Illustrative feature** — `reco-model` serving `reco-serve/homepage` (logged-in US web users, random visitors excluded). Retraining cadence target: weekly scheduled, with drift and performance triggers. Composite for calibration; do not treat as a real system.

Retraining frame:

- What is retrained: `reco-model` family, variant `reco-ranker-v*`, on the `reco-training-core` dataset family.
- Triggers: weekly scheduled on Sunday; drift-driven when F5 drift SLI burns past 20% of its monthly budget in a week; performance-driven on online CTR SLI burn; safety-driven on policy-filter probe-set regression.
- Decision fed: promote after canary; retain previous champion warm for 28 days; retire after.
- Named owners: `team-reco-models` (primary), `team-data-platform` (dataset custody), `team-reco-serve` (surface SRE), `team-trust-safety` (safety reviewer).
- Last reviewed: 2026-11-10.

State machine (illustrative key parameters):

- Concurrency lock: 1 per surface.
- Shadow dwell: 24h.
- C1 canary: 1% for 2 days.
- C2 canary: 5% for 7 days (one weekly cycle).
- `R_rollback`: 28 days warm (7 days warm shadow, 21 days cold-but-fast-reconstitution).

Canary policy:

- Slice guardrails: worst-slice CTR for `new-user-7d`, `locale=en`, `locale=non-en` cohorts; no regression > 1 pp absolute; no worst-slice policy-flag-rate increase.
- Safety probe: `reco-abuse-probe-v4` pass rate ≥ champion's; immediate rollback on regression.
- Latency burn allowance attributable to canary: ≤ 10% of the surface's weekly latency budget.

Rollback classes exercised: in-canary (fast), post-promotion within 24h (fast), post-promotion day 10 (fast while warm), day 25 (cold-fast), day 30 (requires reconstitution; equivalent to a new deploy).

Guardrail suite drawn from offline harness `offline-harness-v3.1` (`mod-305/exercise-01` paired artefact) and SLO set `reco-serve-slo-v2` (`exercise-01` paired artefact).

Trigger review: weekly scheduled retrain has produced 6 promotions, 2 rollbacks, 1 FAILED-in-TRAINING run in the last quarter; next review will consider whether weekly is too frequent given the surface's drift pace.

### 2.9 Runnable sketch: `state_machine.py`

```python
# state_machine.py
"""Reference retrain state machine.

Design goal: explicit state transitions, gated by precondition callables.
A scheduler plugs its real gate implementations into this skeleton.
"""

from dataclasses import dataclass, field
from enum import Enum
from typing import Callable, Optional


class State(str, Enum):
    TRIGGERED = "TRIGGERED"
    ADMITTED = "ADMITTED"
    TRAINING = "TRAINING"
    FAILED = "FAILED"
    EVALUATED = "EVALUATED"
    SHADOWING = "SHADOWING"
    CANARYING = "CANARYING"
    ROLLED_BACK = "ROLLED_BACK"
    PROMOTED = "PROMOTED"
    STABLE = "STABLE"
    RETIRED = "RETIRED"


@dataclass(frozen=True)
class Transition:
    src: State
    dst: State
    name: str
    guard: Callable[["RetrainRun"], bool]
    on_enter: Optional[Callable[["RetrainRun"], None]] = None


@dataclass
class RetrainRun:
    run_id: str
    surface: str
    dataset_snapshot_id: str
    config_version: str
    operator_role: str
    cause: str  # 'scheduled' | 'drift' | 'performance' | 'data' | 'safety'
    state: State = State.TRIGGERED
    history: list[tuple[State, State, str]] = field(default_factory=list)

    def transition(self, transition: Transition) -> None:
        if self.state != transition.src:
            raise RuntimeError(
                f"illegal transition {transition.src} -> {transition.dst} "
                f"from state {self.state}"
            )
        if not transition.guard(self):
            raise RuntimeError(f"guard failed for transition {transition.name}")
        self.history.append((self.state, transition.dst, transition.name))
        self.state = transition.dst
        if transition.on_enter is not None:
            transition.on_enter(self)


# Example guard stubs. A real deployment wires these to the cost-policy,
# lineage registry, offline harness, and canary-telemetry systems.

def _budget_available(run: RetrainRun) -> bool:
    # Delegates to exercise-02's cost-policy admission.
    return True  # pretend.

def _lineage_recorded(run: RetrainRun) -> bool:
    return bool(run.dataset_snapshot_id) and bool(run.config_version)

def _training_ok(run: RetrainRun) -> bool:
    return True

def _offline_guardrails_pass(run: RetrainRun) -> bool:
    return True

def _shadow_gates_pass(run: RetrainRun) -> bool:
    return True

def _canary_gates_pass(run: RetrainRun) -> bool:
    return True

def _post_promotion_monitor_ok(run: RetrainRun) -> bool:
    return True


TRANSITIONS: list[Transition] = [
    Transition(State.TRIGGERED, State.ADMITTED, "admit",
               guard=lambda r: _budget_available(r) and _lineage_recorded(r)),
    Transition(State.ADMITTED, State.TRAINING, "start_training",
               guard=lambda r: True),
    Transition(State.TRAINING, State.EVALUATED, "training_ok",
               guard=_training_ok),
    Transition(State.TRAINING, State.FAILED, "training_fail",
               guard=lambda r: not _training_ok(r)),
    Transition(State.EVALUATED, State.SHADOWING, "offline_ok",
               guard=_offline_guardrails_pass),
    Transition(State.EVALUATED, State.ROLLED_BACK, "offline_fail",
               guard=lambda r: not _offline_guardrails_pass(r)),
    Transition(State.SHADOWING, State.CANARYING, "shadow_ok",
               guard=_shadow_gates_pass),
    Transition(State.SHADOWING, State.ROLLED_BACK, "shadow_fail",
               guard=lambda r: not _shadow_gates_pass(r)),
    Transition(State.CANARYING, State.PROMOTED, "canary_ok",
               guard=_canary_gates_pass),
    Transition(State.CANARYING, State.ROLLED_BACK, "canary_fail",
               guard=lambda r: not _canary_gates_pass(r)),
    Transition(State.PROMOTED, State.STABLE, "post_promotion_ok",
               guard=_post_promotion_monitor_ok),
    Transition(State.PROMOTED, State.ROLLED_BACK, "post_promotion_fail",
               guard=lambda r: not _post_promotion_monitor_ok(r)),
    Transition(State.STABLE, State.RETIRED, "retention_elapsed",
               guard=lambda r: True),
]


def find_transition(src: State, dst: State) -> Transition:
    for t in TRANSITIONS:
        if t.src == src and t.dst == dst:
            return t
    raise LookupError(f"no transition {src} -> {dst}")


if __name__ == "__main__":
    run = RetrainRun(
        run_id="rr-0001",
        surface="reco-serve/homepage",
        dataset_snapshot_id="ds-2026-11-15-abc",
        config_version="train-cfg-v4",
        operator_role="scheduler",
        cause="scheduled",
    )
    for src, dst in [
        (State.TRIGGERED, State.ADMITTED),
        (State.ADMITTED, State.TRAINING),
        (State.TRAINING, State.EVALUATED),
        (State.EVALUATED, State.SHADOWING),
        (State.SHADOWING, State.CANARYING),
        (State.CANARYING, State.PROMOTED),
        (State.PROMOTED, State.STABLE),
        (State.STABLE, State.RETIRED),
    ]:
        run.transition(find_transition(src, dst))
    for h in run.history:
        print(h)
```

The sketch is intentionally skeletal. The point is that the state machine is **explicit**: every transition names its guard; an unnamed transition is a bug. A real deployment plugs the guard functions into cost, lineage, harness, and canary subsystems.

## 3. Validation steps

Walk the retrain policy through this checklist:

1. **Retraining frame names what, when, decision fed, owners, dependencies, and last-reviewed date.**
2. **State machine has at least the states** TRIGGERED, ADMITTED, TRAINING, EVALUATED, SHADOWING, CANARYING, PROMOTED, STABLE, RETIRED, plus failure branches ROLLED_BACK and FAILED.
3. **Each state has a precondition, invariant, and exit gate** named in prose.
4. **Canary policy has ≥ 3 ramp steps** (shadow → 1% → 5% → full) with dwell times and exit gates per step.
5. **Pre-registered rollback triggers** include SRM, safety probe regression, worst-slice regression, latency burn, cost-cap breach.
6. **Rollback classes** cover in-canary, post-promotion within retention, dataset rollback, pipeline rollback.
7. **`R_rollback` is ≥ 28 days** with a documented rationale.
8. **Lineage contract names** dataset snapshot, dataset ancestry, label provenance, known-bad list, poisoning check, training config (framework, library, seed, hardware), run metadata (metrics, cost, operator), and content-addressed artefacts.
9. **Guardrail suite is layered**: offline, online, post-promotion; each layer has a pre-registered threshold.
10. **Safety probe sets reference** OWASP ML Top 10 scenarios and MITRE ATLAS tactics.
11. **Trigger mix is explicit** (scheduled / drift / performance / data / safety) with throttle rules and inter-retrain minimum interval.
12. **Human sign-off roles named** per trigger class; state machine blocks the promotion transition without the required role.
13. **Cross-references to `exercise-01` (SLOs that gate latency/cost/safety), `exercise-02` (cost admission), `exercise-03` (rollback classes and incident response), and `mod-305/exercise-01` (offline harness)** are present.

Static checks to drop into CI on `retrain-deploy.md`:

```bash
grep -ciE "state machine|precondition|invariant|exit gate"  retrain-deploy.md
grep -ciE "shadow|canary|promote|roll ?back"                retrain-deploy.md
grep -ciE "dataset snapshot|lineage|seed|hardware"          retrain-deploy.md
grep -ciE "drift|performance|safety|scheduled"              retrain-deploy.md   # trigger mix present
grep -ciE "nist ai rmf|owasp|mitre atlas"                   retrain-deploy.md
grep -ciE "last reviewed"                                   retrain-deploy.md
python state_machine.py                                      # happy path runs
```

For `state_machine.py`, a smoke test asserts: (a) the happy path TRIGGERED → RETIRED runs without raising; (b) an illegal transition raises `RuntimeError`; (c) a transition whose guard returns False raises; (d) the history log is non-empty and ordered.

## 4. Rubric / review checklist

- [ ] Retraining frame names the model, triggers, decision fed, roles (model owner, data owner, SRE, safety reviewer), dependencies, and last-reviewed date.
- [ ] State machine covers TRIGGERED, ADMITTED, TRAINING, EVALUATED, SHADOWING, CANARYING, PROMOTED, STABLE, RETIRED plus ROLLED_BACK and FAILED branches.
- [ ] Each state has a precondition, invariant, and exit gate stated in prose.
- [ ] Canary policy has ≥ 3 ramp steps (shadow → 1% → 5% → full) with dwell times, exit gates, and pre-registered rollback triggers per step.
- [ ] Slice-level guardrails named (worst-slice protected / revenue / new-user cohorts) with numeric tolerances.
- [ ] Safety probe set reflects OWASP ML Top 10 scenarios and MITRE ATLAS tactics.
- [ ] Rollback classes: in-canary, post-promotion within retention, dataset rollback, pipeline rollback. RTO/RPO named per class.
- [ ] Retention window `R_rollback ≥ 28 days` with a documented rationale.
- [ ] Rollback decision authority named per class (on-call, model owner, product-owner notification).
- [ ] Artefact and lineage contract covers dataset snapshot, dataset ancestry, label provenance, known-bad list, poisoning check, training config (framework, library versions, seed, hardware), run metadata, content-addressed artefact storage.
- [ ] Guardrail suite is layered (offline / online / post-promotion) with pre-registered thresholds per layer.
- [ ] Trigger mix explicit (scheduled, drift, performance, data, safety) with throttle rules (minimum interval, concurrency = 1, back-off on repeated failure).
- [ ] Human sign-off roles named per trigger class; state machine blocks the PROMOTION transition without the required role.
- [ ] NIST AI RMF MAP (pre-deploy categorization), MEASURE (pre-promotion evaluation), and MANAGE (canary/rollback as risk controls) cited.
- [ ] Cross-references to `exercise-01` (SLOs), `exercise-02` (cost admission), `exercise-03` (rollback classes, incident response), `mod-305/exercise-01` (offline harness), and `mod-306/exercise-01` (canary ramp shape).

Review-weighted points for grading:

- Retraining frame and owner clarity: 10%
- State machine (states, preconditions, invariants, exit gates): 25%
- Canary policy (ramp, gates, rollback triggers): 20%
- Rollback plan (classes, RTO/RPO, retention, decision authority): 15%
- Lineage and artefact contract: 15%
- Trigger mix, throttles, and sign-off: 10%
- Standards alignment (NIST AI RMF MAP/MEASURE/MANAGE, OWASP ML Top 10, MITRE ATLAS) and hygiene: 5%

## 5. Common mistakes

- **Retrain as a cron job, not a deploy.** A schedule that runs `train.py && push_model.py` on Monday without a canary gate promotes regressions on schedule. The state machine is the discipline.
- **No shadow stage.** Jumping from EVALUATED to CANARYING skips the check that the candidate behaves on real traffic *at full load without user exposure*. A latency or crash pathology on real traffic shape should not be discovered at 1% canary.
- **Canary gates on overall metrics only.** A candidate that improves overall and regresses a protected-slice cohort has not improved; worst-slice guardrails are first-class.
- **Promotion threshold is "val-accuracy up."** Validation accuracy is the floor, not the ceiling. The guardrail suite has at least three layers and includes safety, cost, and latency.
- **No pre-registered rollback triggers.** A rollback that waits for a meeting is a rollback that is late. Triggers are named conditions.
- **No lineage.** A retrain whose dataset, seed, hardware, and framework versions are not captured is a retrain whose failures cannot be reproduced or defended in audit.
- **Dataset not snapshotted at training time.** Retraining on a "live view" of the training data means two runs with the "same" dataset can diverge; the snapshot ID is immutable and references a point-in-time copy.
- **Known-bad datasets re-used.** A dataset that caused a rollback and is not flagged in the registry will cause the next retrain to fail the same way. The known-bad list is enforced.
- **Poisoning check missing on user-influenced datasets.** RAG corpora, feedback-labelled datasets, and public-crawl sources all admit data poisoning (OWASP ML Top 10). A retrain without a content-integrity pre-check is a retrain with an open attack surface.
- **Concurrency not locked.** Two retrains running on the same surface at once produce two promotion candidates; the scheduler needs a per-surface lock.
- **Retention window too short.** `R_rollback = 7 days` makes a month-end regression unrecoverable because the previous champion is gone. 28 days is the common floor.
- **Previous champion cold at promotion time.** A rollback that has to reconstruct the previous champion from cold storage has an RTO measured in hours, not minutes. Warm-shadow serving for the first 7 days post-promotion is cheap insurance.
- **Scheduled retrains at a cadence disconnected from drift pace.** Weekly scheduled retrains for a stable surface burn compute; monthly scheduled retrains for a volatile surface lag. The cadence is argued from the surface, not chosen by default.
- **Safety-driven retrains sharing the queue with scheduled retrains.** When a safety probe regression is discovered, the retrain that addresses it jumps the queue and bypasses the minimum-interval throttle. The policy names the override.
- **Flapping — retrains chasing retrains.** Without a back-off rule, a failing retrain re-triggers on the next drift burn, trains, fails, re-triggers; the compute bill doubles with no fix. The back-off rule pauses pending review after N failures.
- **No link to the offline harness.** A retrain policy that re-invents its own evaluation diverges from the one `mod-305/exercise-01` already specifies. The guardrail suite references the harness rather than restating it.
- **Promotion promoted on correlation, not causation.** A candidate's online metrics may look better than the champion's for reasons unrelated to the model (traffic shift, seasonality). The canary assignment is randomised and the comparison is against the concurrent champion arm, not against the pre-launch baseline.
- **Human sign-off left for "someone on the team."** Sign-off by role, not person, so the state machine refuses to promote without the correct role regardless of who is around.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MAP function grounds the pre-deploy categorization of the retrain (purpose, context, intended users, impact scope) that determines promotion-threshold stringency. MEASURE function grounds the pre-promotion evaluation and canary guardrails. MANAGE function grounds the canary and rollback as operational risk controls.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source for data-poisoning scenarios (lineage contract's poisoning check), prompt injection and output-integrity scenarios (safety probe sets), and model-denial-of-service scenarios (cost-per-request guardrail).
- MITRE ATLAS — <https://atlas.mitre.org/>. Named adversarial tactics (training-data poisoning, model extraction, inference abuse) that the safety probe set and the dataset-provenance checks are built against.

Local context (authoritative for the state machine shape, the canary ramp, and the cross-references):

- Paired lesson: `lessons/mod-307-ml-reliability-slos/exercises/exercise-04-retraining-as-a-deploy.md`.
- Module README: `modules/mod-307-ml-reliability-slos/README.md`.
- Related exercise (SLOs that the retrain's guardrails reference and whose budget pre-admission gates the retrain): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (cost policy whose admission control gates the TRAINING state entry): `modules/mod-307-ml-reliability-slos/exercise-02-cost-budgets-and-quotas/SOLUTION.md`.
- Related exercise (incident response — the retrain's canary-failure alert routes to `R3.g` and its rollback aligns with the model-rollback class): `modules/mod-307-ml-reliability-slos/exercise-03-ml-incident-response-drill/SOLUTION.md`.
- Related exercise (offline harness whose slice-level thresholds feed the EVALUATED exit gate): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (shadow and canary deploy design — the mechanics the state machine's SHADOWING and CANARYING states run on): `modules/mod-305-advanced-evaluation/exercise-02-shadow-and-canary-deploy-design/SOLUTION.md`.
- Related exercise (A/B ramp plan — the statistical shape the canary follows at C1 and C2): `modules/mod-306-experimentation-at-scale/exercise-01-power-mde-and-ramp-plan/SOLUTION.md`.
- Related exercise (SRM and trust diagnostics — the SRM gate at each canary step): `modules/mod-306-experimentation-at-scale/exercise-02-srm-and-trust-diagnostics/SOLUTION.md`.
