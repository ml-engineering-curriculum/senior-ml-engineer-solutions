# exercise-01 — Paved-Road Consumption Audit (Reference Solution)

Paired exercise: `lessons/mod-308-platform-collaboration/exercises/exercise-01-paved-road-consumption-audit.md`.

## 1. Solution overview

This exercise produces `paved-road-audit.md` — a written inventory of the internal ML platforms, paved roads, and shared services the team's production ML surface touches, classified row-by-row by *consumption posture*, and ending in a prioritized action list. The artefact has six parts:

1. A **system under audit** section that pins down which surface is being audited, which team operates it, and which paved roads are in scope.
2. A **paved-road inventory** — the list of platforms and shared services that *exist* for this class of system (feature store, model registry, CI/CD for ML, training platform, serving platform, evaluation harness, feature flags, observability, data quality, governance review, incident tooling, cost / quota, safety review). The inventory is the universe the audit is measured against; omitting a platform from the inventory is the first failure mode.
3. A **consumption posture table** — one row per paved road, with a classification (`adopted`, `partial`, `bypassed`, `forked`, `absent`), the specific artefact / contract on the paved-road side, the team's current consumption of it, the *reason* for any deviation, and the *cost of the deviation* in operational, cognitive, and risk terms.
4. A **drift and debt panel** — for each `partial / bypassed / forked` row, a dated note on when the deviation began, who authorized it (if anyone), and whether the reason it was taken is still true.
5. An **action list** — the prioritized set of moves out of the audit: adopt the paved road, contribute back what is missing, retire a local fork, keep the deviation under a named-role waiver, or escalate the absence of a paved road.
6. A **review cadence** — when the audit is re-run, by whom, and which triggers (new platform, deprecation, incident, re-org) force an out-of-cycle re-run.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric that calibrates every learner's audit to the same bar.

The exercise exists because a senior ML engineer's first job on a cross-team surface is to *know what their system consumes*. Shadow infrastructure — a hand-rolled feature job that duplicates the feature store, a notebook training loop that bypasses the training platform, a local dashboard that reimplements the central observability tool — is where incidents hide and where platform teams discover the real roadmap they should have been building. An audit is the artefact that makes the shadow visible and names who owes whom what. NIST AI RMF GOVERN grounds the audit as the governance instrument that keeps an ML surface's platform dependencies legible to leadership and reviewers; MANAGE grounds the action list as risk control on those dependencies.

## 2. Worked answer / implementation

### 2.1 System under audit

Six lines, in this order, so a reviewer can situate the audit before reading the table:

- **Surface audited.** The user-facing endpoint, model version, and primary data shape the audit covers. Pinned to a name. "Our recommender" is not a surface; "`reco-serve` producing per-user item lists for the homepage feed of logged-in US web users, model `reco-v12`, served by team `team-reco`" is.
- **Audit owner.** The engineer running the audit and the role they hold (typically the L30 senior on the ranker pod). The owner is the one who files the follow-up actions; the audit is signed, not anonymous.
- **Scope boundary.** Which paved roads are in scope. Default scope is every platform the surface *should* consume for this lifecycle stage; out-of-scope exclusions are listed with a reason ("governance review is out of scope because the surface has an active review cadence owned by `team-governance` and not re-reviewed here").
- **Time window.** The audit describes the system *as of* a date. Platforms change; a six-month-old audit is history, not a current document.
- **Prior audit, if any.** Link to the last audit, so deltas (adoption of a new paved road, retirement of a deviation, emergence of a new shadow) are a diff, not a re-read.
- **Who signs off.** The audit's conclusions are circulated to the surface's product owner and the owning tech-lead; a signed-off audit is the contract for the follow-up actions. An unsigned audit is a draft.

### 2.2 Paved-road inventory

Before classifying, list the universe. The inventory names, for the lifecycle stage of the surface, every paved road that *exists* in the org (per ch.02 of the lesson). The twelve families below are the standard list for a production ML surface; a team with fewer paved roads has a shorter inventory, and a team with a bespoke platform may add rows, but none of these should be silently omitted:

1. **Feature store / feature platform.** Feature spec authoring, point-in-time correctness, serving / training parity, freshness SLA.
2. **Model registry.** Versioned artefact storage, lineage (data → training → eval → release), metadata, access control.
3. **CI/CD for ML.** Repo → test → build → deploy pipeline, canary orchestration, roll-back primitives.
4. **Training platform.** Managed training jobs, distributed-training primitives, dataset versioning, GPU scheduling, cost chargeback.
5. **Serving platform.** Managed inference serving, autoscaling, routing, request-level observability hooks.
6. **Evaluation harness.** Offline eval runner, slice definitions, LLM-judge harness, release-gate integration (ties to `mod-305/exercise-01`).
7. **Experimentation platform.** A/B framework, randomization, SRM diagnostics, pre-registered decision rules (ties to `mod-306`).
8. **Feature flagging / release orchestration.** Named flags, targeting rules, gradual rollout, kill switches.
9. **Observability / monitoring.** Metrics backend, logging, tracing, dashboards, alerting primitives (ties to `mod-307/exercise-01`).
10. **Data quality / contracts.** Schema contracts, freshness monitors, data lineage, upstream SLAs.
11. **Governance review / model-card registry.** Model cards, data statements, fairness slice results, review workflow (ties to `mod-309`).
12. **Security review / threat-model intake.** Threat-model template, ATLAS-indexed probe suites, review gates (ties to OWASP ML Top 10, MITRE ATLAS).
13. **Incident tooling / on-call paging.** On-call rotation, runbook format, incident channel template, retro archive (ties to `mod-307/exercise-03`).
14. **Cost / quota.** Chargeback, quota primitives, cost dashboards, budget alerts (ties to `mod-307/exercise-02`).

A paved road whose *team analogue* exists but whose *platform artefact* is not in use for this surface is still in the inventory — the whole point of the audit is to catch that case. Only a platform that does not exist in the org is marked `absent` with a note (which then feeds the action list as a candidate for an escalation, not a quiet shrug).

### 2.3 Consumption posture table

One row per paved road in §2.2. The classification column takes one of five values, chosen deliberately — binary adoption obscures the common case:

- **`adopted`.** The surface consumes the paved road through the supported interface, with no local wrapping that changes semantics. Minor configuration is allowed; semantic wrapping (a shim that mutates inputs or outputs before they reach the paved road) is not `adopted` — it is `partial`.
- **`partial`.** The surface consumes some of the paved road but has augmented, wrapped, or sidelined part of it. Examples: calling the model registry for artefact storage but keeping a parallel Google Doc of "canonical versions"; using CI/CD for the deploy step but hand-running the canary from a laptop.
- **`bypassed`.** The paved road is available and in scope, but the surface implements a parallel capability locally. Examples: a hand-rolled nightly feature job alongside the feature store; a Prometheus instance the team runs itself rather than the central observability backend. Bypassed roads are the most common source of drift and incidents.
- **`forked`.** The team has copied the paved-road source or spec into its own repo and modified it. Examples: a cloned eval harness with team-specific slice logic; a forked Terraform module for serving infra. Forks accumulate maintenance debt silently.
- **`absent`.** No paved road exists in the org for this capability. The team's current implementation is the local-only default. Examples: no org-wide drift-monitoring platform; no governance review workflow for internal-only surfaces.

**Columns required for every row:**

| Column | Purpose |
|---|---|
| Paved road | Name from §2.2. |
| Posture | One of `adopted / partial / bypassed / forked / absent`. |
| Paved-road artefact | The specific contract on the platform side — `feature_spec.yaml`, registry entry URL, canary orchestrator job name, eval harness run id, review packet template. Named so the reviewer can find it. |
| Current consumption | What the surface does today — the real sentence, not the aspirational one. |
| Deviation reason | For any posture other than `adopted`, the reason the team deviated. "Platform doesn't support X yet" is a reason; "we never looked" is also a reason and must be recorded honestly. |
| Deviation cost | The operational, cognitive, and risk cost the deviation imposes. See §2.4 for the shape of a cost cell. |
| Owner on paved-road side | The team that owns the paved road. Named as a role / team, not an individual, so the audit survives rotation. |
| Last-reviewed | When the posture was last confirmed. Rows older than the review cadence are flagged for re-audit. |

