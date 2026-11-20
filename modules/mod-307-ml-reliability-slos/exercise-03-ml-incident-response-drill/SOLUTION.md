# exercise-03 — ML Incident Response Drill (Reference Solution)

Paired exercise: `lessons/mod-307-ml-reliability-slos/exercises/exercise-03-ml-incident-response-drill.md`.

## 1. Solution overview

This exercise produces three paired artefacts — `runbook.md`, `rollback.md`, and `postmortem-template.md` — plus a drill log (`drill-2026-11.md`) that captures one tabletop exercise in which an on-call team walks through a hypothetical ML incident end-to-end. The artefact has seven parts:

1. A **drill frame** that pins down what the drill rehearses, who participates, what counts as "complete," and how often the drill repeats.
2. A **runbook** — the first-hour, first-four-hour, and first-day playbook for the on-call engineer: how to triage, communicate, mitigate, and escalate an ML incident, distinct from a generic SRE page because the mitigation surface includes model rollback, retraining pause, prompt pinning, feature-store rollback, and safety-filter override.
3. A **rollback plan** — the pre-registered, version-pinned, named-role rollback steps for each class of change (model version, prompt template, feature pipeline, policy config), with recovery-time objectives (RTO) and recovery-point objectives (RPO).
4. A **postmortem template** — the structured incident write-up, blameless, aligned with the Google SRE postmortem shape, extended with ML-specific sections (training data, dataset version, drift state, safety-filter state at the time of incident).
5. A **drill walkthrough** — the tabletop rehearsal itself, with a chosen hypothetical scenario, timed phases, a facilitator script, and the artefacts each role produces during the exercise.
6. A **cross-reference table** — the mapping from SLO burns (from `exercise-01`) and cost-cap breaches (from `exercise-02`) to the runbook entry points, so an alert route resolves to a runbook section, not to "page whoever owns this surface."
7. A **follow-up contract** — what the team commits to after the drill: runbook edits, missing-automation tickets, next drill date, and the review cadence that keeps the runbook from going stale.

The reference solution below shows the structure, works a labeled illustrative drill through it, and gives the grading rubric.

The exercise exists because ML incidents fail modes do not match generic service incident modes. A model that quietly drifts to serving degraded predictions at nominal latency is an incident even though every HTTP response is 200. A policy filter that stops catching prompt-injection is an incident even though no SLO alerts fired. A retrain that promoted a model whose canary passed but whose production slice regressed is an incident whose rollback target is a dataset and a version pin, not "push a config." The runbook makes those distinctions concrete, the rollback plan makes the fix reversible, and the postmortem feeds organizational learning back into the SLOs (`exercise-01`), the cost policy (`exercise-02`), and the retrain pipeline (`exercise-04`). NIST AI RMF GOVERN grounds the role-and-responsibility structure of incident response; MANAGE grounds the mitigation and recovery activities themselves. OWASP ML Top 10 and MITRE ATLAS name the adversarial failure modes a safety-related runbook entry must cover.

## 2. Worked answer / implementation

### 2.1 Drill frame

Six lines, in this order:

- **What the drill rehearses.** A specific class of incident on a specific surface. "A latency incident on `agent-assist/draft`," not "incidents." Each drill rehearses one class; a quarterly schedule rotates classes.
- **Who participates.** Roles, not people. On-call engineer, incident commander (IC), model owner, safety reviewer, product owner, communications lead. The drill fills each role; a role with no assignee during the drill is the gap the drill was for.
- **What counts as complete.** The drill ends when: incident is declared, mitigation is in progress, user communication is drafted, a rollback target is named, and a postmortem skeleton exists. Not when "everyone feels better."
- **How often it repeats.** Quarterly is the common cadence; monthly for high-risk or recently-restructured teams. The cadence is named so the drill schedule survives attention cycles.
- **Facilitator.** A role outside the on-call rotation (reliability lead, platform SRE, or external peer) who runs the tabletop and times phases. A drill facilitated by the person being drilled is a performance; a drill with an external facilitator is a measurement.
- **Last run.** A date. A runbook last drilled 18 months ago is not a runbook.

### 2.2 Runbook

The runbook is a document that survives a page at 02:00. Terse, versioned, searchable. It has four layers.

