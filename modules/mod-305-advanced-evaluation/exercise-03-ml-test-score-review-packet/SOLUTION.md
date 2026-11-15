# exercise-03 — ML Test Score Review Packet (Reference Solution)

Paired exercise: `lessons/mod-305-advanced-evaluation/exercises/exercise-03-ml-test-score-review-packet.md`.

## 1. Solution overview

This exercise produces `ml-test-score-packet.md` — a structured review packet that scores one production ML system against the Breck et al. "ML Test Score" rubric, surfaces the weakest category, and names the two or three gap-closing actions that would most raise the system's production-readiness. The artefact has six parts:

1. A **system frame** that pins down what is being scored: the model, the serving surface, the owning team, and the review scope (one model, one pipeline, one release).
2. A **scored rubric** — every test item in the four ML Test Score categories, each with a score, evidence (link, query id, or doc reference), and a one-line "why this score."
3. A **category summary** — per-category totals, the overall score (the minimum across categories — by design, the weakest category bounds production readiness), and the interpretation band.
4. A **gap register** — the lowest-scoring items that most constrain the overall score, ordered, with proposed remediations.
5. A **remediation plan** — for the top two or three gaps: concrete work items, owners, target dates, and the follow-up re-score schedule.
6. A **sign-off block** — the named role(s) that approve the packet, the date, and the next-review cadence.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric.

The exercise exists because production readiness is a floor, not an average. A system that monitors beautifully but never versions its training data is not "mostly ready"; it is unready in a specific way, and the ML Test Score rubric by Breck et al. is the standard shape teams use to make that unreadiness legible. The NIST AI RMF MEASURE and GOVERN functions call for exactly this kind of documented, repeatable assessment; this packet is a team's recurring GOVERN artefact.

## 2. Worked answer / implementation

### 2.1 System frame

Six lines:

- **System under review.** Model name, version, task, and the serving surface. "The recommendation service" is not a system; "the `reco-v7` ranker behind the homepage carousel endpoint" is.
- **Review scope.** One model and its pipeline; cross-model or cross-team reviews are packaged separately. Scope creep is why these reviews stall.
- **Owning team / role.** The team responsible for the fixes, not the one that commissioned the review.
- **Related artefacts.** Links to the offline harness (exercise-01), the rollout plan (exercise-02), and the monitoring dashboards. The packet depends on these being in place; if they are not, that is a finding, not a blocker.
- **Review purpose.** What decision this packet feeds — annual readiness review, pre-major-release gate, post-incident audit, platform migration readiness. Different purposes weight different categories.
- **Review date / last reviewed.** A date. The packet is not timeless; a six-month-old packet is advisory, not current.

### 2.2 Scored rubric

The rubric covers four categories defined in Breck et al. (2017), "The ML Test Score: A Rubric for ML Production Readiness and Technical Debt Reduction" (IEEE Big Data 2017). Categories:

1. **Features and data.** Tests covering feature expectations, data schemas, dataset provenance, feature-value distribution, feature cost, policy/compliance on data.
2. **Model development.** Tests covering model specification review, offline metrics, hyperparameter sensitivity, impact of staleness, baseline comparison, fairness and inclusion, and reproducibility of training.
3. **ML infrastructure.** Tests covering reproducible training, integration tests across the pipeline, model-quality regression tests, canary serving infrastructure, and disaster-recovery / rollback-of-serving capability.
4. **Monitoring.** Tests covering dependency freshness, data-invariant checks in serving, training/serving skew detection, prediction-distribution monitoring, resource-usage monitoring, model-quality degradation monitoring, and alerting on each.

**Scoring per item (Breck et al. scheme):**

- `0` — the test is not performed at all.
- `0.5` — the test is performed manually or only partially automated.
- `1` — the test is fully automated and runs on every relevant change or on a declared cadence.

**Per-item evidence.** Every non-zero score names the evidence: a link to the test, the query id, the dashboard panel, the runbook, or the documented procedure. A score without evidence is a `0` with wishful thinking.