The table must have one row per inventory item. Rows marked `adopted` with no deviation are allowed to have a one-line "consumption" entry; the discipline is that they appear in the table, so the reviewer can see the adoption coverage explicitly and not infer it from omission.

### 2.4 Deviation-cost framing

Every non-adopted row must state the deviation cost in three honest terms, not as "no cost" or "small cost" generalities:

- **Operational cost.** Incidents the deviation can cause, pager load, maintenance hours per quarter, parity-check overhead. "On-call engineer spent ~4 h last quarter reconciling local Prometheus dashboards with the paved-road Grafana after a metric rename" is a cost; "there is some cost" is not.
- **Cognitive cost.** The mental overhead of maintaining a non-standard mental model. New team members must be told that "when you read the dashboard, remember our local scraper deduplicates by request ID differently than the paved-road one." A deviation that requires documentation to onboard is a deviation with cognitive cost.
- **Risk cost.** The probability-weighted incident severity the deviation adds. A forked eval harness can silently drift from the platform version and miss a slice regression; a bypassed feature store can serve stale features during a feature-store incident because the local pipeline never had the same alerting. Risk is *not* "could be bad"; it is a sentence naming the failure mode.

The deviation-cost cell is the *lever* for the action list. A deviation with zero honest cost across all three is probably a candidate to keep (and should be re-examined on the next audit cycle); a deviation with real operational or risk cost is a candidate to retire.

### 2.5 Drift and debt panel

For every `partial`, `bypassed`, or `forked` row, the audit records a short drift-and-debt note:

- **When the deviation began.** Approximate date and the author / event that triggered it ("hand-rolled feature job added 2026-Q2 when `feature-store-v1` did not support the long-tail-seller feature type").
- **Why it still exists.** The reason is often different from the original reason — "the platform now supports the feature type but we never migrated." The honest reason is what the action list acts on.
- **Who, if anyone, authorized keeping it.** Many deviations are authorized by default (no one said no); some have an explicit, dated waiver. Both are recorded.
- **Named next-review trigger.** Each deviation is tied to a trigger that would force a re-decision: "revisit when `feature-store-v2` ships the long-tail-seller feature type (owner: `team-feature-platform`, ETA 2026-Q4 per their roadmap)."

The drift-and-debt panel is where the audit stops being a static picture and starts being a maintenance contract. An audit without it describes the system; the panel makes the system's deviations *actionable*.

### 2.6 Action list

From the posture table and the drift panel, a prioritized action list falls out. Each action is one of five shapes:

- **Adopt.** Retire the local implementation and move to the paved road. Requires: paved-road adoption plan, migration window, rollback, and deprecation of the shadow. Owned by the surface team; the paved-road team is a reviewer, not an owner, unless the paved road's shape is also changing.
- **Contribute back.** The team authors an RFC (see `exercise-03`) to the paved-road team proposing the extension or change the team needs. Owned by the surface team's L30 engineer; the paved-road team owns review and (if accepted) implementation. This is the move when the deviation exists because the paved road has a gap.
- **Keep (with waiver).** The deviation is kept under a named-role, dated waiver that records the honest cost, the reason it is still worth paying, and a re-review trigger. "We keep the local canary runner because paved-road canary does not yet support shadow-traffic diff checks; re-review when paved-road v3 ships" is a waiver.
- **Retire.** The paved road has evolved, the deviation's reason no longer holds, and the local implementation can be removed. Requires: migration plan + dated removal + deprecation note for anyone consuming the local artefact.
- **Escalate (absent paved road).** No paved road exists; the team names a sponsor (typically a director or platform EM) and requests one. Escalations are rare but important; a universal gap in paved roads is *leadership work*, not individual-team work.

Each action has: a one-line summary, the exact row(s) it addresses, a priority tag (`P0 / P1 / P2`), an owner (role), a target date, and a success criterion. A P0 action has an incident or audit-triggered deadline; a P2 is on the roadmap but not committed-this-quarter. The priority is the senior engineer's judgement call, grounded in the deviation-cost column.