**R1. Entry map.** The one-page index that maps an alert name to a runbook section. The on-call engineer who gets paged clicks the alert link and lands here, not on page 23 of a wiki.

```
Alert: slo-burn-page-latency-agent-assist     → §R3.a Latency SLO burn
Alert: slo-burn-page-availability-agent-assist → §R3.b Availability SLO burn
Alert: slo-burn-page-quality-agent-assist      → §R3.c Quality SLO burn
Alert: cost-cap-breach-agent-assist            → §R3.d Cost cap breach
Alert: safety-signal-spike-agent-assist        → §R3.e Safety / policy incident
Alert: drift-high-agent-assist                 → §R3.f Drift / data-health incident
Alert: retrain-canary-fail-assist-llm          → §R3.g Retrain canary failure
Alert: dependency-down-llm-gateway             → §R3.h Upstream dependency incident
```

Every alert the on-call rotation can receive on this surface maps to a section. An alert without a section is a page that will escalate to a slack question instead of a mitigation.

**R2. First-five-minutes checklist.** The actions the on-call takes before any investigation, in order. Short. Tangible.

```
0:00  Acknowledge the page (so paging does not re-fire the rotation).
0:01  Open the surface's SLO dashboard and the trace sampler link in the alert.
0:02  Check: is the surface in a known maintenance window? Is a retrain or
      deploy in progress right now?
0:03  Declare incident severity using the severity table (§R4). If SEV-1 or
      SEV-2, open the incident channel; assign IC (default: the on-call).
0:04  Post the "triage started" message in #incidents with alert, severity, IC,
      and dashboard link.
```

The first five minutes are the only part of the runbook that is universal. Everything after branches on the class of incident.

**R3. Incident-class playbooks.** One per alert class, with identical shape so the on-call does not have to re-learn the structure.

Each playbook has:

- **Signal.** What the alert measures and what it looks like when real vs. noisy.
- **Immediate questions.** The five questions whose answers triage the incident:
    - *What changed?* (deploys, flags, retraining, dataset refresh, upstream provider announcement, traffic shift)
    - *Is the signal the symptom or the cause?* (a latency burn might be an upstream dependency, not the surface itself)
    - *What is the blast radius?* (one tenant, one region, one slice, all traffic)
    - *Is user harm in progress?* (safety or policy breach makes mitigation urgent even if SLO is intact)
    - *Can we mitigate without full root cause?* (rollback, degrade, re-route; see §R3's mitigation list)
- **Mitigation menu.** Pre-registered actions and their effects, ordered from least to most invasive.
- **Rollback trigger.** The condition that moves the incident from "mitigate" to "roll back" (`rollback.md`, §2.3).
- **Comms template.** A fill-in-the-blanks user-facing status message and an internal update message.
- **Hand-off contract.** When the incident lasts past shift boundary, what gets handed off and to whom.

Shape of §R3.a (Latency SLO burn) — illustrative:

```
Signal
  The latency SLI dropped below target and the MWMB burn-rate page fired.

Immediate questions
  1. What changed in the last 24h on this surface? (Check deploy log,
     retraining log, feature-store refresh log, flag-change log.)
  2. Is the burn broad (all routes) or narrow (one route, one segment)?
  3. Is the upstream (llm-gateway, embed-serve) also burning latency?
  4. Is the error budget pacing to zero within the shift?
  5. Is the rollback from the most recent deploy safely available?

Mitigation menu (least → most invasive)
  M1. Re-route traffic to healthy region (if multi-region).
  M2. Raise cache TTL for cacheable routes (reduces load on model).
  M3. Shed non-essential traffic via cost policy's 429 path (exercise-02).
  M4. Pin to previous model version (model rollback — see rollback.md).
  M5. Pin to previous prompt template (prompt rollback — see rollback.md).
  M6. Full surface roll back to last known good config bundle.

Rollback trigger
  Latency SLO burn exceeds "2x page threshold for 30 minutes" OR the
  surface's error budget hits 0% remaining. Both conditions are pre-
  registered; neither requires a meeting.

Comms template (user-facing)
  "We are investigating elevated response times on <surface> affecting
  <scope>. Users may experience <impact>. Next update by <timestamp>."

Comms template (internal)
  "SEV-<n>: <surface> latency burn. IC: <role>. Current mitigation:
  <M1..M6>. Rollback candidate: <version>. Next review: <timestamp>."

Hand-off
  Current IC posts a hand-off block with: timeline, hypotheses, actions
  in flight, rollback-trigger status, next-update time.
```

The other playbooks (§R3.b–§R3.h) follow the same shape. The safety playbook (§R3.e) is the one with the most ML-specific structure; its mitigation menu includes safety-filter tightening, prompt-injection-blocking regex deploys, abuse-pattern IP block, and immediate rollback of any recent prompt or model change aligned with OWASP ML Top 10 scenarios (prompt-injection success, output integrity, model DoS) and MITRE ATLAS tactics.

**R4. Severity table.** Three levels, deterministic.

```
SEV-1  User-visible, widespread, or safety-critical. Page IC, model
       owner, product owner, comms lead immediately. Target MTTD < 5m,
       MTTR < 30m, status page updated within 15m.
       Triggers: safety/policy breach; > 50% traffic affected;
                 revenue-impacting; media-visible.

SEV-2  User-visible, limited scope OR non-user-visible but likely to
       become SEV-1 within the shift. Page IC and model owner. MTTD < 15m,
       MTTR < 4h.
       Triggers: single-tenant outage; one segment degraded; cost cap
                 breached without user harm.

SEV-3  Internal only, no user impact. Ticket not page. MTTD ≤ next
       business day; MTTR ≤ 1w.
       Triggers: monitoring pipeline gap; drift-SLI burn without
                 user-observable regression; attribution-completeness
                 degradation.
```

Severity is assigned at declaration. Escalation up a level requires a logged reason and triggers the higher-severity notification path. Severity does not downgrade automatically; it downgrades only after a human review posts the rationale.

### 2.3 Rollback plan

Rollback is the discipline that makes an incident recoverable. A mitigation that cannot be reversed is a risk compounding on the incident; the rollback plan names what can be rolled back, how, by whom, and in how long.

**Rollback classes.**

- **Model version rollback.** The serving layer pins to a previous model version. Precondition: previous version is retained for ≥ 28 days, loaded in a warm shadow, and its own canary signals are green. RTO: ≤ 5 minutes once the decision is made (the serving infra is already warm). RPO: 0 (no in-flight state is lost; requests mid-flight either complete or retry on the previous version).
- **Prompt / template rollback.** The prompt-as-code store (`mod-304/exercise-02`) rolls back to the previous template version. Precondition: templates are version-pinned, stored declaratively, and deployable via the same path as model config. RTO: ≤ 2 minutes (config change, not model load).
- **Feature pipeline rollback.** The feature store rolls back to a previous snapshot or a previous transform version. RTO: longer, often 10–60 minutes, because feature data is larger than config and may need re-materialization. RPO: 0 at the feature layer — the snapshot is a point-in-time copy — but downstream caches may need warm-up.
- **Policy / safety-filter rollback.** The policy layer rolls back to a previous ruleset or model version. RTO: ≤ 5 minutes. Safety-filter rollback is special in that rollback *forward* (to a stricter filter) is sometimes correct when the incident is a safety breach; the plan names both directions.
- **Full config bundle rollback.** When model, prompt, policy, and feature pipeline were all changed together, the rollback is a bundle rollback to the previous named release. RTO: ≤ 15 minutes if the bundle is pre-pinned.

**Rollback contract.**

For each class the plan records:

- **Precondition.** What must be true before rollback is safe: previous version is retained, warm, and passed canary; snapshots exist; config store supports atomic rollback.
- **Trigger.** The pre-registered signal that starts rollback. Typically the rollback trigger from the matching runbook playbook (§R3.*).
- **Executor role.** Which on-call role has the authority to roll back without an approval meeting. For model and prompt rollbacks, the on-call is enough. For feature pipeline rollbacks, the data-platform on-call may also be required. The plan names roles so the drill tests role presence.
- **Steps.** The exact commands or console actions, in order. Not "re-deploy the previous version"; the specific command invocation. Steps are tested in the drill.
- **Verification.** What signals confirm the rollback succeeded: latency restored, quality SLI recovered on the canary slice, safety-probe-pass rate restored.
- **Post-rollback gate.** What must be true before traffic returns to normal routing: canary signals green, SLO burn plateaued, incident channel confirms.
- **Communication.** The update that goes out at rollback start, rollback complete, and incident close.

**Non-reversible changes.** The plan explicitly lists changes that are *not* rollback-safe: a database schema migration applied in the same release, a training run that burned disk the retention policy already freed, a logged-out user whose identity-provider state no longer matches the previous policy. For each, the plan names the forward-fix path and warns that rollback is not available; this is the information that stops a rollback decision from making a recoverable incident unrecoverable.

### 2.4 Postmortem template

The postmortem is a structured write-up produced within one week of incident closure. It is blameless by charter — the subject is systems and decisions, not individuals. The template below extends the generic SRE postmortem shape with ML-specific sections.

```
# Postmortem: <surface> <incident class>  (<SEV-n>)

## Metadata
Incident start:       <ts>
Incident declared:    <ts>
Incident mitigated:   <ts>
Incident closed:      <ts>
MTTD (declared - start):      <duration>
MTTR (mitigated - declared):  <duration>
IC:                   <role>
Scribes:              <role>
Impacted surfaces:    <list>
Impacted users:       <scope + count estimate>

## Summary (2–4 sentences)
What happened, from a user's perspective, and what we did about it.

## Timeline
hh:mm:ss  <event>
...

## Impact
Users affected:      <count / segment>
SLO burned:          <SLOs and % budget consumed>
Cost impact:         <excess spend or none>
Revenue impact:      <estimate or "none observed">
Safety impact:       <scope / severity / probes affected>
Data integrity:      <any dataset or feature-store corruption>

## What happened
### Contributing factors (not "root cause")
Enumerate the chain of conditions that made the incident possible.
Explicitly: what changed, what signals were missed, what
assumptions broke.

### ML-specific state at time of incident
Model version(s):       <list>
Prompt template(s):     <list>
Dataset version(s):     <list>
Feature snapshot:       <identifier>
Drift state (pre-inc):  <values from drift SLI>
Safety-filter version:  <id>
Retraining window:      <most recent train run id, canary verdict>
Guardrails that held:   <list>
Guardrails that did not fire: <list>

## Detection
How did we find out? (SLO burn, user report, internal observation, external disclosure)
How did we not find out earlier?
Monitoring gaps named — tickets filed.

## Response
What went well: <bullets>
What went poorly: <bullets>
Where the runbook helped: <bullets>
Where the runbook was missing or wrong: <bullets>

## Mitigation and recovery
Which mitigation steps ran (M1..M6), in order, with their observed effect.
If rolled back: which class (model / prompt / feature / policy / bundle),
preconditions checked, observed recovery.

## Action items
Each action item has: owner (role), due date, type (preventive / detective /
mitigative), link to tracker ticket. Action items are the only part of
the postmortem that outlive the week; they are tracked to closure.

## Lessons for the SLOs, the cost policy, and the retrain pipeline
Does the SLO need recalibration? (exercise-01)
Does the cost policy need a new quota or degradation step? (exercise-02)
Does the retrain pipeline need a new canary gate or guardrail? (exercise-04)

## Blameless statement
A paragraph acknowledging that the subject is the system, not the individuals.
Not performative — it sets the frame for the next contributor to read
the postmortem without defensive filtering.
```

The blameless frame is not a politeness. It is the condition under which the ML-specific state section gets filled in truthfully: engineers write down "the canary-gate threshold was 0.8 and we lowered it to 0.75 last sprint under pressure" only if that admission feeds a fix instead of a blame.

### 2.5 Drill walkthrough

A drill is a tabletop. No production change is made during the drill; mitigations and rollbacks are *simulated* and the artefacts (incident channel transcript, status update drafts, rollback-step walkthrough, postmortem skeleton) are the drill outputs.

```
Phase 0 — Scenario reveal (facilitator)
  Facilitator reads a prepared scenario:
    "At 14:03 UTC, the latency SLO burn-rate page fires on agent-assist/draft.
     Dashboard shows p95 doubled in the last 10 minutes. The 1h burn rate is 18.
     No deploy was run in the last 24h. Upstream llm-gateway shows a slight
     latency uptick but within its SLO. Three user reports in #support in the
     last 5 minutes mention slow drafts."
  Role assignments posted. Timer starts. (Target: 45 minutes to Phase 5.)

Phase 1 — First five minutes (on-call)
  On-call runs §R2 checklist aloud. Facilitator notes what is missing
  (did they check the deploy log? the flag-change log? did they
  acknowledge the page?).

Phase 2 — Triage (IC + on-call + model owner)
  Team works §R3.a Immediate Questions. Facilitator plays the role of
  dashboards and injects new information at timed intervals:
  "14:12 — upstream status page now reports regional degradation."
  "14:18 — quality SLI unaffected; it's purely latency."
  Team settles on mitigation plan (M1 re-route region? M3 shed non-essential?).

Phase 3 — Mitigation walkthrough (on-call)
  On-call walks the exact commands or console actions to execute M1 and M3.
  Facilitator stops them if a command is wrong, missing a flag, or would
  require an approval the runbook does not pre-authorize. No real command
  is executed.

Phase 4 — Rollback decision (IC)
  Facilitator announces "it is now 14:42; mitigation has not reduced the
  burn." IC must decide: continue, or invoke rollback. If rollback, team
  walks through the §R2.3 rollback contract — precondition check,
  executor, steps, verification, communication.

Phase 5 — Comms, hand-off, and postmortem skeleton (comms lead + IC)
  Comms lead produces the user-visible status update and the internal
  update. IC drafts a hand-off block for the next shift. Scribe starts
  the postmortem skeleton using §R2.4, filling in metadata, timeline,
  impact, and ML-specific state as the drill produced them.

Phase 6 — Debrief (all)
  Facilitator calls time. Group walks:
    - Which runbook sections were used?
    - Which were missing, wrong, or stale?
    - Which rollback precondition was uncertain?
    - Which role was unrepresented?
    - Which communication template needed editing live?
  Each finding is a ticket. The drill log (drill-2026-11.md) captures
  them with named owners and dates.
```

The drill is scored on **artefacts produced**, not on "team felt confident." Three artefacts are mandatory: the incident-channel transcript, the user-facing comms draft, and the postmortem skeleton. A drill that ends without all three is a drill that did not finish.

### 2.6 Cross-reference table

The runbook sits in the middle of a graph of surrounding policies. The cross-reference table makes the edges explicit.

```
Trigger (what fires)                              Runbook entry       Downstream artefact
-----------------------------------------------   ----------------    -----------------------------
SLO burn page (exercise-01 §2.4)                  R3.{a,b,c}          Postmortem (§2.4);
                                                                       SLO re-review (exercise-01 §2.7)

Cost cap breach alert (exercise-02 §2.6)          R3.d                Postmortem;
                                                                       cost-policy update (exercise-02)

Retrain canary failure (exercise-04 §2.3)         R3.g                Postmortem;
                                                                       retrain-pipeline hardening (exercise-04)

Safety signal / probe-set regression              R3.e                Postmortem;
                                                                       safety-filter update;
                                                                       OWASP ML Top 10 scenario review

Drift SLI burn (exercise-01 §2.2 F5)              R3.f                Postmortem;
                                                                       possible retrain trigger (exercise-04)

Upstream dependency outage                        R3.h                Postmortem;
                                                                       dependency-SLO renegotiation (exercise-01 §2.1)
```

Every alert the surface emits has a row here. A row without an alert is dead policy; an alert without a row is a surprise page.

### 2.7 Follow-up contract

The drill ends with commitments. The log records them in a format that is easy to audit at the next drill.

```
Finding ID   Finding                                         Owner (role)         Due date      Type
F-01         §R3.e missing explicit rollback-to-stricter-     safety reviewer      2026-12-05    preventive
             filter step; team invented it live.
F-02         Model rollback preconditions unclear on          model owner          2026-12-01    detective
             whether warm-shadow was green at incident time.
F-03         Comms template §R3.a uses "elevated response     comms lead           2026-11-27    mitigative
             times" without a quantitative pattern; unclear
             what to say if it's 2x vs. 10x.
F-04         Severity table §R4 does not name "safety         IC                   2026-12-05    preventive
             signal breach" as SEV-1 explicitly.
Next drill date: 2027-02-15. Class: cost-cap breach.
```

Review cadence: drill log is reviewed at the next drill's debrief. Open findings are either closed, re-assigned, or escalated. A finding that has rolled across three drills is an escalation to engineering management, not another ticket.

### 2.8 Illustrative worked drill (not a real incident)

> **Illustrative scenario** — the same `agent-assist/draft` surface from `exercise-01` and `exercise-02`. Composite for calibration; do not treat as a real incident.

Scenario — latency SLO burn at 14:03 UTC caused by an upstream provider regional degradation, compounded by a cache-hit-ratio drop after a feature-store refresh four hours earlier.

Severity: declared SEV-2 at 14:08 UTC; escalated to SEV-1 at 14:21 UTC when the burn broadened past 50% of traffic.

Mitigation sequence:

- M1 (region re-route) — 14:12 UTC. Reduced burn rate from 18 to 9.
- M3 (traffic shedding via cost-policy 429 path) — 14:19 UTC. Reduced burn rate to 4.
- No rollback invoked because the provider recovered its region at 14:34 UTC and the burn rate dropped below the page threshold at 14:38 UTC.

Postmortem skeleton findings:

- Detection was from the SLO burn-rate page, not from a user report; MTTD 3 minutes.
- The runbook did not include a "upstream status page is primary source of truth; check it in the first-five-minutes" step; the team found it at minute 7.
- The cache-hit-ratio drop at 10:04 UTC (feature-store refresh) was the contributing factor that turned a provider degradation into a user-visible burn; a monitoring gap. Action item: add a cache-hit SLI as a new monitoring signal, consider whether it is an SLO.
- Comms template §R3.a worked but needed a factual-tone edit under pressure; action item: pre-draft two variants (tone-neutral and user-apologetic) so the comms lead picks rather than drafts.

Drill artefacts (deliverables of the drill, not of the incident): `runbook.md`, `rollback.md`, `postmortem-template.md`, `drill-2026-11.md` (this walkthrough's log).

### 2.9 Runnable sketch: `drill_check.py`

```python
# drill_check.py
"""Reference drill-artefact validator.

Design goal: a drill is complete when three artefacts exist and conform to
the templates. This script validates the artefacts a drill produced and
flags missing sections.
"""

from dataclasses import dataclass
from pathlib import Path
import re


REQUIRED_RUNBOOK_SECTIONS = [
    "entry map",
    "first-five-minutes",
    "severity table",
    "incident-class playbook",
]

REQUIRED_ROLLBACK_SECTIONS = [
    "model version rollback",
    "prompt",
    "feature pipeline rollback",
    "policy",
    "rto",
    "rpo",
    "non-reversible",
]

REQUIRED_POSTMORTEM_SECTIONS = [
    "metadata",
    "summary",
    "timeline",
    "impact",
    "ml-specific state",
    "detection",
    "response",
    "mitigation and recovery",
    "action items",
    "blameless",
]


@dataclass(frozen=True)
class ValidationResult:
    path: Path
    missing_sections: list[str]
    ok: bool


def validate(path: Path, required_terms: list[str]) -> ValidationResult:
    text = path.read_text(encoding="utf-8").lower()
    missing = [term for term in required_terms if not re.search(re.escape(term), text)]
    return ValidationResult(path=path, missing_sections=missing, ok=not missing)


def validate_drill(drill_dir: Path) -> list[ValidationResult]:
    plan = [
        (drill_dir / "runbook.md",             REQUIRED_RUNBOOK_SECTIONS),
        (drill_dir / "rollback.md",            REQUIRED_ROLLBACK_SECTIONS),
        (drill_dir / "postmortem-template.md", REQUIRED_POSTMORTEM_SECTIONS),
    ]
    return [validate(path, terms) for path, terms in plan]


if __name__ == "__main__":
    import sys
    drill_dir = Path(sys.argv[1]) if len(sys.argv) > 1 else Path(".")
    results = validate_drill(drill_dir)
    exit_code = 0
    for r in results:
        status = "OK" if r.ok else "MISSING"
        print(f"{status}: {r.path}")
        for term in r.missing_sections:
            print(f"    - missing section: {term}")
            exit_code = 1
    sys.exit(exit_code)
```

The script is deliberately a template-conformance check, not a semantic grader. A drill that produces three files passing this script but whose content is nonsense is a drill that failed in a way only a human reviewer catches; the script keeps the structural gap from being invisible.

## 3. Validation steps

Walk the drill outputs through this checklist:

1. **Runbook has an entry map** mapping every alert the surface can emit to a numbered section. No orphan alerts.
2. **First-five-minutes checklist is ≤ 6 steps** and stands alone at the top of the runbook.
3. **Severity table has three levels** (or four if the team's practice uses SEV-4 for internal-only) with triggers and MTTR targets.
4. **Each incident-class playbook has the same shape**: signal, immediate questions, mitigation menu, rollback trigger, comms template, hand-off.
5. **Rollback plan names at least four classes** (model, prompt, feature, policy) with RTO/RPO and preconditions.
6. **Non-reversible changes listed** with forward-fix paths.
7. **Postmortem template has an ML-specific state section** (model versions, dataset version, drift state, safety-filter version) in addition to the generic SRE postmortem shape.
8. **Blameless frame stated** in the template, not implied.
9. **Drill walkthrough has a scenario**, timed phases, and three mandatory artefacts (incident channel, user comms, postmortem skeleton).
10. **Cross-reference table links alerts** to runbook sections and to the surrounding `exercise-01`/`exercise-02`/`exercise-04` artefacts.
11. **Follow-up log records findings** with owner (role), due date, and finding type (preventive / detective / mitigative).
12. **OWASP ML Top 10 and MITRE ATLAS tactics** are named in the safety playbook (§R3.e).
13. **NIST AI RMF GOVERN** is cited in the role/responsibility framing; MANAGE is cited in the mitigation / rollback framing.

Static checks to drop into CI on the drill directory:

```bash
grep -ciE "sev.?1|sev.?2|sev.?3"                    runbook.md
grep -ciE "entry map|first.five.minutes|severity"   runbook.md
grep -ciE "rto|rpo|precondition"                    rollback.md
grep -ciE "model rollback|prompt rollback"          rollback.md
grep -ciE "ml.?specific state|dataset version"     postmortem-template.md
grep -ciE "blameless"                               postmortem-template.md
python drill_check.py .
```

## 4. Rubric / review checklist

- [ ] Drill frame names surface, participant roles, success criteria, cadence, facilitator role, last-run date.
- [ ] Runbook entry map covers every alert the surface emits.
- [ ] First-five-minutes checklist is terse (≤ 6 steps) and includes acknowledge, dashboard-open, change-log check, severity declaration, incident-channel post.
- [ ] Severity table has triggers and MTTR targets per level; SEV-1 explicitly includes safety/policy breach.
- [ ] Each incident-class playbook follows the same shape: signal, immediate questions, mitigation menu (ordered least-to-most invasive), rollback trigger, comms templates (user + internal), hand-off contract.
- [ ] Safety playbook (§R3.e) references OWASP ML Top 10 scenarios (prompt injection, output integrity, model DoS) and MITRE ATLAS tactics explicitly.
- [ ] Rollback plan has ≥ 4 classes (model, prompt, feature, policy) with RTO/RPO/preconditions/executor-role/steps/verification per class.
- [ ] Full-bundle rollback is defined when model/prompt/policy/features change together.
- [ ] Non-reversible changes are listed with forward-fix paths.
- [ ] Postmortem template has: metadata (with MTTD and MTTR computed from timestamps), timeline, impact, ML-specific state, detection, response, mitigation/recovery, action items, lessons-for-SLOs/cost/retrain, blameless frame.
- [ ] Drill walkthrough has a labeled hypothetical scenario, timed phases, role assignments, and three mandatory artefacts (incident channel, user comms, postmortem skeleton).
- [ ] Cross-reference table maps every alert the surface emits to a runbook section and names the downstream artefact (postmortem, SLO re-review, cost-policy update, retrain hardening).
- [ ] Follow-up log has owner (role), due date, finding type per finding; next drill date and class named.
- [ ] NIST AI RMF GOVERN (roles/responsibilities) and MANAGE (mitigation/recovery) cited.
- [ ] Review cadence (quarterly default) named; last-reviewed and last-drilled dates present and within cadence.

Review-weighted points for grading:

- Drill frame and role assignment clarity: 10%
- Runbook (entry map + first-five + severity + playbooks): 25%
- Rollback plan (classes, RTO/RPO, preconditions, non-reversible list): 20%
- Postmortem template (ML-specific state + blameless frame): 15%
- Drill walkthrough (scenario, timed phases, mandatory artefacts): 15%
- Cross-references and follow-up contract: 10%
- Standards alignment (NIST AI RMF GOVERN/MANAGE, OWASP ML Top 10, MITRE ATLAS) and hygiene: 5%

## 5. Common mistakes

- **Generic SRE runbook used for ML surfaces.** A runbook that covers HTTP availability but not drift, retraining failures, or safety-filter breaches is missing half the incidents the surface can have.
- **No pre-registered rollback triggers.** A rollback that waits for a meeting is a rollback that is late. Triggers are named conditions, not judgement calls at a page.
- **Rollback targets that are not retained.** A model version that cannot be rolled back to because the serving infrastructure deleted it last week is not a rollback target. Retention policy is part of the rollback plan.
- **Prompt and model rollbacks coupled.** When a release changes both, the rollback must bundle both; rolling back only the model while the prompt still assumes the new model's capabilities is an incident of its own.
- **Severity by seniority, not by scope.** Declaring SEV-1 because an executive is in the channel, or SEV-3 because "no one has noticed yet," defeats the table's purpose. Severity follows the pre-registered trigger.
- **"Blameless" as a slogan.** A postmortem whose contributing-factors section reads "human error" is not blameless; it is blame rephrased. Blameless means the subject is systems and decisions, and the fix is a system or decision change.
- **MTTD and MTTR not computed.** A postmortem that writes "detection took a while" instead of "MTTD 23 min" cannot be compared to the drill's targets or to the prior incident's numbers.
- **No ML-specific state section.** An ML postmortem that does not record the model version, dataset version, drift state, and safety-filter state at the time of incident is missing exactly the state the fix is likely to involve.
- **Drills not run.** A drill plan with no quarterly dates is a drill plan that will not survive the first calendar year. The cadence is a commitment.
- **Drill without a facilitator.** A team drilling itself cannot measure itself; the gaps a team is used to are invisible to the team. External facilitation is the move.
- **Comms templates written live.** Writing the user-visible status message during the incident costs minutes that MTTR cannot afford. Templates are drafted in advance and personalized at incident time.
- **Action items without owners or dates.** Action items without a role and a date become memorials. The follow-up contract names both.
- **Safety playbook missing OWASP/ATLAS references.** A safety incident whose mitigation menu does not reflect OWASP ML Top 10 scenarios or MITRE ATLAS tactics is missing the vocabulary the broader security review uses.
- **Cross-reference table missing.** Without it, an alert routes to a page; a page routes to a search; a search routes to a wiki page out of date by two sprints. The table is the anchor.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds the role/responsibility structure of incident response (named roles, blameless frame, auditable follow-up). MANAGE function grounds the mitigation and recovery activities (rollback contract, degradation plan) as operational risk controls.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source for the adversarial scenarios the safety playbook (§R3.e) must cover: prompt injection, output integrity failures, model denial of service, data poisoning.
- MITRE ATLAS — <https://atlas.mitre.org/>. Named adversarial tactics (evasion, model extraction, inference-API abuse) that the safety playbook mitigation menu and postmortem adversarial-state capture are built against.

Local context (authoritative for the runbook, rollback, and postmortem shapes, and for the cross-references):

- Paired lesson: `lessons/mod-307-ml-reliability-slos/exercises/exercise-03-ml-incident-response-drill.md`.
- Module README: `modules/mod-307-ml-reliability-slos/README.md`.
- Related exercise (SLOs and burn-rate alerts that drive the runbook entry map): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (cost budgets whose hard-cap breaches route to §R3.d and whose degradation plan supplies the M3 "shed non-essential traffic" mitigation): `modules/mod-307-ml-reliability-slos/exercise-02-cost-budgets-and-quotas/SOLUTION.md`.
- Related exercise (retraining as a deploy — its canary-failure alert routes to §R3.g and its rollback aligns with the model-rollback class): `modules/mod-307-ml-reliability-slos/exercise-04-retraining-as-a-deploy/SOLUTION.md`.
- Related exercise (shadow and canary deploy design — the deploy mechanics that make rollback fast): `modules/mod-305-advanced-evaluation/exercise-02-shadow-and-canary-deploy-design/SOLUTION.md`.
- Related exercise (prompt-as-code — the versioning discipline that makes prompt rollback a config change): `modules/mod-304-production-llm-integration/exercise-02-prompt-as-code/SOLUTION.md`.
