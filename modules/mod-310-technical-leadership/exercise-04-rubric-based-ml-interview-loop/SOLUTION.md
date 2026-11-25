# exercise-04 — Rubric-Based Senior ML Engineer Interview Loop (Reference Solution)

Paired exercise: `lessons/mod-310-technical-leadership/exercises/exercise-04-rubric-based-ml-interview-loop.md`.

## 1. Solution overview

This exercise produces `interview-loop.md` — the full design of a rubric-driven interview loop for hiring a Senior ML Engineer. The artefact answers five questions a hiring manager, a recruiter, and an individual interviewer should read it to get aligned on:

1. **What is the role.** The specific scope of responsibilities the loop is hiring for, named concretely (e.g., "owns a multi-quarter roadmap like `mod-310/exercise-01`; owns the Model Card lifecycle like `mod-309/exercise-01`; sets the design-review bar like `mod-310/exercise-02`; staffs-and-grows engineers like `mod-310/exercise-03`") rather than at title level.
2. **What is the signal set.** The named dimensions the loop evaluates (technical ML depth, system design, applied judgement, communication, leadership / influence, collaboration, inclusion / bias-aware behaviour), the weight of each, the stages they are tested in, and the "evidence" shape the loop accepts for each dimension.
3. **What are the stages.** A five-to-six stage loop (recruiter screen → hiring-manager conversation → technical screen → full on-site with four to five interviews → debrief → decision) with each stage's purpose, the signals it is responsible for surfacing, the rubric it uses, and the time-boxed shape (duration, structure).
4. **How is the bar set and defended.** The rubric that every interviewer scores to (independent-of-vendor "exceeds / meets / below / strong-no-hire" bands with written anchors for each band on each signal), the per-stage sign-off, and the debrief protocol that resolves disagreements against the rubric rather than against the loudest voice.
5. **How is the loop kept fair.** Bias-aware practices (structured questions with pre-written probes; closed-book-vs-open-book choices; accommodation protocol; demographic blinding where allowed and useful), the calibration cadence that keeps interviewer scores consistent across cohorts, and the post-loop evaluation that treats a hire as a measurable outcome that feeds back into the rubric.

The artefact is one document (the loop design) plus three supporting rubric tables (one per primary stage with a scored rubric: technical screen, system design, applied judgement / leadership) and one calibration / debrief protocol.

The reference solution below shows the structure, works a labeled example through it, and gives the grading rubric that calibrates the loop design itself to a consistent bar.

The exercise exists because a hiring loop is the mechanism by which a team's bar is defined, defended, and reproduced. A senior ML engineer's hiring loop is one of the highest-leverage artefacts the senior produces: a single hire can shift team capability for years, and a loop that hires inconsistently either floods the team with people who do not meet the bar or filters out candidates who would. NIST AI RMF GOVERN grounds the loop as an organizational governance instrument: a hiring decision for a role that owns ML-system risk artefacts (Model Card, threat model, SLOs) is itself a governance decision, and the loop that produces that decision should be auditable, reproducible, and reviewable against the same discipline as any other high-stakes decision.

## 2. Worked answer / implementation

### 2.1 Role scope

The loop opens with a one-page definition of what the hire owns. A role defined by title ("Senior ML Engineer") is a role every candidate interprets differently; a role defined by artefacts is a role every candidate can be scored against.

The role is pinned to:

- **Primary surface(s) the hire will own or co-own.** The system, the product, the user population.
- **Artefact-level responsibilities.** The named artefacts the hire is expected to author, own, review, or facilitate. Use the catalog from this module and `mod-309`:
  - Multi-quarter roadmap (`mod-310/exercise-01`).
  - Code / design review bar (`mod-310/exercise-02`).
  - Mentorship of mid-level engineers (`mod-310/exercise-03`).
  - Model Card lifecycle (`mod-309/exercise-01`).
  - Data lineage and PII register ownership (`mod-309/exercise-02`).
  - Fairness / robustness slice cadence (`mod-309/exercise-03`).
  - Misuse / adversarial threat model (`mod-309/exercise-04`).
  - SLO ownership (`mod-307/exercise-01`).
  - Build-vs-adopt and contribute-back decisions (`mod-308/exercises-02-03`).
  - Evaluation harness ownership (`mod-305`).
  - Experiment design and measurement (`mod-306`).
- **Non-responsibilities.** What the hire is explicitly *not* on the hook for. Scope-out matters: a role that silently includes "manage a team" when the loop was designed for IC capability-building will mis-hire and mis-staff.
- **Expected impact horizon.** The timescale over which the hire is expected to produce the senior-level artefacts (typically: first senior-authored roadmap by ~6 months; first Model Card owned end-to-end by ~9 months; first mentee relationship initiated by ~12 months). Impact horizons are the back-end of the rubric: a loop that cannot name when a hire becomes net-positive cannot evaluate whether its prior hires were on-track.