### 2.7 Review cadence

The audit is re-run on a cadence and on trigger events:

- **Quarterly** by the surface team's L30 owner. Inputs: the previous audit, new paved-road releases, incidents since the last audit, and the drift-panel next-review triggers.
- **Trigger-based.** A new paved road is launched, a paved road is deprecated, an incident attributes to a shadow deviation, the surface changes substantially (new model class, new data source, new deployment surface), or the team re-organizes. Any of these forces a re-audit out of cycle.
- **Deprecation.** When a surface is retired, the audit is archived with the retirement date so the team's deviations do not haunt the platform's deprecation planning.

### 2.8 Illustrative worked example (not a real system)

> **Illustrative feature** — `agent-assist/draft`, an internal LLM-backed support-agent assistant, auditing paved-road consumption as of 2026-11-15. Internal team: `team-agent-assist` (L30: audit owner), stakeholders: `team-support-ops`. This example is a composite for calibration; do not treat it as a real system or incident.

System under audit:

- Surface: `agent-assist/draft` endpoint, backend model `assist-llm-v4`, streaming completions; ~2k rps peak.
- Audit owner: `team-agent-assist` senior ML engineer (L30).
- Scope: all 14 paved-road families from §2.2. Governance review is in scope.
- Time window: audit as of 2026-11-15.
- Prior audit: 2026-08-14 (previous quarter).
- Signed off by: `team-agent-assist` tech-lead; `team-support-ops` product owner.

Paved-road inventory covers all 14 families; the posture table excerpt below shows six illustrative rows across the five postures. Real audits have one row per family.

| # | Paved road | Posture | Paved-road artefact | Current consumption | Deviation reason | Deviation cost | Owner | Last-reviewed |
|---|---|---|---|---|---|---|---|---|
| 1 | Feature store | `adopted` | `feature_spec.yaml` in `feature-platform-v2` registry | All 23 features for `assist-llm-v4` are declared in the shared registry with freshness SLAs; retrieval index ingest is a registered feature source. | — | — (adopted, no deviation) | `team-feature-platform` | 2026-11-10 |
| 2 | Model registry | `partial` | `model-registry` entry per release | Registry entry created on each release, but the team maintains a parallel Google Doc of "promoted prompts" for `assist-llm-v4` because prompt versions are not first-class in the registry. | Platform does not yet model prompt versions as a registry artefact class. | Op: ~2 h/release reconciling doc vs. registry. Cog: new engineers told "check the doc too, it's authoritative for prompts." Risk: a prompt rollback can be lost if the doc is out of sync with the registry. | `team-mlops` | 2026-11-12 |
| 3 | Observability | `bypassed` | Central Grafana + Prometheus via `obs-platform` | Team runs a local Prometheus instance for `assist-llm-v4` latency metrics because the central scraper's 60 s scrape interval is too coarse for streaming TTFT. Central dashboard shows a lagged view; local one is "truth." | Scrape-interval limitation; no support for sub-15 s scrape on the paved road. | Op: on-call must look at two dashboards during incidents (~15 min overhead per page last quarter × 4 pages). Cog: onboarding doc explicitly says "look at the local one first." Risk: central alert thresholds are tuned against a different data source, so a real latency burn can underfire. | `team-observability` | 2026-11-12 |
| 4 | Evaluation harness | `forked` | `ml-eval-harness v3` in the shared repo | `team-agent-assist` maintains `ml-eval-harness-forked-agent` with team-specific judge-model logic and slice definitions for support-ticket categories. Fork diverged in 2026-Q2 and tracks `ml-eval-harness v3` monthly. | Original harness did not support LLM-judge evaluations against a per-category rubric; fork added the capability. | Op: ~1 engineer-day/month tracking upstream. Cog: new engineers learn a non-standard harness. Risk: silent drift in slice definitions between fork and platform version → a release passes the forked harness but a slice regression is only visible in the platform version. | `team-model-eval` | 2026-11-13 |
| 5 | Drift monitoring platform | `absent` | *(none exists)* | Team wrote a bespoke drift job that runs nightly and writes to a local dashboard; no platform equivalent exists. | No org-wide drift-monitoring paved road. | Op: ~0.5 engineer-day/month. Cog: no org vocabulary for drift (each team invents its own PSI threshold). Risk: no cross-team incident learning when one team's drift pattern repeats in another. | *(escalation candidate)* | 2026-11-14 |
| 6 | Governance review | `partial` | Model card template in `governance-registry` | Model card exists but is not updated per release; last updated 2026-Q2 despite three releases since. | Release cadence (weekly) exceeds the team's muscle for model-card maintenance; no automation ties release → model-card diff. | Op: card is stale, so governance reviewers rely on verbal updates. Cog: team is unsure which card is current for which model version. Risk: a governance-visible regression could ship without a model-card update, breaking the review contract. | `team-governance` | 2026-11-15 |