**Per-item one-line "why this score."** For borderline items, this is the audit trail of the decision. Blind pattern-matching to prior packets is how scores drift up without the underlying system changing.

Senior-engineer notes on scoring:

- **Default to the lower score on ambiguity.** If the team cannot point at the evidence in under two minutes, it is `0` or `0.5`, not `1`. A test that nobody can find is not running.
- **Partial automation is `0.5`, not `1`.** A monitoring query that fires an email that a human reads on Monday mornings is `0.5`. The test must be *actionable automation* — a pager, an auto-rollback, a blocked merge — to earn `1`.
- **Shared tests count once.** If a team's data-quality suite is shared across ten models, each model inherits the score, but a finding in the suite is a finding for all ten. Flag the shared dependency in the evidence line.
- **Legacy "we used to do that" is `0`.** A test that has not run in six months is not a test.

### 2.3 Category summary and overall score

Per the Breck et al. rubric, the overall score is the **minimum** of the four category totals. The rationale: a well-monitored system with no data-quality tests is one bad upstream schema change from catastrophic silent failure; the weakest category bounds the risk posture. A mean would hide that.

The rubric interpretation bands (from Breck et al. 2017) describe how readiness grows as the score grows — roughly:

- A score at or near `0` describes a system that is not production-tested; shipping it to real users is an incident waiting for a trigger.
- Low positive scores describe a system with basic sanity checks; failure modes are likely to be noticed, not prevented.
- Mid-range scores describe reasonable production testing with gaps that are known and tracked.
- High scores describe a system with strong automated testing across all four categories.

The team's packet reports:

- **Per-category totals** with the scored items.
- **Overall score** as the minimum category total.
- **Band interpretation** for the team's own band (drawn from the Breck et al. paper), with the understanding that bands are rubric-indicative, not pass/fail gates.

### 2.4 Gap register

Rank the lowest-scoring items, then within a score bucket rank by *how much raising the item would raise the overall score*. Because the overall score is the category minimum, items in the weakest category dominate.

Each gap entry contains:

- **Item.** The specific rubric test item.
- **Current score.** `0` or `0.5`.
- **Why it is this score.** One sentence from the rubric evidence column.
- **What `1` looks like.** A specific, testable description. "Automated schema check on training data runs on every merge, blocks merge on failure, is reviewed on deprecation."
- **Lift to overall.** How many points the overall score moves if this item moves to `1`. (Often zero, if the weakest category has other low items; often the whole lift, if this is the floor-setter.)
- **Cost.** T-shirt size (S / M / L) and dependency notes.

The gap register is ordered first by *lift to overall*, then by *lowest cost*. The team's remediation plan takes the top two or three.

### 2.5 Remediation plan

For each remediation item:

- **The gap.** Referenced from §2.4.
- **The work.** Concrete, testable description of what shipping looks like. "Add Great Expectations suite to the feature pipeline covering schema and null-rate expectations for the top-20 features; wire into the DAG so that a validation failure fails the DAG."
- **The owner.** Named role (not a person), for durability.
- **The target date.** Specific date.
- **The re-score plan.** Which item is re-scored, by whom, on what evidence. A remediation without a re-score is a promise.

The plan does not try to close every gap. Closing two gaps a quarter beats listing twelve that never move. The rest stay in the register.

### 2.6 Sign-off

A short block:

- **Reviewed by.** Named role(s).
- **Reviewed on.** Date.
- **Next review cadence.** Quarterly, pre-release, or incident-triggered.
- **Next review scheduled.** Date.

Without the sign-off, the packet is advisory; with it, it is accountable.

### 2.7 Illustrative worked example (not a real system)

> **Illustrative system** — `router-v12` (from exercise-01 / exercise-02), a tabular classifier that auto-routes support tickets. This example is a composite for calibration; do not treat it as a real system.

System frame:

- System under review: `router-v12`, binary classifier, ticket-intake surface.
- Scope: this model and its pipeline.
- Owning role: `routing-ml-team`.
- Related artefacts: offline harness `harness-v4`, rollout plan `online-rollout-router-v12`, monitoring dashboard `canary-health`.
- Review purpose: annual readiness review.
- Review date: 2026-11-10. Last reviewed: 2026-11-10.