### 2.2 Signal set (dimensions)

Seven dimensions. Each has a short definition, a weight (sum = 100), a list of the stages that test it, and an "evidence shape" that scores well. The dimensions cover the surface the role owns; a loop that tests four dimensions and ships three unmeasured is a loop that hires on partial information.

| # | Dimension | Weight | Tested in | What good looks like |
|---|-----------|--------|-----------|-----------------------|
| 1 | ML depth: correctness, numerical, evaluation, data | 20 | Technical screen; applied-judgement | Catches the silent dtype / shape / numerical bug in the exercise; reasons about offline-vs-online metric divergence; anticipates eval-slice gaps; reasons about training-serving skew. |
| 2 | System design: ML-specific topology end-to-end | 20 | System design | Proposes a working training / serving / evaluation / monitoring topology; names dependencies, failure modes, rollout, measurement; integrates governance artefacts (Model Card, lineage); argues capacity. |
| 3 | Applied judgement: trade-offs, risk, scoping | 15 | Applied judgement; hiring-manager | Identifies the one or two bets a design or a plan is making; names the non-goals; chooses the right specific metric for a stated harm model; proposes rollback and measurement. |
| 4 | Communication: writing, framing, listening | 10 | Every stage | Writes clearly in the structured response; frames the problem before the solution; listens to clarifying questions and adjusts. |
| 5 | Leadership / influence: authoring and reviewing at bar | 15 | Applied judgement; leadership | Describes real work where the candidate raised the review bar, authored a cross-team artefact, mentored someone through a transition, or made a decision stick in a room of skeptics. |
| 6 | Collaboration: cross-team, cross-function | 10 | Hiring-manager; leadership | Candidate's examples show working with product owners, platform teams, governance / security, with names, with specifics; not "I worked with X team." |
| 7 | Inclusion / bias-aware behaviour | 10 | Every stage; one dedicated probe | Candidate reasons about fairness slices, under-represented populations, consent, and dataset provenance when the problem touches it; candidate signals inclusive behaviour in interactions during the loop itself. |

Weights add to 100. Weights are set *before* interviews, not back-fit to a candidate. If a team wants to re-weight for a particular role (e.g., first-hire-on-surface may weight system design higher than mentorship), the re-weighting is documented and signed by the hiring manager before the loop begins, not during debrief.

An "evidence shape" per dimension is critical. The loop does not score "communication: strong"; it scores "communication: when asked to frame the problem, candidate wrote a clear one-paragraph framing with a stated outcome and a named non-goal within 90 seconds."

### 2.3 Loop stages

A six-stage loop. Each stage has a purpose, a duration, a structured rubric, a decision outcome, and a sign-off.

**Stage 1. Recruiter screen (30 min).**

- Purpose: confirm basic fit, compensation alignment, logistics, and that the role scope matches what the candidate is looking for. Not an evaluation of technical skills.
- Structure: recruiter-led conversation against a one-page role summary.
- Decision: proceed / not-proceed / pause-for-info. Recruiter documents the decision in the ATS.
- Sign-off: recruiter.

**Stage 2. Hiring-manager conversation (45 min).**

- Purpose: evaluate whether the candidate's scope of past work and interests match the role's artefact-level responsibilities and expected impact horizon. The HM also surfaces leadership and collaboration signal in a semi-structured way.
- Structure: 10 min HM introduces the role (not the company; the role); 25 min HM-led structured conversation on three questions pre-written for this role; 10 min candidate questions.
- Pre-written question shapes:
  - "Tell me about a time you owned a multi-quarter plan for an ML initiative. What was the thesis, how did you scope the milestones, where did the plan survive contact with reality and where did it not?" — tests dimension 2 (system design, loosely), dimension 3 (applied judgement), dimension 5 (leadership).
  - "Tell me about a review where you held a bar that was hard to hold. Code review or design review." — tests dimension 5 (leadership) and dimension 7 (inclusion if the review bar involved under-represented populations or governance artefacts).
  - "Walk me through a time you worked with a platform or governance team to land something that benefited both your surface and the paved road." — tests dimension 6 (collaboration).
- Decision: proceed / not-proceed. HM documents signal against rubric dimensions 3 / 5 / 6 within 24 hours of the conversation.
- Sign-off: hiring manager.

**Stage 3. Technical screen (60 min).**

