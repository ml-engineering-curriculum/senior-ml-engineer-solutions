# exercise-01 — Multi-quarter ML Initiative Roadmap (Reference Solution)

Paired exercise: `lessons/mod-310-technical-leadership/exercises/exercise-01-roadmap-and-scoping.md`.

## 1. Solution overview

This exercise produces `roadmap.md` — a two-to-four-quarter roadmap for a named ML initiative that a senior ML engineer owns. The roadmap is the artefact a surface team, a platform partner, a product owner, and a governance reviewer read together to answer four questions:

1. **What is the initiative, and why does it exist now?** The thesis, the measurable outcome, and the decision or incident that forced it onto the calendar.
2. **What is being delivered, and in what order?** The initiative decomposed into milestones, each milestone scoped to a demonstrable slice of value, each milestone's dependencies named.
3. **What are the bets and the risks?** Named assumptions that, if they turn out false, change the plan. Named risks with owners and mitigations.
4. **How does the plan survive contact with reality?** The signals that trigger a re-plan, the decision reviewer, and the cadence at which the roadmap is re-reviewed.

The roadmap is explicitly *not* a Gantt chart, *not* a feature list, and *not* a status report. It is a decision document that frames a multi-quarter investment, scopes it into quarter-sized chunks each of which produces value by itself, and names the mechanics by which the plan is adjusted when quarter 1 ships and the world has changed by quarter 2.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric that calibrates every learner's roadmap to the same bar.

The exercise exists because a senior ML engineer's planning output is a governance artefact that stakeholders outside the engineering team read: a product owner reads it to confirm the initiative delivers the outcome they funded; a platform-team lead reads it to confirm the surface team's roadmap depends on paved roads that will exist when the milestones hit; a governance reviewer reads it to confirm each milestone's risk surface is mapped (NIST AI RMF MAP function) and managed (MANAGE function); an engineering peer reads it to confirm the plan is decomposable, not a single two-quarter cliff. NIST AI RMF GOVERN grounds the roadmap as the governance instrument that makes a multi-quarter investment legible and auditable; MAP grounds the per-milestone risk framing; MEASURE grounds the exit-criteria that make each milestone verifiable; MANAGE grounds the risk register and the re-plan triggers.

## 2. Worked answer / implementation

### 2.1 Roadmap structure

The roadmap has eight required sections. Each section is short, specific, and dated. Aspirational prose ("this initiative will unlock the next generation of ML at the company") is a roadmap smell and is rejected on review.

**§1. Thesis and outcome.**

- **Thesis.** One paragraph. Why this initiative exists, what changes about the world when it is done, and the measurable outcome that defines success. "Improve the support assistant" is not a thesis; "reduce the median support-agent handle time on the three highest-volume queues (billing, shipping, account-mgmt) by 20% by 2027-Q3, measured against the 2026-Q4 baseline, while holding the customer-satisfaction SLI (CSAT) at or above the current p50" is a thesis.
- **Primary outcome metric.** The single number the roadmap is judged on at the end. Pinned to a named dataset / instrumentation path / existing dashboard, so the measurement does not become a Q4 debate. Cross-references the surface's SLIs (see `mod-307/exercise-01`).
- **Secondary outcome metrics.** The counter-metrics and guardrails. A primary-metric improvement that comes at the cost of a secondary metric is not progress; the roadmap must name the guardrails up front so the "tradeoff we quietly made" conversation is already documented.
- **Non-goals.** The things this initiative is *not* chartered to solve. Non-goals are load-bearing — they are what prevent scope creep through the quarter. "This initiative is not chartered to solve cross-queue agent routing; routing remains with team-support-ops" is a non-goal. A roadmap without non-goals is a roadmap that will absorb every adjacent ask.

**§2. Context and trigger.**

- **Trigger.** What forced this roadmap to be written now — a product commitment, an audit finding (see `mod-308/exercise-01`), an incident retrospective, a platform deprecation (see `mod-308/exercise-02`), a regulatory change (see `mod-309`), a competitive pressure, a leadership ask. A roadmap with no trigger date tends to have been written for the wrong reasons.
- **Prior art.** The initiatives and decisions this roadmap builds on. If the roadmap replaces an earlier plan, say so and name what changed about the problem that invalidated the prior plan.
- **Stakeholders.** A named list: surface team(s) delivering, product owner, platform partners (feature store, serving, eval harness, governance), downstream consumer teams if any, governance reviewers if the surface is in scope for `mod-309`. Named by role, not by person, so the roadmap survives rotation.
- **Decision reviewer.** The single person (or small council) who signs off on the quarterly plan and signs off on re-plans mid-quarter. Without one named reviewer, re-plans drift.

**§3. Milestones.**

The decomposition is the heart of the roadmap. Milestones are the mechanism by which a two-to-four-quarter investment is made legible: each milestone ships a demonstrable slice of value by itself, each milestone is scoped to approximately one quarter (not more; shorter is fine), each milestone has explicit exit criteria.