Scored rubric (abbreviated — a real packet lists every test item from Breck et al. 2017; sample entries only):

| Category | Item | Score | Evidence | Why |
|---|---|---|---|---|
| Features & data | Feature schemas exist, versioned, and checked at ingest. | 1 | `features/schemas/*.json`, DAG task `schema_check` | Fully automated; run on every DAG tick. |
| Features & data | Training data lineage traceable back to source. | 0.5 | manual notes in training README | Not machine-verifiable; relies on contributor memory. |
| Features & data | Features reviewed for policy / privacy. | 0 | — | No documented review. |
| Model dev | Offline metrics include slice reporting. | 1 | exercise-01 harness | Hard-gated slices in harness. |
| Model dev | Fairness and inclusion tests. | 0.5 | manual audit in 2026-Q2 | Not automated; not scheduled. |
| Model dev | Training is reproducible from a pinned commit + pinned data snapshot. | 0.5 | `scripts/train.sh` + `data_version.txt` | Pinned data path, but data mutation outside source control not detected. |
| Infra | Canary serving infra exists and is used. | 1 | exercise-02 ramp controller | Canary ramp with auto-rollback live. |
| Infra | Rollback of serving is a single action with < 5 min budget. | 1 | `router.candidate_enabled` flag | Flag flip revert within 60 s. |
| Infra | Model quality regression test runs before each deploy. | 1 | exercise-01 harness in CI | `pass` required pre-merge. |
| Monitoring | Training-serving skew monitored. | 0.5 | nightly batch comparison job | Not alerting; dashboard-only. |
| Monitoring | Prediction distribution monitored. | 1 | `qid:pred-dist` alert | Alerts page the on-call. |
| Monitoring | Model-quality degradation monitored in production. | 0 | — | No live quality metric wired in; relies on offline re-eval. |

Category totals (illustrative, not a full enumeration):

- Features & data: partial (zeros present on policy review and partials on lineage).
- Model development: partial (zeros absent but two partials).
- ML infrastructure: strong (all three sample items at 1).
- Monitoring: partial-to-weak (one zero on quality degradation, one partial on skew).

Overall score: the minimum category total. In this abbreviated example, the weakest category is Monitoring, driven primarily by "no live quality-degradation signal." The overall score sits in the band whose interpretation is "basic production testing exists, with known gaps that need remediation."

Gap register (top entries):

| Item | Current | What `1` looks like | Lift | Cost |
|---|---|---|---|---|
| Monitoring: model-quality degradation in production | 0 | Live quality proxy (e.g., agent-accept rate) streamed to a dashboard with an alert rule; weekly offline eval against the frozen golden set with a page on regression. | High (moves the weakest category; raises the overall score). | M |
| Monitoring: training-serving skew | 0.5 → 1 | Nightly skew job emits structured results; alert rule on deviation > threshold pages ML on-call. | Medium. | S |
| Features & data: policy / privacy review of features | 0 → 0.5 | Documented review checklist executed and signed off by named role per feature release; automation is a next step. | Medium (raises Features & data toward the next band). | S |

Remediation plan (top two):

1. **Wire a live model-quality signal.** Agent-accept rate streamed hourly to the `canary-health` dashboard, with an alert rule on regression below baseline band. Owner: `routing-ml-on-call`. Target: 2026-12-15. Re-score: `monitoring: model-quality degradation` moves from `0` to `1`; re-scored by the ML platform reviewer against the dashboard and the alert rule.
2. **Automate training-serving skew alerting.** Wire the nightly skew job to the alerting pipeline, with a paged alert on deviation. Owner: `routing-ml-on-call`. Target: 2026-12-01. Re-score: `monitoring: training-serving skew` moves from `0.5` to `1`.

Sign-off:

- Reviewed by: `ml-platform-review-chair`, `routing-ml-on-call`.
- Reviewed on: 2026-11-10.
- Next review cadence: quarterly, with an incident-triggered review if a page-worthy regression occurs.
- Next review scheduled: 2027-02-10.