(Illustrative; real audit has 14 rows.)

Drift-and-debt notes for the deviations:

- **Row 2 (model registry, prompt versions).** Began 2026-Q2 when the first prompt was promoted. Why still exists: platform's prompt-version proposal is on the roadmap for 2027-Q1 but not committed. Authorization: no formal waiver; adopted-by-default. Next-review trigger: platform ships prompt-version RFC (or earlier, if a prompt-rollback incident lands).
- **Row 3 (observability, local Prometheus).** Began 2026-Q1 when TTFT SLI was added. Why still exists: platform's scrape-interval limitation unchanged. Authorization: dated waiver (2026-03-15) from the team's engineering manager, reviewed quarterly. Next-review trigger: `obs-platform` ships 10 s scrape (rumoured for 2026-Q4; on their roadmap per the quarterly sync).
- **Row 4 (eval harness, fork).** Began 2026-Q2. Why still exists: original gap closed in `ml-eval-harness v4` (shipped 2026-Q3), but the team has not migrated. Authorization: original fork had a waiver; it has since expired. Next-review trigger: *now*. This is the primary P0 action out of this audit.
- **Row 5 (drift platform, absent).** Began 2026-Q1 when drift jobs were added. Why still exists: no platform. Next-review trigger: escalation to platform leadership via the quarterly platform-review council.
- **Row 6 (governance, stale card).** Began 2026-Q2 after release cadence accelerated. Why still exists: no CI gate forces card updates. Next-review trigger: *now* — stale cards are a release-gate issue by the policy declared in `mod-309`.

Action list (illustrative):

- **P0 — Retire eval-harness fork (row 4).** The upstream gap is closed. Migrate to `ml-eval-harness v4`, retire the fork, update release-gate wiring. Owner: `team-agent-assist` L30 + `team-model-eval` as reviewer. Target: 2026-Q4. Success: the fork's repo is archived, the release gate reads from the paved-road eval harness, and the team's slice definitions are contributed back (see exercise-03). 
- **P0 — Update governance model card per release (row 6).** Add a CI gate that fails release if the model card's `last_updated` is older than the release's commit range. Owner: `team-agent-assist` L30 + `team-governance`. Target: 2026-Q4. Success: the CI gate is in place and three consecutive releases update the card.
- **P1 — Contribute-back RFC for prompt versions in model registry (row 2).** Author the RFC to the model-registry team. Owner: `team-agent-assist` L30. Target: 2026-Q4 RFC submitted. Success: RFC is accepted (or counter-proposed) by `team-mlops`; see exercise-03 for the RFC body.
- **P1 — Keep local Prometheus with waiver (row 3).** Renew the dated waiver for one more quarter. Owner: `team-agent-assist` engineering manager. Success: waiver renewed in writing with a 2026-Q4 re-review trigger pinned to the `obs-platform` v4 release.
- **P2 — Escalate absence of drift platform (row 5).** Draft a one-pager for the platform-review council requesting an org-wide drift platform. Owner: `team-agent-assist` L30 + peer teams who also run bespoke drift jobs. Target: 2027-Q1 council. Success: council decision recorded.

Review cadence:

- Quarterly audit (next: 2027-02-15), owner: `team-agent-assist` L30.
- Trigger-based re-audit on any of: `feature-platform-v3` launch, `ml-eval-harness v5` launch, `obs-platform v4` launch, any incident attributed to a shadow deviation, any re-org of `team-agent-assist` or peer platform teams.

### 2.9 Optional runnable sketch: `audit_lint.py`

Audits are easier to keep honest with a lightweight linter that reads the Markdown and asserts the discipline. The script is deliberately small; its job is to prevent regressions in audit quality over quarters, not to replace the audit itself.

```python
# audit_lint.py
"""Lint a paved-road audit Markdown artefact for completeness.

A reviewer still reads the audit. This script enforces the mechanical
discipline so the reviewer's attention goes to the judgement calls.
"""

from dataclasses import dataclass
import re
import sys


REQUIRED_SECTIONS = [
    "System under audit",
    "Paved-road inventory",
    "Consumption posture",
    "Drift and debt",
    "Action list",
    "Review cadence",
]

VALID_POSTURES = {"adopted", "partial", "bypassed", "forked", "absent"}

REQUIRED_PAVED_ROADS = [
    "feature store", "model registry", "ci/cd", "training platform",
    "serving platform", "evaluation harness", "experimentation",
    "feature flag", "observability", "data quality", "governance",
    "security review", "incident tooling", "cost",
]


@dataclass
class AuditFinding:
    severity: str  # 'error' or 'warn'
    message: str


def lint(audit_text: str) -> list[AuditFinding]:
    findings: list[AuditFinding] = []
    lowered = audit_text.lower()

    for section in REQUIRED_SECTIONS:
        if section.lower() not in lowered:
            findings.append(AuditFinding("error", f"missing section: {section}"))

    for road in REQUIRED_PAVED_ROADS:
        if road not in lowered:
            findings.append(AuditFinding("warn", f"paved road not referenced: {road}"))

    posture_tokens = re.findall(r"`(adopted|partial|bypassed|forked|absent)`", audit_text)
    if not posture_tokens:
        findings.append(AuditFinding("error", "no posture classifications found"))
    unseen_postures = VALID_POSTURES - set(posture_tokens)
    if unseen_postures:
        findings.append(AuditFinding(
            "warn",
            f"postures never used (expected at least one of each in a non-trivial audit): {sorted(unseen_postures)}",
        ))

    if "last reviewed" not in lowered and "last-reviewed" not in lowered:
        findings.append(AuditFinding("error", "no 'last reviewed' date found"))

    if "waiver" not in lowered:
        findings.append(AuditFinding(
            "warn",
            "no 'waiver' language — audits with any non-`adopted` row usually record at least one waiver or action to retire",
        ))

    if not re.search(r"\bP[012]\b", audit_text):
        findings.append(AuditFinding("error", "no priority tags (P0/P1/P2) found in action list"))

    return findings


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("usage: audit_lint.py <path-to-audit.md>")
        sys.exit(2)
    text = open(sys.argv[1]).read()
    findings = lint(text)
    for f in findings:
        print(f"{f.severity:>5}: {f.message}")
    errors = [f for f in findings if f.severity == "error"]
    sys.exit(1 if errors else 0)
```

The linter is not a substitute for a human reviewer. Its job is to fail CI when an audit skips a section, forgets a date, or omits a priority tag — all of which are mechanical mistakes that would otherwise consume a reviewer's attention budget.

## 3. Validation steps

Walk the audit through this checklist before considering it done:

1. **System-under-audit section is non-generic.** The one-paragraph scope names a specific surface, model version, team, and date. A reviewer cannot audit an audit without this.
2. **Paved-road inventory covers the 14 families** (or clearly labels an exclusion with a reason). Omitting a family is the first failure mode.
3. **Every inventory item appears as a row** in the posture table, including `adopted` rows. Omission by exception hides adoption gaps.
4. **Each posture is one of the five defined values**; "mostly adopted" is not a posture.
5. **Deviation-reason cells name the honest reason**, including "we never looked" where that is true. Reasons that are only aspirations ("we plan to adopt eventually") are flagged.
6. **Deviation-cost cells are specific** — named pager hours, named onboarding overhead, named failure mode. "Low cost" without evidence is not a cost cell.
7. **Drift-and-debt panel exists for every non-adopted row**, with a date, an authorization note, and a next-review trigger.
8. **Action list has priorities (`P0 / P1 / P2`)**, owners expressed as roles, target dates, and success criteria.
9. **At least one action is a contribute-back candidate** (ties to `exercise-03`) or is explicitly ruled out with a reason.
10. **Review cadence is named** and the last-reviewed date is within the cadence.
11. **NIST AI RMF GOVERN is cited** for the audit-as-governance framing; MANAGE for the action-list-as-risk-control framing.
12. **OWASP ML Top 10 and MITRE ATLAS are cited** on the security-review paved-road row, if the surface has a safety/abuse surface.

Static checks a CI step can enforce on `paved-road-audit.md`:

```bash
grep -ciE "adopted|partial|bypassed|forked|absent"       paved-road-audit.md   # postures present
grep -ciE "feature store|model registry|ci/cd|eval"      paved-road-audit.md   # standard inventory
grep -ciE "deviation cost|operational|cognitive|risk"    paved-road-audit.md   # cost framing
grep -ciE "action list|P0|P1|P2"                         paved-road-audit.md   # prioritized actions
grep -ciE "last reviewed|review cadence"                 paved-road-audit.md   # cadence
grep -ciE "nist ai rmf|govern|manage"                    paved-road-audit.md   # standards cited
```

`audit_lint.py` from §2.9 gives a programmatic version of the same checks.

## 4. Rubric / review checklist

- [ ] System under audit names surface, owner role, scope boundary, time window, prior audit link, signed-off reviewers.
- [ ] Paved-road inventory names all 14 families or declares exclusions with reasons.
- [ ] Posture table has one row per inventory item — `adopted` rows included for coverage legibility.
- [ ] Every posture value is one of `adopted / partial / bypassed / forked / absent`.
- [ ] Deviation-reason cells are written honestly (platform gap, legacy, never-looked, team preference) rather than as aspirations.
- [ ] Deviation-cost cells name operational cost, cognitive cost, and risk cost in specific terms (hours, failure modes, slice regressions).
- [ ] Drift-and-debt panel exists for every non-adopted row, with begin date, why-still-true, authorization, and next-review trigger.
- [ ] Action list uses `P0 / P1 / P2` with role-owners, target dates, and named success criteria.
- [ ] At least one action is a contribute-back candidate (or explicitly ruled out).
- [ ] At least one action is an adopt-and-retire-shadow if any `bypassed` or expired-waiver row exists.
- [ ] At least one action is an escalation if any `absent` row has an operational or risk cost.
- [ ] Review cadence (quarterly default) named with last-reviewed date in cadence and trigger-based re-audit conditions listed.
- [ ] NIST AI RMF GOVERN and MANAGE cited for the audit-as-governance and action-as-risk-control framings.
- [ ] OWASP ML Top 10 and MITRE ATLAS cited on the security-review row for any surface with a safety/abuse exposure.
- [ ] Cross-references to `exercise-02` (build-vs-adopt decision for any `absent` or `bypassed` row where "build locally" is on the table) and `exercise-03` (the contribute-back RFC the audit identifies).

Review-weighted points for grading:

- System-under-audit clarity (scope, owner, signatures): 10%
- Paved-road inventory coverage (all 14 families or justified exclusions): 15%
- Posture table completeness and classification discipline: 20%
- Deviation-reason and deviation-cost honesty and specificity: 20%
- Drift-and-debt panel per non-adopted row: 10%
- Action list with priorities, owners, dates, success criteria: 15%
- Standards alignment (NIST AI RMF GOVERN / MANAGE, OWASP ML Top 10 / MITRE ATLAS for safety) and hygiene: 10%

## 5. Common mistakes