- Purpose: test dimension 1 (ML depth). This is a working session on a piece of ML code, offline eval, or data-correctness problem. Not a leetcode puzzle; not a from-scratch model implementation.
- Structure: 10 min briefing; 40 min working session (shared screen / IDE / repl / notebook on a provided problem); 10 min debrief where the interviewer asks the candidate to narrate their decisions.
- Problem shape: a ~50-line snippet of ML code with 2–3 latent bugs (dtype, shape, boundary, numerical stability, training-serving skew) and an offline-eval harness that is subtly mis-scoring. Candidate is asked to run, reason, fix, and justify. The problem should be set at the level of senior ML engineering, not research mastery.
- Decision: strong-hire / hire / lean-hire / lean-no-hire / no-hire / strong-no-hire. Rubric anchored (see §2.4 below).
- Sign-off: technical interviewer; must be IC at or above the hiring level.

**Stage 4. System design (75 min).**

- Purpose: test dimension 2 (system design) and parts of dimension 3 (applied judgement). The interviewer presents an open-ended design brief; the candidate walks through a full ML system end-to-end.
- Structure: 10 min problem briefing with candidate asking clarifying questions; 55 min candidate drives the design on a shared whiteboard / diagram tool; 10 min debrief and interviewer-led probing on neglected areas.
- Problem-brief shape: a specific ML surface in a plausible business context — e.g., "design a drift-monitoring layer for an LLM-backed support-assistant surface," "design an experimentation platform for feature-launch A/B for a classical-ML recommender," "design an offline evaluation harness with fairness slices for a multi-language assistant." The problem is open enough that the candidate makes trade-offs; narrow enough that the interviewer can score the same problem across multiple candidates.
- The interviewer tracks: thesis-and-outcome framing; training / serving / eval / monitoring / governance coverage; dependency graph; capacity envelope; rollout mechanism; risk register; bets and assumptions; measurement plan; non-goals. All the way down the design-review rubric from `mod-310/exercise-02`.
- Decision: six-band rubric score.
- Sign-off: system-design interviewer; at or above hiring level; different person from stage 3.

**Stage 5. Applied judgement / leadership (60 min).**

- Purpose: test dimension 3 (applied judgement) and dimension 5 (leadership / influence). This stage is a structured behavioural interview against three question families, each with written probes that go three levels deep when the candidate's answer is light.
- Structure: three 15-minute question families; 15 min for candidate questions.
- Question families:
  - "Tell me about a time an ML launch went wrong on your watch and how you ran the response." Probes: what was the SLO breach; how did you page / escalate; what was the rollback; what did the post-mortem produce as action items; which of those action items landed and which did not; how did you decide what to prioritise.
  - "Tell me about a time you made an unpopular technical call and defended it." Probes: what was the call; why unpopular; how did you frame the decision; how did you handle disagreement; how did it resolve; what would you do differently.
  - "Tell me about a time you grew another engineer." Probes: what was the growth target; what coaching tactic did you use; what evidence did you look for; what did you do when it was not working; what was the outcome.
- Decision: six-band rubric score on dimensions 3 and 5.
- Sign-off: applied-judgement interviewer; at or above hiring level.

**Stage 6. Inclusion / collaboration cross-check (45 min).**

- Purpose: test dimensions 6 (collaboration) and 7 (inclusion / bias-aware behaviour), both of which are also tested across the whole loop but which get a dedicated probe here. This stage can be combined with a cross-team interviewer (e.g., `team-governance` or `team-platform` senior IC) and intentionally looks at bias-aware practice when the problem surface touches populations.
- Structure: a specific scenario where the candidate reasons about a situation involving a fairness slice (from `mod-309/exercise-03`), an under-represented language or demographic, or a consent boundary (from `mod-309/exercise-02`). 15 min candidate reasons through the problem; 15 min follow-up on collaboration ("walk me through how you'd collaborate with legal / governance / product / platform to land this"); 15 min for candidate questions and interviewer note-taking.
- Decision: six-band rubric score on dimensions 6 and 7.
- Sign-off: cross-team interviewer.

**Debrief and decision.**

- Timing: within 48 hours of the last stage. Debrief is structured, not free-form.
- Order: each interviewer first commits their written rubric scores to the ATS *before* the debrief meeting; the debrief does not start until all written feedback is submitted. This prevents the "I'll just say what the HM said" anchoring effect.
- In the meeting: each interviewer presents evidence against rubric anchors for their stage (90 seconds each), then the group discusses the dimensions where opinions diverge (which are flagged by the written scores). Debrief resolves disagreements against the rubric anchors, not against the loudest voice.
- Decision rule: hire requires strong-hire or hire on dimension 1 (ML depth), hire-or-above on dimension 2 (system design), hire-or-above on dimensions 3 and 5, no "below the bar" on dimensions 6 and 7. The rule is pre-stated; deviating from it on a candidate requires a written justification signed by the HM.
- Outcome options: hire; hire-with-notes (hire but with growth areas identified for the mentorship plan from `mod-310/exercise-03`); no-hire; no-hire-with-reroute (candidate may be a fit for a different role; recruiter follows up).
- Sign-off: hiring manager; with the loop lead (typically the senior ML engineer who owns the surface) as co-signer.