### 2.8 Runnable sketch: a packet validator

The packet is YAML (or JSON) alongside the Markdown. A validator keeps the two in sync and keeps evidence-free `1`s from sneaking in:

```python
# tools/validate_packet.py
from dataclasses import dataclass
from typing import Literal

Score = Literal[0, 0.5, 1]
Category = Literal["features_data", "model_development", "ml_infrastructure", "monitoring"]

@dataclass
class ItemScore:
    category: Category
    item: str
    score: Score
    evidence: str | None
    rationale: str

def validate(items: list[ItemScore]) -> list[str]:
    errors: list[str] = []
    for i in items:
        if i.score > 0 and not i.evidence:
            errors.append(f"{i.category}:{i.item}: score {i.score} has no evidence")
        if i.score == 1 and "manual" in (i.rationale or "").lower():
            errors.append(f"{i.category}:{i.item}: rationale mentions 'manual' but score is 1")
        if not i.rationale:
            errors.append(f"{i.category}:{i.item}: missing rationale")
    return errors

def category_totals(items: list[ItemScore]) -> dict[Category, float]:
    totals: dict[Category, float] = {c: 0.0 for c in ("features_data", "model_development", "ml_infrastructure", "monitoring")}
    for i in items:
        totals[i.category] += float(i.score)
    return totals

def overall(totals: dict[Category, float]) -> float:
    return min(totals.values())  # rubric reads overall = min(category)
```

The validator exists so the packet does not quietly rot. A packet that cannot pass its own validator is not a scored rubric; it is a wish list.

## 3. Validation steps

Walk the artefact through this checklist:

1. **System frame is scoped to one model / one pipeline.** Cross-model packets are split.
2. **Every item has a score, evidence (if non-zero), and a rationale.** The validator passes.
3. **"Automated" means pager / block / rollback, not email.** Scores of `1` cite the actionable automation.
4. **Category totals are reported; overall score is the minimum.** The team does not silently use a mean.
5. **The band interpretation is quoted from the Breck et al. paper or marked `open-item:` when uncertain.** Do not invent band labels.
6. **Gap register is ordered by lift-to-overall, then cost.** A gap list that is not ordered is a wish.
7. **Remediation plan targets two or three items with named owners, dates, and re-score plans.** Not twelve.
8. **Sign-off block has a named role, a date, and a scheduled next review.**
9. **Related artefacts (harness, rollout plan, dashboards) exist.** If they do not, the packet records that as a finding.
10. **The packet is dated and in version control.**

Static checks:

```bash
grep -c "^| " ml-test-score-packet.md                     # scored items present
grep -c "open-item:" ml-test-score-packet.md              # mark uncertainty; must be addressed before auto-merge
grep -ci "overall score\|minimum\|min(" ml-test-score-packet.md   # min-across-categories rule referenced
grep -ci "last reviewed\|reviewed on" ml-test-score-packet.md     # must exist
grep -ci "sign-off\|signed off" ml-test-score-packet.md           # named role
```

The packet validator (§2.8) is the dynamic check.

## 4. Rubric / review checklist

- [ ] System frame is one model / one pipeline; scope is stated; owning role named.
- [ ] Rubric covers all four Breck et al. categories: Features & data, Model development, ML infrastructure, Monitoring.
- [ ] Every item has a score (`0`, `0.5`, or `1`) and a rationale.
- [ ] Every non-zero score names concrete evidence (link, query id, dashboard panel, runbook).
- [ ] `1` is reserved for actionable automation (pager, block, auto-rollback), not dashboards or emails.
- [ ] Per-category totals reported.
- [ ] Overall score reported as the minimum across categories (not the mean).
- [ ] Band interpretation is drawn from the published paper; exact numeric bucket labels are quoted or marked with `open-item:` where unverified locally.
- [ ] Gap register ordered by lift-to-overall then cost, with "what `1` looks like" for each top gap.
- [ ] Remediation plan limited to two or three items, each with owner (role), date, and re-score plan.
- [ ] Sign-off block present: roles, date, next review scheduled.
- [ ] Packet depends on exercise-01 (offline harness) and exercise-02 (rollout plan); absence of either is a finding in the relevant category, not a reason to skip the packet.
- [ ] NIST AI RMF GOVERN / MEASURE referenced where the packet grounds the team's governance cadence.
- [ ] `open-item:` comments addressed before the packet is treated as final.