- **Audit by exception.** Listing only the deviations leaves the reviewer guessing at adoption coverage. Every inventory item gets a row, even the quiet `adopted` ones.
- **"Mostly adopted" postures.** Five classifications exist for a reason. A paved road that is wrapped by a semantic shim is `partial`, not `adopted`. Classification is binary within each row even if the system-wide picture is grey.
- **Deviation reason as aspiration.** "We'll migrate soon" is not a reason; it is a plan. The reason is why the deviation existed when the audit ran. The plan goes in the action list.
- **"Low cost" with no evidence.** A deviation cost that cannot be named in hours, failure modes, or onboarding overhead is a deviation cost that was not measured. If the team cannot name the cost, the honest cell says "unknown — to measure next quarter," not "low."
- **Forgetting the forks.** `forked` rows are often the oldest deviations and the hardest to see because the fork has become the team's mental model. The audit asks explicitly: is any artefact in your repo a copy-with-modifications of a platform artefact? If yes, it is a `forked` row.
- **Treating `absent` as a quiet shrug.** A paved road that does not exist is a signal to leadership, not a resigned "we'll build locally." The action for `absent` is either escalate (ask for one) or explicitly run the `build-vs-adopt` decision (exercise-02) with "no paved road exists" as the key input.
- **Action list without priorities.** `P0 / P1 / P2` is the senior engineer's calibration of risk. Without priorities, the action list is a backlog — a lot of words, no next step.
- **Owner as a person, not a role.** "Jane will do it" is not an action list entry; Jane changes teams. "`team-agent-assist` L30 owner" survives rotation.
- **Waivers without expiry.** A waiver with no re-review trigger becomes a permanent entitlement. Every waiver names a trigger (platform ships X, incident lands, next quarter).
- **Running the audit alone.** The audit's signatures (tech-lead + product owner) are what make it the surface team's contract. An audit signed only by the audit owner is a draft, not a decision.
- **No cross-reference to the contribute-back RFC.** Many deviations exist because the paved road has a gap. The audit is the place the gap is named; `exercise-03` is the place it is proposed-against. An audit that never surfaces an RFC candidate is probably an audit that never looked hard enough.
- **Ignoring the lifecycle dimension.** A pre-production surface and a mature surface audit against different inventories (a pre-prod surface is often `absent` on governance review by design). The audit's lifecycle note (scope) tells the reviewer which inventory applies.
- **Running the audit once.** A one-shot audit is a snapshot; the paved-road landscape changes quarterly. The review cadence is the artefact that keeps the audit from decaying into history.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds the audit as a governance instrument that keeps an ML surface's platform dependencies legible to leadership and reviewers. MANAGE function grounds the action list as the risk-control surface over those dependencies.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Cited on the security-review paved-road row for the threat classes any adoption of the security-review paved road must cover (adversarial inputs, data poisoning, supply-chain and dependency risk).
- MITRE ATLAS — <https://atlas.mitre.org/>. Cited for the adversarial-tactic taxonomy that a security-review paved road or its probe suites are built against.

Local context (authoritative for the audit's shape and for the cross-references):

- Paired lesson: `lessons/mod-308-platform-collaboration/exercises/exercise-01-paved-road-consumption-audit.md`.
- Module README: `modules/mod-308-platform-collaboration/README.md`.
- Related exercise (build-vs-adopt tradeoff that the audit's `absent` and `bypassed` rows feed into): `modules/mod-308-platform-collaboration/exercise-02-build-vs-adopt-tradeoff-doc/SOLUTION.md`.
- Related exercise (contribute-back RFC that the audit's `partial` and `forked` rows feed into when the deviation reason is a platform gap): `modules/mod-308-platform-collaboration/exercise-03-contribute-back-rfc-to-platform/SOLUTION.md`.
- Related exercise (role hand-off map from the senior-ML role module, whose peer-platform category is the structural counterpart to the audit's paved-road inventory): `modules/mod-301-senior-ml-role-scope/exercise-02-role-hand-off-map/SOLUTION.md`.
- Related exercise (SLO document whose monitoring contract is the observability paved road being consumed): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (cost-budgets artefact whose cost / quota paved road the audit maps against): `modules/mod-307-ml-reliability-slos/exercise-02-cost-budgets-and-quotas/SOLUTION.md`.
- Related exercise (offline evaluation harness whose consumption posture the audit measures against): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