For each milestone:

- **Milestone name and quarter.** `M1 (2027-Q1): offline eval harness with language slices`. A verb-oriented name beats a noun-only name: a reviewer can tell `offline eval harness with language slices` is doing something, `evaluation` is a topic.
- **Scope.** A one-paragraph statement of what is in-scope and, critically, what is out-of-scope for the milestone (to prevent the milestone absorbing the quarter's full ask). The out-of-scope list is derived from §1 non-goals plus any temporary deferrals.
- **Exit criteria.** Measurable, verifiable, dated. "Harness runs nightly; three language slices (en-US, pt-BR, es-MX) each report a point estimate and 95% CI on the four primary SLIs; harness is added to launch-gate config; owner documented in the model card (see `mod-309/exercise-01`); on-call rotation includes the harness." Exit criteria are the contract; a milestone that ships against vague criteria has not shipped.
- **Demonstration.** How the milestone is shown to be done at the end of the quarter — a demo, a dashboard, a Model Card signoff, a loadtest result, a CI-green state. The demonstration belongs in the roadmap because it defines what "shipped" looks like without ambiguity.
- **Dependencies.** Named upstream artefacts, paved-road components, or sister-team deliverables. Each dependency is tagged with its own owner and its own deadline, so the roadmap can be read as a dependency graph.
- **Owner.** The surface-team engineer responsible for landing the milestone. Named by role (`team-agent-assist` tech-lead), not by person.

The first milestone (M1) is special: it should either ship a thin end-to-end slice of the eventual system (preferred — the surface team learns about the integration edges early) or de-risk the highest-uncertainty assumption from §4. M1 is where the roadmap meets reality; nothing after M1 is trustworthy until M1 ships.

**§4. Assumptions and bets.**

A short, numbered list of the claims the roadmap depends on. Each claim is specific enough to be *falsifiable*. If the claim is "the feature store will add prompt-version indexing by 2027-Q2," the roadmap can be checked against the feature-store team's RFC (`mod-308/exercise-03`); if the claim is "AI infrastructure will get better," the roadmap is a wish.

Each assumption carries:

- **Statement.** What is being claimed.
- **Source.** Who or what underwrites it — a team's roadmap, a prior measurement, a vendor contract, a benchmark.
- **Sensitivity.** What happens to the plan if the assumption turns out false. An assumption whose failure invalidates the plan outright is a *bet* and is called out in a `Bets` subsection — bets are the places the surface team is taking risk, and a roadmap with no identified bets is a roadmap that has not been honest about its uncertainty.

**§5. Risks and mitigations.**

The risk register for the initiative. Distinct from §4 in that §4 is about factual claims (the world is X) and §5 is about adverse events (bad thing Y might happen). Named risks are triaged by probability × impact; not all risks need mitigations (some are acceptable), but every risk has an owner and a response plan.

Risks are drawn from the following structured categories, which cross-reference the governance work in `mod-309`:

- **Technical risk.** Capacity, serving, correctness, tail-latency regressions, dependency churn.
- **Delivery risk.** Headcount, cross-team contention, upstream deprecations, platform-team roadmap slippage.
- **Model risk.** Quality regression (tied to `mod-305` evaluation), fairness regression (tied to `mod-309/exercise-03`), robustness regression (tied to `mod-309/exercise-04`).
- **Operational risk.** On-call load, SLO burn-rate budget consumption (tied to `mod-307/exercise-01`), rollback complexity.
- **Governance risk.** Model Card expiry (tied to `mod-309/exercise-01`), new regulatory guidance, incident with governance attribution.
- **Security risk.** Supply chain, adversarial misuse, PII exposure — cross-referenced to the OWASP ML Top 10 taxonomy where the initiative changes ML-specific exposure, and to MITRE ATLAS tactics for generative surfaces.

Each row: `risk description | likelihood (L/M/H) | impact (L/M/H) | owner | mitigation / response plan | trigger that elevates it`.

**§6. Resources and capacity.**

The roadmap is a plan against finite capacity. This section makes capacity explicit so the plan can be read for feasibility:

- **Headcount.** Named FTEs allocated to the initiative by quarter. Partial allocations (e.g., `0.5 FTE for M1`) are declared explicitly.
- **Cross-team asks.** Capacity requested from paved-road teams (e.g., `feature-platform` review of the M1 integration, 3 engineer-days; `governance` review of M2 Model Card, 1 engineer-day). Cross-team asks are the places the plan depends on someone else's calendar; they belong in the roadmap, not in a side-channel ticket.
- **Compute and infrastructure.** Named reservations, quotas, GPU hours, or budget envelopes. A roadmap that assumes unbounded compute is a roadmap that will discover the quota at the worst possible moment.
- **Non-engineering resources.** Annotator budget (for `mod-305` eval set builds), vendor-contract review capacity (if the initiative adds a third-party dependency), legal / compliance review capacity (for `mod-309` surfaces).

**§7. Rollout, measurement, and reversibility.**

For each milestone, how it rolls out and how it is measured:

- **Rollout mechanism.** Named gates: canary (traffic %), per-region progression, cohort opt-in, A/B experiment (see `mod-306`). A milestone that ships "to prod" without a named rollout mechanism is a roadmap that cannot name its blast radius.
- **Measurement window.** The post-launch window during which the primary outcome metric is read, including statistical power considerations (tied to `mod-306/exercise-01`). A "launch, measure next week" plan is a plan with no power.
- **Rollback plan.** The steps to reverse the milestone if the measurement goes wrong. A rollback plan with a single bullet ("we revert the PR") is not a rollback plan if the milestone has downstream dependents.
- **Reversibility of commitments.** Some milestones commit the team irreversibly (a new data retention policy, a published Model Card, a deprecation announcement, a vendor contract). These are flagged `irreversible: yes` with a one-line "why the commitment is accepted at this stage" note.

**§8. Cadence and re-plan triggers.**

The roadmap is a living document. This section names how it is kept alive:

- **Review cadence.** When the roadmap is re-reviewed (minimum: end of each milestone's quarter; typical: monthly check-in, quarterly re-sign, emergency on trigger).
- **Re-plan triggers.** Explicit named signals that force an out-of-cycle re-plan: assumption from §4 turns out false; a risk from §5 materializes; the primary outcome metric from §1 moves materially from a sibling change not anticipated in the plan; a stakeholder from §2 leaves or is re-chartered; a dependency from §3 slips by more than one quarter; a governance review (see `mod-309`) blocks a milestone.
- **Change log.** Dated list of material changes to the roadmap since v1: milestone re-scopes, assumption invalidations, risk escalations, scope reductions, scope additions (added scope must be named, not slipped in).
- **Expiry.** The date by which the roadmap must be re-reviewed unconditionally. A roadmap without an expiry drifts into "it says Q1 but it's actually Q3 now" territory.

### 2.2 Decomposition heuristics

Decomposing a multi-quarter initiative into quarter-sized milestones is the single judgement call that most distinguishes a senior-level roadmap from a junior-level one. The heuristics that work:

- **Each milestone ships a demonstrable slice of value by itself.** A quarter that ends in "we built the foundations for the next quarter" is a quarter that did not ship. The slice may be small (an offline harness that nobody uses in production yet) but it must be *complete*: documented, owned, run on a cadence, with an exit criterion a reviewer can verify.
- **Earliest milestones de-risk the latest.** The first milestone exercises the integration edges, the dependency on the paved road, the eval harness, the Model Card workflow — not the model quality. Model quality is improved on top of a working end-to-end skeleton; a roadmap that plans model quality first and integration last is a roadmap that will not integrate at quarter 4.
- **Dependencies on sister teams become milestone 0, not milestone 3.** If M3 depends on `feature-platform-v2` shipping prompt versions, the roadmap's M0 is *the RFC or alignment that commits `feature-platform-v2` to ship it*. Baking a cross-team dependency into the back half of the roadmap is baking a quarter-sized slip into the plan.
- **No more than one "bet" per milestone.** A milestone that depends on two independent unknowns (the model class is good enough *and* the paved road ships on time) has a probability of success equal to the product of the two, not the max. Separate the bets across milestones so one unknown is resolved before the next is taken.
- **"Measurement" is a verb in the milestone, not a milestone of its own.** Every milestone has exit criteria tied to measurement; the roadmap does not include a milestone called "measure." The measurement window belongs in §7 for that specific milestone.
- **"Documentation" is not a milestone.** A milestone that ships code but not the Model Card / RFC / runbook it requires has not shipped. Documentation is part of the exit criteria of the milestone that produces the thing documented.

### 2.3 Illustrative worked example (not a real system)

> **Illustrative initiative** — a multi-quarter plan to launch `agent-assist/draft`, an internal LLM-backed support-agent assistant, into production on three support queues in three languages, with the governance, evaluation, and monitoring posture a senior ML engineer owns end-to-end. This example is a composite for calibration; do not treat it as a real initiative, team, metric, or incident.

```markdown
# Roadmap: agent-assist-draft, 2027-Q1 through 2027-Q4

## §1. Thesis and outcome
Reduce median support-agent handle time on the three highest-volume queues
(billing, shipping, account-mgmt) by 20% by 2027-Q3, measured against the
2026-Q4 baseline (`dashboard/support-handle-time`, offset -1 quarter),
while holding CSAT p50 at or above the 2026-Q4 value.

- Primary outcome: median handle time (minutes/ticket), measured weekly,
  signed on a 4-week post-launch window.
- Secondary / guardrails: CSAT p50 (not-below guardrail); schema-validity
  rate on drafts (≥ 99%, tied to `mod-307/exercise-01` SLIs); refusal
  rate on safety probes (≥ 97%, tied to `mod-309/exercise-04`); escalation
  rate from draft-assist queues (not-above guardrail).
- Non-goals: agent routing across queues; customer-facing autonomous
  reply; PII extraction model work. Routing remains with team-support-ops;
  autonomous reply is out of scope until EU-AI-Act conformity review
  completes; PII extraction is a separate initiative under `team-data-platform`.

## §2. Context and trigger
Trigger: support-ops Q4 planning identified agent handle time as the
#1 capacity constraint; `agent-assist/draft` was named in the Q4 planning
memo as the mitigation. This roadmap is the engineering response.
Prior art: `assist-v2` (2024) shipped draft replies to 10% of agents; its
retrospective flagged schema-validity drift and no fairness slices. v4
addresses both.
Stakeholders: `team-agent-assist` (delivery); `team-support-ops` (product
owner); `team-data-platform` (training-data, lineage from
`mod-309/exercise-02`); `team-governance` (Model Card, from
`mod-309/exercise-01`); `team-platform-eval` (offline harness from
`mod-305`); `team-security` (threat model from `mod-309/exercise-04`).
Decision reviewer: `team-agent-assist` tech-lead, with monthly cross-team
review chaired by `team-support-ops` PD.

## §3. Milestones

### M1 (2027-Q1): thin end-to-end slice, en-US, billing queue, shadow mode
Scope: one base-model fine-tune for en-US billing-queue drafts; offline
eval harness with en-US slice; Model Card v1 draft; serving behind feature
flag in **shadow mode only** (no agent surface exposure); drift and PSI
monitors (tied to `mod-307`) deployed.
Out of scope for M1: pt-BR, es-MX, non-billing queues, live agent surface,
A/B experiment.
Exit criteria:
- Nightly offline eval with 95% CIs on four primary SLIs on en-US billing
  eval set (N ≥ 2000 per slice).
- Shadow traffic running at 100% of billing-queue tickets with
  schema-validity ≥ 99% over a 2-week window.
- Model Card v1 signed by `team-agent-assist`, `team-support-ops`,
  `team-governance` (three signatures per `mod-309/exercise-01`).
- Runbook for shadow-serving on-call signed by `team-agent-assist`.
Demonstration: Model Card signoff; shadow-eval dashboard review with
`team-support-ops` PD.
Dependencies: `training-ds-v4` (Data Statement from `mod-309/exercise-02`);
`eval-ds-v4-en-US`; `feature-platform` draft-input schema; `model-registry`
entry for `assist-llm-v4`.
Owner: `team-agent-assist` tech-lead.

### M2 (2027-Q2): en-US billing live, A/B, 10% to 100%
Scope: agent-facing draft surface live on billing-queue, en-US only.
A/B experiment (power analysis per `mod-306/exercise-01`); progressive
rollout 10% → 50% → 100% gated on CSAT + schema-validity SLOs.
Out of scope: pt-BR, es-MX, non-billing queues.
Exit criteria:
- A/B experiment hit statistical power on primary outcome (handle time)
  with p < 0.05 at design effect; handle-time median improved by ≥ 10%
  on billing en-US at full ramp.
- All SLOs (schema-validity, refusal rate, latency p95) green for
  4 consecutive weeks at 100%.
- Fairness slice table in Model Card v4.1 refreshed (tied to
  `mod-309/exercise-03` cadence).
- Red-team probe pass (tied to `mod-309/exercise-04`): prompt-injection
  success rate ≤ 5%, refusal rate on safety probes ≥ 97%.
Dependencies: M1 complete; `mod-306/exercise-02` experiment-platform
support for progressive rollout; governance Model Card sign-off scope
extended to include live surface.
Owner: `team-agent-assist` tech-lead.

### M3 (2027-Q3): pt-BR and es-MX added; shipping + account-mgmt queues added
Scope: three-language and three-queue coverage, same gates as M2.
Out of scope: external-facing surface (still out-of-scope per §1 non-goals);
EU-resident ticket coverage (blocked on EU-AI-Act review).
Exit criteria:
- Handle-time median improved by ≥ 20% on the full-language × full-queue
  set at full ramp (this is the §1 primary outcome; M3 is the shipping
  milestone).
- Per-language × per-queue fairness slice table in Model Card v4.3: no
  slice gap > 20% relative on primary quality SLI; any gap > 20% has a
  time-boxed waiver (tied to `mod-309/exercise-01` pattern).
- Guardrail metrics (CSAT, refusal rate) within thresholds across all
  slices for 4 consecutive weeks at full ramp.
Dependencies: M2 complete; `training-ds-v4-pt-BR` and `training-ds-v4-es-MX`
built and signed by `team-data-platform`.
Owner: `team-agent-assist` tech-lead.

### M4 (2027-Q4): quality improvement + sustainment
Scope: address the top two slice gaps from M3, with the biggest expected
being pt-BR acceptance rate (from the exemplar fairness analysis); move
Model Card to v5 cadence (per `mod-309/exercise-01` re-review triggers);
transition on-call from in-team rotation to `team-support-platform` (if
agreed in M2 cross-team review).
Out of scope: next-generation model work (that becomes a new roadmap in
2028-Q1).
Exit criteria:
- Model Card v5 signed; expiry set to 2028-Q2.
- Every slice gap > 20% from M3 either closed or carries a time-boxed
  waiver with a remediation owner.
- On-call handoff memo signed by both teams (if applicable).
Owner: `team-agent-assist` tech-lead.

## §4. Assumptions and bets
A1. The `base-llm-v3` licence permits the planned fine-tunes and
    deployment scope (internal commercial, no redistribution). Source:
    `legal-register/entry-0412`. Sensitivity: HIGH — if invalid, M1 is
    blocked. Mitigation: legal review as M0 (pre-M1).
A2. `feature-platform-v2` ships prompt-version indexing by 2027-Q2.
    Source: `feature-platform` RFC `fp-v2-rfc` (2026-11). Sensitivity:
    MEDIUM — if slipped, M2 can proceed with a shim; but M3 slips by one
    quarter if the shim becomes the long-term plan.
A3. The pt-BR training data expansion lands by 2027-Q2 end. Source:
    `team-data-platform` 2027 H1 plan. Sensitivity: HIGH — if slipped,
    M3 slips by one quarter. **This is a bet.**
A4. The 20% handle-time reduction is attainable at the agent-acceptance
    rates observed in offline eval (0.72 ± 0.02 on en-US billing in the
    M1 shadow-eval). Source: offline eval + agent-side workflow study.
    Sensitivity: HIGH — if offline-to-online gap is > 15 points, the
    initiative misses the primary outcome. **This is a bet.**

## §5. Risks and mitigations (excerpt)
| Risk | L | I | Owner | Mitigation / response | Elevation trigger |
|---|---|---|---|---|---|
| Schema-validity drift post-launch | M | H | tag-agent-assist | Launch gate ≥ 99%; monitor PSI (mod-307); page on burn-rate alerts. | SLO burn > 2% / hour. |
| EU-AI-Act review slips, EU exclusion widens | M | M | team-governance | Keep EU-resident exclusion in Model Card §2 out-of-scope; don't expand scope until review completes. | Review slip > 1 quarter. |
| pt-BR training expansion slips | M | H | team-data-platform | A3 is a bet; mitigation is to keep pt-BR scope time-boxed in M3 and continue en-US + es-MX if pt-BR slips. | A3 blocker reported by `team-data-platform` any monthly check-in. |
| Prompt-injection probe regression | L | H | team-security | Probe set from `mod-309/exercise-04` run on each release; launch-gate at 5%. | Probe success > 5% at any gate. |
| CSAT guardrail regression | L | H | team-support-ops | Secondary guardrail from §1; rollback mechanism per §7. | CSAT p50 < 2026-Q4 at any ramp. |
| Model Card expiry during live surface | L | M | team-governance | Expiry triggers mandatory re-review; cadence from `mod-309/exercise-01`. | 30 days before expiry. |
| On-call load exceeds budget | M | M | team-agent-assist | SLO burn-rate budget from `mod-307/exercise-01`; M4 includes handoff option. | Burn-rate > 50% of quarterly budget. |

## §6. Resources and capacity
- Headcount: 3.0 FTE from `team-agent-assist` for 2027-Q1 through
  2027-Q3; 2.0 FTE for 2027-Q4. Includes one named tech-lead at 0.6
  FTE on roadmap + review, 0.4 on IC work.
- Cross-team asks: `team-data-platform` 3 engineer-weeks across
  M1 and M3; `team-platform-eval` 1 engineer-week in M1 for harness
  on-boarding; `team-governance` 0.5 engineer-week per Model Card
  cycle (4 cycles total); `team-security` 1 engineer-week per red-team
  cycle (2 cycles).
- Compute: reserved GPU envelope for training and offline eval;
  serving compute negotiated with `team-serving-platform` for launch.
- Non-engineering: pt-BR annotator budget approved for 2027-Q2 under
  `team-data-platform`.

## §7. Rollout, measurement, reversibility (per milestone)
- M1: shadow mode only; no rollout; measurement is offline-eval CI-table
  review. Reversibility: feature-flag off is a one-liner; no irreversible
  commitments.
- M2: feature-flag progressive rollout 10% → 50% → 100% on billing en-US;
  A/B at 50% stage with 2-week measurement window; power analysis
  design in `mod-306/exercise-01` style. Rollback: revert ramp to
  previous stage on any SLO breach; incident runbook in-repo. No
  irreversible commitments.
- M3: same mechanism, expanded to three languages × three queues; per-
  slice measurement window 4 weeks. Rollback: per-slice ramp-down.
  Model Card v4.3 is an *irreversible* commitment in that it is published
  to the governance registry; a retract-and-reissue is costly but allowed.
- M4: no new user-facing rollout; measurement is per-slice gap closure.
  On-call handoff, if agreed, is an irreversible commitment (contractual
  between teams); the roadmap flags this and requires both teams' sign.

## §8. Cadence and re-plan triggers
- Review cadence: monthly check-in with `team-support-ops` PD;
  quarterly re-sign at milestone boundary; governance review per Model
  Card cadence (`mod-309/exercise-01`).
- Re-plan triggers: any Sensitivity:HIGH assumption invalidated; any
  §5 risk materializes with Impact:H; primary outcome moves > 5% from
  a sibling change not in the plan; a stakeholder role rotates; a
  dependency slips > 1 quarter; a governance review blocks a milestone.
- Change log: v1 – initial plan, signed 2026-12-15. (Changes recorded
  per revision from here.)
- Expiry: unconditional re-review 2027-Q4 end.
```

### 2.4 Lightweight roadmap linter (static checks)

A small CI script that reads `roadmap.md` and enforces the mechanical discipline below. The linter is not a substitute for a reviewer; it keeps the reviewer's attention on the judgement calls.

```python
# roadmap_lint.py
"""Lint a roadmap for structural completeness.

Enforces: required sections, named milestones with exit criteria,
numbered assumptions, named decision reviewer, named expiry.
"""

import re
import sys


REQUIRED_SECTIONS = [
    "Thesis and outcome", "Context and trigger", "Milestones",
    "Assumptions and bets", "Risks and mitigations",
    "Resources and capacity", "Rollout",
    "Cadence and re-plan triggers",
]

MILESTONE_SUBPARTS = ["Scope", "Exit criteria", "Dependencies", "Owner"]


def lint(text: str) -> list[str]:
    errors: list[str] = []
    lowered = text.lower()
    for s in REQUIRED_SECTIONS:
        if s.lower() not in lowered:
            errors.append(f"missing section: {s}")
    if not re.search(r"primary outcome", lowered):
        errors.append("no named primary outcome metric")
    if not re.search(r"non.?goal", lowered):
        errors.append("no non-goals declared")
    if not re.search(r"decision reviewer|reviewer:", lowered):
        errors.append("no decision reviewer named")
    milestones = re.findall(r"^#+\s*M\d+", text, re.M)
    if len(milestones) < 2:
        errors.append(f"expected ≥ 2 milestones (found {len(milestones)})")
    for sub in MILESTONE_SUBPARTS:
        if sub.lower() not in lowered:
            errors.append(f"no milestone subpart found: {sub}")
    if not re.search(r"\bA\d+\.", text):
        errors.append("no numbered assumptions (A1., A2., ...)")
    if "bet" not in lowered:
        errors.append("no bets declared; a roadmap with no bets is "
                      "either trivial or not honest about uncertainty")
    if not re.search(r"re-?plan trigger", lowered):
        errors.append("no re-plan triggers named")
    if not re.search(r"expir", lowered):
        errors.append("no expiry date / unconditional re-review")
    if not re.search(r"nist ai rmf|govern|map|measure|manage", lowered):
        errors.append("NIST AI RMF not cited (GOVERN/MAP/MEASURE/MANAGE)")
    return errors


if __name__ == "__main__":
    text = open(sys.argv[1]).read()
    errs = lint(text)
    for e in errs:
        print(f"roadmap-lint error: {e}")
    sys.exit(1 if errs else 0)
```

## 3. Validation steps

Walk the roadmap through this checklist before sign-off:

1. **§1 names a measurable primary outcome** tied to an existing dashboard or SLI. "Improve X" is not an outcome; "move SLI Y by Z by date D" is.
2. **§1 names non-goals.** The roadmap's defence against scope creep is the non-goals list; missing it is a roadmap that will absorb every adjacent ask.
3. **§2 names the trigger.** Roadmaps without a trigger date have been written for the wrong reasons.
4. **§2 names stakeholders by role and a single decision reviewer.** No reviewer → no sign-off → no re-plan discipline.
5. **§3 milestones each have scope, exit criteria, demonstration, dependencies, and owner.** Missing any of the five is an incomplete milestone.
6. **M1 is a thin end-to-end slice** or de-risks the highest-uncertainty assumption. M1 that only "lays foundations" is a quarter that did not ship.
7. **Every milestone has an exit criterion a reviewer can verify** (dated, measured, with a reference dataset or SLI).
8. **No milestone has more than one bet.** Combined bets inflate failure probability.
9. **§4 assumptions are numbered, sourced, and have a sensitivity label.** Bets are called out explicitly.
10. **§5 risks have likelihood, impact, owner, mitigation, and an elevation trigger.** Rows missing any are not risk entries.
11. **§6 names headcount in FTEs by quarter, cross-team asks in engineer-weeks, and compute envelopes.** Un-scoped capacity is a plan without constraints.
12. **§7 names a rollout mechanism and a rollback for every milestone that ships to users.** Irreversible commitments are flagged.
13. **§8 names a review cadence, re-plan triggers, change log, and expiry.** Without §8, the roadmap decays silently.
14. **NIST AI RMF (GOVERN / MAP / MEASURE / MANAGE) is cited** against the sections it grounds. GOVERN on §1 / §2 / §8; MAP on §4 / §5; MEASURE on §1 / §3 exit criteria / §7 measurement; MANAGE on §5 / §7.
15. **The roadmap cites cross-module artefacts** where it depends on them: Model Card (`mod-309/exercise-01`), lineage (`mod-309/exercise-02`), fairness cadence (`mod-309/exercise-03`), threat model (`mod-309/exercise-04`), eval harness (`mod-305/exercise-01`), experiment design (`mod-306/exercise-01`), SLOs (`mod-307/exercise-01`), platform audit / build-vs-adopt (`mod-308/exercises-01-02`).

Static checks a CI step can enforce:

```bash
grep -ciE "primary outcome|non.?goal|decision reviewer" roadmap.md
grep -ciE "exit criteria|dependencies|owner"            roadmap.md
grep -ciE "assumption|bet|sensitivity"                  roadmap.md
grep -ciE "risk|likelihood|impact|elevation"            roadmap.md
grep -ciE "expir|cadence|re-?plan trigger"              roadmap.md
grep -ciE "nist ai rmf|govern|map|measure|manage"       roadmap.md
```

`roadmap_lint.py` from §2.4 gives a programmatic version of the same checks.

## 4. Rubric / review checklist

- [ ] §1 primary outcome is a measurable number tied to a named SLI or dashboard.
- [ ] §1 secondary / guardrail metrics are named and tied to existing SLIs (`mod-307`).
- [ ] §1 non-goals are specific prohibitions, not aspirations.
- [ ] §2 trigger is dated and traceable to an incident, audit, memo, or launch commitment.
- [ ] §2 stakeholders are named by role and the single decision reviewer is named.
- [ ] §3 has at least two milestones, each with scope, exit criteria (measurable, dated), demonstration, dependencies, and a named owner.
- [ ] M1 is a thin end-to-end slice OR explicitly de-risks the highest-uncertainty §4 assumption.
- [ ] No milestone contains more than one bet.
- [ ] §4 assumptions are numbered; source and sensitivity are named; bets are called out explicitly.
- [ ] §5 risk rows contain likelihood, impact, owner, mitigation, and elevation trigger.
- [ ] §5 risks span the six categories (technical, delivery, model, operational, governance, security) with references to `mod-309` and OWASP ML Top 10 / MITRE ATLAS where ML-specific exposure applies.
- [ ] §6 names FTE allocations per quarter, cross-team asks in engineer-weeks, compute / infrastructure envelopes, and non-engineering resource budgets.
- [ ] §7 names rollout mechanism (canary / A/B / progressive) and rollback per milestone; irreversible commitments flagged.
- [ ] §7 measurement windows reference `mod-306/exercise-01` power considerations.
- [ ] §8 names review cadence, re-plan triggers, dated change log, and expiry.
- [ ] NIST AI RMF (GOVERN / MAP / MEASURE / MANAGE) is cited against the relevant sections.
- [ ] OWASP ML Top 10 is cited where the initiative changes ML-specific threat exposure; MITRE ATLAS is cited for generative surfaces.
- [ ] Cross-module references are present: `mod-305`, `mod-306`, `mod-307`, `mod-308`, `mod-309` as applicable.

Review-weighted points for grading:

- §1 thesis and outcome clarity (specific SLI, non-goals): 15%
- §3 milestone decomposition (end-to-end slice, exit criteria, one-bet-per-milestone): 25%
- §4 assumptions / bets honesty: 10%
- §5 risk register coverage and specificity: 15%
- §6 resource realism (FTEs, cross-team asks, compute): 10%
- §7 rollout and reversibility rigor: 10%
- §8 cadence and re-plan trigger discipline: 10%
- Cross-module and standards references (NIST AI RMF, OWASP, ATLAS, related solutions): 5%

## 5. Common mistakes

- **Thesis as marketing.** "Unlock the next generation of ML-powered support." Not measurable, not dated, not falsifiable. A thesis is a number by a date against a baseline.
- **No non-goals.** A roadmap without non-goals is a roadmap that will discover its scope mid-quarter.
- **Milestones without exit criteria.** "M2: ship to prod" is not a milestone. "M2: live on billing en-US at 100% with four SLIs green for four consecutive weeks" is.
- **M1 as "build the foundations."** A quarter that ends in "we built the plumbing but have not shipped anything" is a quarter that did not ship. M1 must produce something a reviewer can see.
- **Combined bets in one milestone.** A milestone that depends on two independent unknowns multiplies the failure rate.
- **Assumptions without sensitivity labels.** An un-labelled assumption is a mine; the roadmap will step on it when the assumption turns out false and nobody will remember it was a known.
- **Risk register as paragraphs.** Risks without likelihood / impact / owner / mitigation / trigger are prose, not triage. The register is a table, not an essay.
- **Capacity implied, not declared.** "We'll resource this appropriately" is not a resource plan. Named FTEs by quarter are.
- **No rollback plan.** A milestone that ships without a named rollback is a milestone that will not roll back well under pressure.
- **Irreversible commitments not flagged.** Model Card publications, deprecation announcements, vendor contracts, cross-team on-call handoffs — all irreversible on a quarterly horizon. Each must be labelled, and the roadmap must say why the irreversibility is acceptable at the chosen moment.
- **No re-plan triggers.** A roadmap with no named triggers is a roadmap whose Q3 version is the Q1 version with the dates shifted — a lie.
- **No expiry.** A roadmap with no expiry is a document the team will forget to re-visit until someone asks "wait, is this still the plan?"
- **No cadence to the governance artefacts.** Model Card expiry (`mod-309/exercise-01`), fairness slice cadence (`mod-309/exercise-03`), and SLO-burn alarms (`mod-307/exercise-01`) all run on their own cadences; the roadmap must align its milestone boundaries with these or acknowledge the misalignment explicitly.
- **Cross-team dependencies in the back half.** If M3 depends on the paved road shipping a feature, M0 is the RFC committing the paved road to ship it (`mod-308/exercise-03`). Baking the dependency into M3 without a preceding alignment milestone bakes a slip into the plan.
- **"We're a strong team" as the capacity argument.** Capacity is quantified. A roadmap that leaves capacity vague will be over-allocated silently.
- **No distinction between §4 assumptions and §5 risks.** Assumptions are factual claims; risks are adverse events. Mixing them loses the ability to say "assumption falsified → re-plan" versus "risk materialized → execute mitigation."

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds the roadmap as the governance instrument that makes multi-quarter investment legible, auditable, and re-reviewable (§1 / §2 / §8). MAP function grounds the assumption framing and risk identification (§4 / §5). MEASURE function grounds the per-milestone exit criteria and the primary-outcome measurement window (§1 / §3 / §7). MANAGE function grounds the risk register, mitigations, rollout, rollback, and re-plan triggers (§5 / §7 / §8).
- NIST AI 600-1 (Generative AI Profile) — <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>. For generative surfaces, each milestone's risk register references the GAI risk taxonomy (CBRN info, confabulation, data privacy, harmful bias, human-AI configuration, information integrity, information security, intellectual property, obscenity/degradation, value chain); the Model Card re-review cadence driven by this taxonomy feeds §8 re-plan triggers.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Used as the ML-specific threat taxonomy for §5 security risk rows on classical-ML surfaces (input manipulation, data poisoning, model inversion, membership inference, model theft, supply-chain, transfer-learning risks, model skewing, output integrity, denial of ML service).
- MITRE ATLAS — <https://atlas.mitre.org/>. Used as the adversarial-tactic taxonomy against which §5 security rows are framed for generative surfaces (e.g., AML.T0051 LLM Prompt Injection, AML.T0043 Craft Adversarial Data, AML.T0024 Exfiltration via ML Inference API). Threat-model artefact reference: `mod-309/exercise-04`.

Practitioner implementation examples:

- VeriSwarm engineering blog — <https://veriswarm.ai>. Useful only as a shape-of-artefact reference for team-level roadmap write-ups; not a standard.

Local context (authoritative for the artefact shape and cross-references):

- Paired lesson: `lessons/mod-310-technical-leadership/exercises/exercise-01-roadmap-and-scoping.md`.
- Module README: `modules/mod-310-technical-leadership/README.md`.
- Sibling exercise (code review / design review cadence this roadmap's milestones go through): `modules/mod-310-technical-leadership/exercise-02-code-review-and-design-review-at-bar/SOLUTION.md`.
- Sibling exercise (mentorship plan that develops the engineers who deliver these milestones): `modules/mod-310-technical-leadership/exercise-03-mentorship-plan-for-a-mid-level/SOLUTION.md`.
- Sibling exercise (interview loop that staffs the team delivering the roadmap): `modules/mod-310-technical-leadership/exercise-04-rubric-based-ml-interview-loop/SOLUTION.md`.
- Related exercise (evaluation harness feeding each milestone's exit criteria): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (experiment design for each milestone's measurement window): `modules/mod-306-experimentation-at-scale/exercise-01-power-mde-and-ramp-plan/SOLUTION.md`.
- Related exercise (SLO document whose SLIs the §1 outcome and §5 operational-risk rows reference): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (paved-road audit that may be the §2 trigger): `modules/mod-308-platform-collaboration/exercise-01-paved-road-consumption-audit/SOLUTION.md`.
- Related exercise (build-vs-adopt decision whose outcome §3 milestones depend on): `modules/mod-308-platform-collaboration/exercise-02-build-vs-adopt-tradeoff-doc/SOLUTION.md`.
- Related exercise (contribute-back RFC that unblocks a cross-team dependency): `modules/mod-308-platform-collaboration/exercise-03-contribute-back-rfc-to-platform/SOLUTION.md`.
- Related exercises (governance artefacts whose cadences anchor §8 re-plan triggers): `modules/mod-309-responsible-ai-governance/` (all four exercises).