### 2.4 Rubric anchors (per dimension, six bands)

The rubric is written as a six-band scale with concrete anchors for each band on each dimension. Anchors are what make the rubric calibratable; a rubric that says "strong" without a what-makes-it-strong anchor is a rubric that reproduces each interviewer's private bar.

Example anchors for dimension 1 (ML depth) in the technical screen:

- **Strong hire.** Identifies the primary bug in < 10 minutes; names at least one additional subtle correctness issue the problem did not intend (e.g., "this metric is implemented symmetrically but shouldn't be for this task"); proposes a test that would catch the bug and ships it; narrates the thought process in a way that would let another engineer learn.
- **Hire.** Identifies the primary bug within the session; proposes a reasonable fix; tests the fix; narrates the reasoning. May need one hint to find the subtler bug.
- **Lean hire.** Identifies the primary bug with one or two targeted hints; fix is correct; tests present but not comprehensive; narration is adequate.
- **Lean no-hire.** Finds a symptom but not the root cause; fix is a workaround; tests are thin; narration is uncertain.
- **No hire.** Does not identify the root cause even with strong hints; does not reason about training-serving skew; silently accepts the buggy output when it is anomalous.
- **Strong no-hire.** Does not notice the bug; does not reason about the correctness of a metric; demonstrates misconceptions (e.g., confuses accuracy with precision, uses float32 for an index tensor, applies a normalization with the serving mean but training std).

Example anchors for dimension 2 (system design):

- **Strong hire.** Covers training, serving, evaluation, monitoring, governance (Model Card, lineage), rollout, measurement; names dependency graph, named capacity envelopes, specific SLOs; names two or more realistic alternatives and why not; identifies at least one bet the design is making and how it would be de-risked.
- **Hire.** Covers training / serving / eval / monitoring with specifics; names a rollout plan; names a measurement plan; references governance artefacts; identifies at least one bet.
- **Lean hire.** Covers the big-picture topology; some specifics missing; rollout is described but rollback is light; mentions governance but does not integrate it.
- **Lean no-hire.** Covers architecture but not evaluation; no monitoring; no rollout plan; no mention of risk.
- **No hire.** Architecture only; missing eval, monitoring, governance; no rollout or measurement; no mention of alternatives.
- **Strong no-hire.** Design would not work at claimed scale; would not integrate with a paved road; or demonstrates misunderstanding of ML-system basics (e.g., no training-serving separation, no distinction between offline eval and online measurement).

Example anchors for dimension 5 (leadership / influence):

- **Strong hire.** Stories include specific artefacts (named roadmaps, Model Cards, RFCs), specific collaborators (named teams or roles), specific decisions and their outcomes; candidate reflects on what they would do differently; candidate's role in the outcome is clear.
- **Hire.** Stories are specific with artefacts and outcomes; candidate is clearly the owner or key contributor; reflection is present.
- **Lean hire.** Stories are specific but candidate's role is slightly abstracted; outcomes are described but not quantified; reflection is light.
- **Lean no-hire.** Stories are present but generic; outcomes unclear; candidate's role described in passive voice ("we decided to," "the team concluded").
- **No hire.** Stories are abstract; no named artefacts; candidate cannot answer probing questions about specifics.
- **Strong no-hire.** No stories available; candidate attributes all outcomes to others or to circumstances; no evidence of leadership behaviour.

Analogous anchors are written for dimensions 3, 4, 6, 7. The full artefact includes a per-dimension-per-band table so every interviewer scores to the same written standard.

### 2.5 Bias-aware practice

A loop that does not actively defend against bias will reproduce the biases of its interviewers by default. Named practices:

- **Structured questions with pre-written probes.** Every behavioural stage has pre-written question families and probes. Interviewers do not improvise. Improvisation on top of a structured question is allowed, but the base questions are the same across candidates for a role.
- **Written rubric scores before debrief.** Each interviewer commits their score and evidence before the debrief, so the group discussion is informed by independent evidence rather than anchored by the first voice.
- **Rubric anchors, not interviewer gut.** Each band has a written anchor; interviewer feedback cites the anchor. "Hire because strong communicator" is not acceptable feedback; "hire because in the HM stage, when asked to frame the roadmap problem, candidate wrote a one-paragraph framing with a named outcome, a named non-goal, and a named bet within 90 seconds — anchor 'hire' under dimension 4" is.
- **Inclusion probe in every behavioural stage.** Every behavioural stage includes at least one probe that gives the candidate the opportunity to show bias-aware behaviour (dimension 7). A loop where only the "diversity stage" asks about inclusion is a loop where inclusion is performative.
- **Accommodation protocol.** A written accommodation protocol is sent to every candidate before the loop. Common accommodations (break between stages, longer response time, written rather than verbal format for a stage, note-taking pre-prepared, assistive tech) are offered proactively. Accommodation does not change the rubric; it changes the format by which the rubric is applied.
- **Interviewer training cadence.** Interviewers are trained on the rubric before they participate in loops, and re-trained annually. Training includes bias-awareness, calibration against anchored examples, and shadowing-before-leading.
- **Demographic blinding where allowed and useful.** Where legally permissible and operationally tractable (early stages like resume screen), demographic information is blinded. Later stages are, by nature, not blind; the practice shifts to rubric discipline and interviewer diversity.
- **Interviewer-panel diversity.** The loop panel is as diverse as the hiring team can make it. A loop that is uniformly composed in demographic terms is a loop whose bar will reflect that uniformity.
- **Post-loop evaluation.** The loop's outcomes are treated as a signal that is itself measured. Hires' six-month, 12-month, and 24-month performance (against the artefact-level responsibilities from §2.1) feeds back to a yearly calibration review of the rubric: do the "strong hire" scores actually predict hires who hit the artefact milestones? If not, the rubric anchors are revised.

### 2.6 Calibration cadence

- **Per-loop calibration.** Before each loop, the panel reviews the role scope and rubric once (10 minutes). Any role-specific re-weights (per §2.2) are confirmed.
- **Quarterly calibration meeting.** Interviewers across the hiring team meet to review the quarter's loops: outcomes, score patterns, dimensions where interviewers disagreed, any anchors that need re-writing. One hour.
- **Annual rubric review.** Hiring managers across the organization review the rubric against the post-hire data: did hires with strong-hire scores on dimension 1 actually catch the production bug? Did hires with strong-hire scores on dimension 2 actually author designs that passed design-review on the first iteration? Where the predictive validity is weak, the anchors are revised.
- **New-interviewer onboarding.** A new interviewer shadows at least three loops, is debriefed by the loop lead, and leads a loop stage only after a pre-loop training on the rubric. First three loops they lead are co-scored by an experienced interviewer.

### 2.7 Illustrative worked example (partial)

> **Illustrative role** — a Senior ML Engineer hire for the `agent-assist/draft` surface described in the sibling exercises. This example is a composite for calibration.

```markdown
# Interview Loop: Senior ML Engineer — `agent-assist/draft` surface

## §1. Role scope
Primary surface: the `agent-assist/draft` LLM-backed support-agent assistant
on billing / shipping / account-mgmt queues, EN-US / PT-BR / ES-MX.
Artefact-level responsibilities:
- Author and own the multi-quarter roadmap for the surface
  (`mod-310/exercise-01`).
- Own the Model Card lifecycle for `assist-llm-v*` (`mod-309/exercise-01`).
- Set and defend the design-review and code-review bar for the surface
  (`mod-310/exercise-02`).
- Mentor one mid-level ML engineer (`mod-310/exercise-03`).
- Co-own (with `team-platform-eval`) the offline eval harness
  (`mod-305/exercise-01`) and the per-feature drift-monitoring
  (`mod-307/exercise-01`).
- Author the fairness-slice cadence (`mod-309/exercise-03`) and update
  the threat model (`mod-309/exercise-04`).
Non-responsibilities:
- Not a people-manager role; no direct reports.
- Not the sole on-call IC — participates in a rotation.
- Not the owner of the base-model training (that is `team-model-base`).
Expected impact horizon:
- First senior-authored roadmap: ~6 months.
- First Model Card owned end-to-end: ~9 months.
- First mentee relationship initiated: ~12 months.

## §2. Signal set (per 2.2 table)
Dimensions 1 (20) / 2 (20) / 3 (15) / 4 (10) / 5 (15) / 6 (10) / 7 (10) = 100.
No role-specific re-weights for this req.

## §3. Loop stages
Stage 1: recruiter screen (30 min).
Stage 2: hiring-manager conversation (45 min).
Stage 3: technical screen on an LLM-eval harness snippet with two
training-serving-skew bugs and one subtly mis-scoring metric (60 min).
Stage 4: system design — "design a fairness-slice evaluation harness
for a multi-language assistant with a monthly cadence" (75 min).
Stage 5: applied judgement / leadership — the three behavioural
question families from §2.3 Stage 5 (60 min).
Stage 6: inclusion / collaboration — scenario on PT-BR slice gap
(45 min), run with a `team-governance` senior as co-interviewer.
Debrief within 48 hours.

## §4. Rubric anchors
(Full table in-repo; §2.4 excerpt above shows dimensions 1 / 2 / 5
anchors.)

## §5. Bias-aware practice
(All practices from §2.5. Accommodation protocol in
`accommodations.md`. Loop panel: 5 interviewers; recruiter blinds
demographic info through Stage 1 and Stage 2.)

## §6. Calibration cadence
(Per 2.6. This loop's panel reviewed the rubric on 2026-12-10;
next quarterly calibration 2027-Q1 end.)
```