Review-weighted points:

- System frame and scope discipline: 10%
- Scoring discipline (evidence + rationale per item, `1` = automation not dashboard): 25%
- Category totals + overall-as-minimum computation + band interpretation: 20%
- Gap register (ordered by lift-to-overall then cost; "what `1` looks like" present): 15%
- Remediation plan (two-to-three concrete items with owner, date, re-score plan): 15%
- Sign-off, review cadence, standards alignment (Breck et al. rubric cited; NIST AI RMF GOVERN tie-in): 15%

## 5. Common mistakes

- **Averaging the score.** A mean hides the weakest category, which is the one that sinks the system. The overall score is the minimum.
- **Scoring from memory.** Every non-zero score has evidence. A `1` that nobody can locate in two minutes is a `0`.
- **"Automated" without alerts.** A nightly dashboard panel is partial automation; without a page or a block or a rollback, the test is `0.5`.
- **Policy / privacy review as a single sentence.** The Features & data category includes policy review. "We looked at it" is `0`.
- **Fairness / inclusion scored by whether tests exist, not whether they are run on cadence.** A once-a-year manual audit is `0.5`, not `1`.
- **Twelve-point remediation plans.** The plan targets two or three items. More is theatre.
- **Remediation without re-score.** A gap-closure is only closed when the item re-scores on new evidence.
- **Scoring up to look good.** A packet whose scores drift up without the underlying system changing is a governance failure, not a success. The validator (§2.8) catches evidence-free `1`s.
- **Skipping the packet because the harness is still being built.** If exercise-01 or exercise-02 artefacts are missing, that is a finding in the Model development or ML infrastructure category, not a reason to defer.
- **Treating the packet as a one-shot.** The packet is a recurring artefact. Six-month-old packets are advisory; the sign-off block schedules the next review.
- **Inventing band labels.** The Breck et al. paper uses specific interpretation bands; quote them from the paper or mark the claim with `open-item:`. Do not paraphrase into a new rubric.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. The GOVERN function underpins the recurring sign-off cadence; MEASURE underpins the evidence requirement on every non-zero score.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Referenced in the Features & data policy / privacy items and in the Monitoring category for adversarial-input monitoring.
- MITRE ATLAS — <https://atlas.mitre.org/>. Referenced in the Monitoring category for threat-aligned production monitoring (evasion, model-extraction detectors) where the system's threat model includes those tactics.

Published rubric this packet implements:

- Breck, Cai, Nielsen, Salib, Sculley, "The ML Test Score: A Rubric for ML Production Readiness and Technical Debt Reduction," IEEE Big Data 2017. The four categories (Features & data, Model development, ML infrastructure, Monitoring), the per-item scoring (`0` / `0.5` / `1`), and the overall-as-minimum-of-categories rule come from this paper. Exact numeric band labels used in any specific team's packet should be quoted from the paper; this solution marks band labels with `open-item:` where it does not quote specifics.

Local context (authoritative for the packet taxonomy and the review ordering):

- Paired lesson: `lessons/mod-305-advanced-evaluation/exercises/exercise-03-ml-test-score-review-packet.md`.
- Module README: `modules/mod-305-advanced-evaluation/README.md`.
- Related exercise (offline harness that feeds Model development scores): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (rollout plan that feeds ML infrastructure scores): `modules/mod-305-advanced-evaluation/exercise-02-shadow-and-canary-deploy-design/SOLUTION.md`.
- Related exercise (LLM-as-judge, when the Monitoring category includes an LLM-based quality signal): `modules/mod-305-advanced-evaluation/exercise-04-llm-as-judge-when-and-how/SOLUTION.md`.