### 2.8 Lightweight loop linter (static checks)

```python
# interview_loop_lint.py
"""Lint an interview-loop design for structural completeness.

Enforces: role scope by artefact; signal set with weights summing to 100;
all seven dimensions covered; six-band rubric; bias-aware practices;
calibration cadence; debrief protocol.
"""

import re
import sys


REQUIRED_SECTIONS = [
    "role scope", "signal set", "loop stages", "rubric anchors",
    "bias-aware", "calibration cadence",
]

REQUIRED_DIMENSIONS = [
    "ml depth", "system design", "applied judgement",
    "communication", "leadership", "collaboration", "inclusion",
]

RUBRIC_BANDS = [
    "strong hire", "lean hire", "no hire", "strong no-hire",
]


def lint(text: str) -> list[str]:
    errors: list[str] = []
    lowered = text.lower()
    for s in REQUIRED_SECTIONS:
        if s.lower() not in lowered:
            errors.append(f"missing section: {s}")
    for d in REQUIRED_DIMENSIONS:
        if d not in lowered:
            errors.append(f"dimension not covered: {d}")
    for b in RUBRIC_BANDS:
        if b not in lowered:
            errors.append(f"rubric band missing: {b}")
    # weights sum to 100
    weights = [int(m) for m in re.findall(r"\|\s*(\d{1,3})\s*\|", text) if 0 < int(m) <= 100]
    if weights and sum(weights[:7]) != 100:
        errors.append(f"dimension weights do not sum to 100 (got {sum(weights[:7])})")
    if "accommodat" not in lowered:
        errors.append("no accommodation protocol referenced")
    if "debrief" not in lowered:
        errors.append("no debrief protocol described")
    if not re.search(r"written.*before|before.*debrief", lowered):
        errors.append("debrief protocol does not require written scores "
                      "before the meeting")
    if "post-loop" not in lowered and "post-hire" not in lowered:
        errors.append("no post-hire evaluation feeding back to rubric")
    return errors


if __name__ == "__main__":
    errs = lint(open(sys.argv[1]).read())
    for e in errs:
        print(f"interview-loop-lint error: {e}")
    sys.exit(1 if errs else 0)
```

## 3. Validation steps

Walk the loop design through this checklist before running it on a candidate:

1. **§1 names the role by artefacts, not by title.** A loop whose role scope is one sentence ("Senior ML Engineer, strong candidates") will produce inconsistent evaluation across the panel.
2. **§1 names non-responsibilities.** Scope-out defends against mis-hiring and mis-staffing.
3. **§2 dimensions cover the surface the role owns.** Seven dimensions is the baseline; a loop that tests three is a loop that will miss two.
4. **§2 weights sum to 100 and are set before the loop.** Back-fitting weights after a candidate is a bar-re-draw.
5. **§2 has an evidence shape for each dimension.** "Good at X" is not an evidence shape; "when asked to do Y, candidate produced Z within W minutes" is.
6. **§3 has at least five stages** with named purposes, durations, structures, decision outcomes, and sign-offs.
7. **§3 stages cover every dimension in §2** (and vice versa).
8. **§3 debrief requires written scores before the meeting.** Anchoring is defended against.
9. **§4 rubric has six bands with written anchors** per band per dimension. Anchors cite concrete evidence shapes, not adjectives.
10. **§4 decision rule is pre-stated.** The hire / no-hire mapping from scored dimensions is written down before the loop.
11. **§5 bias-aware practices are named** — structured questions, written probes, rubric anchors, inclusion probes, accommodation protocol, interviewer training, demographic blinding where allowed, interviewer-panel diversity, post-loop evaluation.
12. **§5 accommodation protocol is sent to candidates proactively.**
13. **§6 calibration cadence is named** — per-loop, quarterly, annual.
14. **Post-hire evaluation feeds back to rubric.** Hires' six-month / 12-month / 24-month outcomes against the §1 artefacts refine the anchors annually.
15. **NIST AI RMF is cited.** The loop is treated as a governance instrument, not a soft HR process.
16. **Cross-references to the senior-level artefacts the role owns** (`mod-310/exercises-01-02-03`; `mod-309`; `mod-308`; `mod-307`; `mod-305`; `mod-306`).

Static checks a CI step can enforce:

```bash
grep -ciE "role scope|non-?responsibilit"               interview-loop.md
grep -ciE "signal set|dimension|weight"                 interview-loop.md
grep -ciE "ml depth|system design|leadership|inclusion" interview-loop.md
grep -ciE "strong hire|strong no-hire|anchor"           interview-loop.md
grep -ciE "accommodat|calibration|debrief"              interview-loop.md
grep -ciE "nist ai rmf|govern"                          interview-loop.md
```

`interview_loop_lint.py` from §2.8 gives a programmatic version.

## 4. Rubric / review checklist

- [ ] §1 pins the role to artefact-level responsibilities (not title) and names non-responsibilities.
- [ ] §1 states an expected impact horizon with dated milestones.
- [ ] §2 lists seven dimensions covering ML depth, system design, applied judgement, communication, leadership, collaboration, inclusion.
- [ ] §2 weights sum to 100, were set before the loop, and any role-specific re-weights are documented with HM sign.
- [ ] §2 every dimension has a written evidence shape.
- [ ] §3 has at least five stages, each with purpose, duration, structure, decision outcome, sign-off.
- [ ] §3 each dimension in §2 maps to a stage in §3 and every stage is assigned at least one dimension.
- [ ] §3 debrief requires written scores committed before the meeting.
- [ ] §3 decision rule is pre-stated and deviations require written HM justification.
- [ ] §4 rubric has six bands with written anchors per band per dimension; anchors cite concrete evidence shapes.
- [ ] §5 names structured questions with pre-written probes; rubric anchors cited in feedback; inclusion probe in every behavioural stage.
- [ ] §5 accommodation protocol is proactively shared with candidates.
- [ ] §5 interviewer training and interviewer-panel diversity named.
- [ ] §5 post-hire evaluation feeds back to annual rubric review.
- [ ] §6 calibration cadence (per-loop, quarterly, annual) is named with owners.
- [ ] Cross-references to the sibling senior-bar artefact exercises (`mod-310/exercises-01-02-03`), governance exercises (`mod-309`), platform-collaboration exercises (`mod-308`), SLO (`mod-307`), eval (`mod-305`), experimentation (`mod-306`).
- [ ] NIST AI RMF is cited as the governance frame.

Review-weighted points for grading:

- §1 role scope anchored in artefacts, not titles: 10%
- §2 signal set clarity, weights, evidence shapes: 15%
- §3 stage design (purpose, duration, structure, dimensions covered): 20%
- §3 debrief protocol discipline (written scores before meeting, pre-stated decision rule): 10%
- §4 rubric anchors (six bands, concrete evidence shapes): 20%
- §5 bias-aware practices and inclusion probes: 15%
- §5 accommodation protocol and interviewer training: 5%
- §6 calibration cadence and post-hire feedback loop: 5%

## 5. Common mistakes

- **Role defined by title.** "Senior ML Engineer" is ambiguous. Pin the role to the artefacts it owns.
- **Rubric adjectives.** "Strong technical" is not a rubric; a six-band scale with written anchors per band per dimension is.
- **Weights set during debrief.** Back-fitting weights to a candidate is a bar-re-draw; the panel decides who they wanted to hire and re-weights to make the rubric say that. Weights are frozen before the loop.
- **Fewer than seven dimensions.** A loop that tests only ML depth and system design hires people who are strong at the technical surface but weak at the leadership surface the role requires.
- **Anchoring in the debrief.** The first voice sets the bar; others anchor to it. The fix is written scores before the meeting.
- **Decision rule invented in the moment.** A rule like "we hire if two interviewers say strong-hire" that is invented in the meeting is not a rule; it is a rationalization. Pre-state the rule.
- **Behavioural stage without probes.** "Tell me about a hard decision" without written probes becomes "whatever the candidate wants to tell me about." Probes go three levels deep so every candidate reaches the same depth of signal.
- **Inclusion stage only in the inclusion stage.** Bias-aware signal should be observable across every behavioural interaction; dedicating it to one stage signals it is a compliance line item.
- **No accommodation protocol.** The candidate who needs accommodation but is afraid to ask performs worse than they otherwise would, and the loop mis-scores them. The protocol is proactive.
- **No interviewer training.** Untrained interviewers reproduce their taste; trained ones apply the rubric. First-time interviewers shadow at least three loops before leading.
- **Panel uniformity.** A panel composed uniformly in demographic terms will hire for familiarity. The loop's calibration cadence must notice this; the practice of diversifying the panel is an ongoing action, not a one-time fix.
- **No post-hire evaluation.** A loop that does not measure its outcomes is a loop that cannot improve. Hires' six-month / 12-month / 24-month evidence against the §1 artefacts is the loop's own feedback signal.
- **Technical screen as leetcode.** Senior ML engineering is not solving a linked-list puzzle; it is spotting the training-serving skew in a real snippet, reasoning about offline-vs-online metrics, and designing an eval harness that has the right slices. Use ML-shaped problems.
- **System design as whiteboard puzzle.** A system-design stage that rewards the candidate who can recite a textbook topology produces the same hires that a textbook can describe. Ask for the trade-offs, the dependency graph, the measurement plan.
- **Candidate questions treated as fluff.** The quality of candidate questions is itself a signal on dimension 4 and dimension 6. Reserve genuine time for it.
- **Loop runs over time without calibration.** Rubric drift is slow and silent. Without a quarterly calibration meeting, the "hire" bar in Q4 is not the same as the "hire" bar in Q1.
- **No cross-reference to artefact standards.** A loop that hires for owning a Model Card without citing `mod-309/exercise-01` is a loop whose interviewers do not share a definition of what Model Card ownership is.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds the interview loop as the organizational governance instrument that produces hires who will own ML risk artefacts (Model Card, lineage, threat model, SLOs); treating the loop with the same discipline as any other high-stakes decision (auditable, reproducible, reviewable, measurable) is the GOVERN-aligned practice.
- NIST AI 600-1 (Generative AI Profile) — <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>. Referenced when the role scope includes generative-surface Model Cards; interviewer probes on dimension 7 may reference the GAI risk taxonomy (harmful bias, information integrity, inclusion impacts) when the role is for a generative surface.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Referenced on dimension 1 (ML depth) and dimension 2 (system design) stages when the candidate is asked to reason about ML-specific threat classes; appears as a probe shape in the system-design brief when the role owns a threat model.
- MITRE ATLAS — <https://atlas.mitre.org/>. Referenced on dimension 2 (system design) for generative-surface design briefs and on dimension 3 (applied judgement) for threat-model reasoning questions.

Practitioner implementation examples:

- VeriSwarm engineering blog — <https://veriswarm.ai>. Shape-of-artefact reference for team-level hiring-loop write-ups; not a standard.

Local context (authoritative for the artefact shape and cross-references):

- Paired lesson: `lessons/mod-310-technical-leadership/exercises/exercise-04-rubric-based-ml-interview-loop.md`.
- Module README: `modules/mod-310-technical-leadership/README.md`.
- Sibling exercise (roadmap authoring that the role owns and that §1 cites): `modules/mod-310-technical-leadership/exercise-01-roadmap-and-scoping/SOLUTION.md`.
- Sibling exercise (code / design review bar that the role defends and that dimension 2 / 5 test against): `modules/mod-310-technical-leadership/exercise-02-code-review-and-design-review-at-bar/SOLUTION.md`.
- Sibling exercise (mentorship plan the hire would initiate within 12 months, per §1 impact horizon): `modules/mod-310-technical-leadership/exercise-03-mentorship-plan-for-a-mid-level/SOLUTION.md`.
- Related exercise (Model Card authorship the role owns): `modules/mod-309-responsible-ai-governance/exercise-01-model-card-authoring/SOLUTION.md`.
- Related exercise (data lineage / PII register the role owns): `modules/mod-309-responsible-ai-governance/exercise-02-data-lineage-and-pii-audit/SOLUTION.md`.
- Related exercise (fairness slice cadence the role authors): `modules/mod-309-responsible-ai-governance/exercise-03-fairness-and-robustness-slice-cadence/SOLUTION.md`.
- Related exercise (threat model the role owns): `modules/mod-309-responsible-ai-governance/exercise-04-misuse-and-adversarial-threat-model/SOLUTION.md`.
- Related exercise (SLO ownership): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (paved-road audit the role may initiate): `modules/mod-308-platform-collaboration/exercise-01-paved-road-consumption-audit/SOLUTION.md`.
- Related exercise (build-vs-adopt doc the role may author): `modules/mod-308-platform-collaboration/exercise-02-build-vs-adopt-tradeoff-doc/SOLUTION.md`.
- Related exercise (contribute-back RFC the role may author): `modules/mod-308-platform-collaboration/exercise-03-contribute-back-rfc-to-platform/SOLUTION.md`.
- Related exercise (evaluation harness co-ownership): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (experiment design the role may author): `modules/mod-306-experimentation-at-scale/exercise-01-power-mde-and-ramp-plan/SOLUTION.md`.
