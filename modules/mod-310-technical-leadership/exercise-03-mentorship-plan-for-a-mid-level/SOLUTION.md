# exercise-03 — Mentorship Plan For A Mid-Level Engineer (Reference Solution)

Paired exercise: `lessons/mod-310-technical-leadership/exercises/exercise-03-mentorship-plan-for-a-mid-level.md`.

## 1. Solution overview

This exercise produces `mentorship-plan.md` — a two- to four-quarter written plan a senior ML engineer co-authors with a mid-level engineer who is working toward senior-level responsibilities. The plan is a joint artefact. The mentor drafts it; the mentee marks it up and co-signs it; the mentee's manager reviews it. The plan is reviewed on a monthly cadence and re-signed at each quarter boundary.

The plan is not a performance-management document and is not a substitute for one. It is a *development* document that names:

1. **Where the mentee is today.** A specific, evidenced read of the mentee's current scope, strengths, and gaps against the senior-level expectations the team uses. The read is anchored in observable artefacts (PRs the mentee authored, designs the mentee led, incidents the mentee ran) rather than feelings.
2. **Where the mentee is going.** The senior-level responsibilities the mentee is building toward, named concretely (e.g., "owning a multi-quarter roadmap like `mod-310/exercise-01`," "setting the design-review bar like `mod-310/exercise-02`," "owning the governance artefact lifecycle like `mod-309/exercise-01`"), not as abstract traits.
3. **The specific growth areas for the next two to four quarters.** Three to five areas, each with a measurable objective, the artefacts that will produce evidence of growth, the mentor's coaching tactic, and the sustainment plan after the objective is hit.
4. **The mechanisms.** How the mentor and mentee work — the 1:1 cadence, the coaching touchpoints (pair-programming sessions, design-doc co-authoring, review-shadowing), the stretch assignments, the feedback-in-both-directions norms, the escalation paths.
5. **The review cadence.** When the plan is re-reviewed, what triggers an out-of-cycle re-plan, and the exit criteria that mark "this mentee has grown into the next level of responsibility for the areas in scope."

The plan is the mentee's document. The mentor is the drafter and the first reviewer; the mentee owns the execution; the manager is informed and provides calibration against the organization's leveling expectations.

The reference solution below shows the structure, works a labeled illustrative plan through it, and gives the grading rubric that calibrates the plan to a consistent bar across different mentors in the same team.

The exercise exists because senior-level responsibility is not conferred by title; it is grown through named, observed, evidence-rich practice. A mentorship plan that lives only in the mentor's head is a plan that reaches different endings depending on which mentor the mentee had; a plan written down is a plan the team can calibrate and the mentee can execute against. NIST AI RMF GOVERN grounds the plan as the governance instrument that makes team capability-building legible and auditable; the mentee's development into a senior role is itself a governance outcome — a senior ML engineer owns risk artefacts (Model Card, threat model, roadmap), and the mentorship plan documents how that ownership is transferred.

## 2. Worked answer / implementation

### 2.1 Mentorship-plan structure

The plan has eight required sections. Each section is short, evidence-anchored, and dated. "Keep up the great work" and "develop leadership" are plan smells and are rejected on review.

**§1. Parties and scope.**

- **Mentee.** Named by role (`team-agent-assist` mid-level ML engineer), with current level (e.g., `L25`) and target level or scope (`L30` senior ML engineer, specifically against the four Senior ML Engineer modules' expectations: leadership, governance, SLO/reliability, platform-collaboration).
- **Mentor.** Named by role (`team-agent-assist` senior ML engineer), with a one-line "why this match" (shared surface, complementary gap-coverage, cross-team exposure). If the mentee needs capability the mentor cannot provide, a second mentor is named for the gap — one mentor is a bottleneck.
- **Manager.** The mentee's manager, who is informed, provides leveling calibration, and acts as the escalation path if the mentor-mentee relationship needs adjusting. The manager is not a signatory on growth decisions; the mentor and mentee are, with the manager's awareness.
- **Scope of mentorship.** What is in-scope ("coaching on roadmap authorship, design-review facilitation, Model Card ownership, cross-team RFC drafting") and what is out-of-scope ("performance-management conversations, compensation discussions, pager-rotation adjustments"). Out-of-scope is load-bearing: a plan that silently absorbs performance management undermines the trust that mentorship depends on.

**§2. Current read.**

The mentee's current state in evidence form. Each read is backed by three to five specific artefacts (PRs, design docs, meeting notes, incident retros, model-card entries) that the mentor and mentee both agree are representative. Prose without evidence is a vibe.

- **Scope today.** The surfaces, systems, and projects the mentee currently owns, with a one-line "ownership shape" per each (sole owner, co-owner, contributor, consulted).
- **Strengths.** Three to five specific strengths, each anchored to an artefact. "Debugging numerical bugs — identified the dtype-promotion regression in `training/make_features.py#L144` in 2026-Q3 that would have hit prod" is a strength. "Strong debugger" is not.
- **Gaps against the senior bar.** Three to five specific gaps the mentee and mentor agree on. "Has not yet authored a multi-quarter roadmap (per `mod-310/exercise-01`); has co-authored three quarterly plans but has not been the thesis-owner" is a gap. "Needs to grow into leadership" is not.
- **Calibration.** A one-paragraph note from the manager confirming that the strengths/gaps read matches the leveling expectations the organization uses. If the manager disagrees, the disagreement is noted and resolved before the plan is signed — a plan the manager disagrees with will not be supported.

**§3. Target responsibilities.**

The responsibilities the mentee is building toward, at a concrete artefact level. The target is not a title; the target is the set of things the mentee can lead, author, review, or own end-to-end at senior level. Each target is one of the following shapes:

- **Author.** The mentee produces an artefact at senior-bar quality. Examples: a multi-quarter roadmap (`mod-310/exercise-01`); a Model Card with full sign-off (`mod-309/exercise-01`); a build-vs-adopt doc (`mod-308/exercise-02`); a contribute-back RFC (`mod-308/exercise-03`); an SLO doc (`mod-307/exercise-01`); a threat model (`mod-309/exercise-04`).
- **Facilitate.** The mentee runs a recurring process at senior-bar quality. Examples: chairs the monthly design-review for the surface; runs the fairness-slice cadence (`mod-309/exercise-03`); owns the incident retrospective process for the surface.
- **Review.** The mentee is a peer reviewer on senior-bar changes. Examples: a reviewer on cross-team RFCs; a reviewer on Model Card refreshes; a reviewer on design docs.
- **Own.** The mentee is the end-to-end owner of a system or lifecycle. Examples: owns the drift-monitoring lifecycle for the surface; owns the feature-platform integration lifecycle.

Three to five targets is typical. More than that fragments the plan and under-serves each. The targets are ordered: the plan will likely focus 60% of coaching effort on the first two.

**§4. Growth areas and objectives.**

For each target in §3, a growth-area block:

- **Growth area name.** E.g., "multi-quarter roadmap authorship."
- **Measurable objective.** The specific evidence of growth. "By 2027-Q2, mentee authors a 3-quarter roadmap for the drift-monitoring initiative, passes senior-mentor review with ≤ 3 structural comments, and signs the final version as drafter." Measurable objectives are dated and have a specific artefact.
- **Coaching tactics.** The specific things the mentor will do. "Mentor pairs with mentee on §4 of `mod-310/exercise-01` draft (assumption framing) in a 90-minute working session; mentor provides written feedback on the draft within 48 hours of each iteration; mentor and mentee review one peer's roadmap together each month as calibration practice." Generic ("I'll mentor") does not count.
- **Stretch assignment.** The piece of work that will produce the evidence. "Mentee is the primary author of the drift-monitoring roadmap, with the mentor as a named reviewer rather than a co-author; the roadmap is the mentee's artefact at the quarterly readout." Stretch assignments are real work the team needs done, not synthetic exercises.
- **Sustainment plan.** Once the objective is hit, how does the capability stay in the mentee's practice? "Mentee is the primary author on the next two roadmaps the surface produces; mentor rotates to reviewer-only on roadmap work." Without a sustainment plan, growth evaporates when the mentor's attention moves.

A mentorship plan with three growth areas is a reviewable plan; one with eight is a to-do list. Depth beats breadth.

**§5. Mechanisms.**

- **1:1 cadence.** Weekly, 30 minutes, same slot, protected. Half the time is mentee-driven agenda (what they want coached on), half is mentor-driven agenda (what the mentor is seeing that the mentee may not). "Protected" means the slot is not the first thing to move when calendars fill — a cadence that moves twice a quarter is a cadence that is signalling it is low-priority.
- **Pair-programming / pair-authoring.** Named monthly sessions where mentor and mentee work on the same artefact together. One 90-minute pair-authoring session is worth five 1:1 debriefs for the specific artefact; the plan budgets for both.
- **Review-shadowing.** The mentee shadows the mentor in design reviews, code reviews of senior-bar changes, incident retros, and cross-team meetings where the mentor represents the surface. The mentee does not just observe — the mentee drafts the artefact (the design-review notes, the code-review comments, the incident-retro summary) and the mentor reviews.
- **Review-leading.** After a few quarters of shadowing, the mentee leads one review per month, with the mentor in the room as a backstop. The leveling moment is when the mentee can lead the review without the mentor in the room; the plan tracks this transition.
- **Feedback in both directions.** The mentor solicits feedback from the mentee on the mentoring — what is working, what is not, what the mentee wants more / less of. The absence of upward feedback is not evidence of satisfaction; it is evidence that the mentor has not set the norm.
- **Peer exposure.** The mentee is deliberately connected to senior engineers outside the mentor's team (other `team-*` senior ML engineers; `team-platform` senior engineers) to broaden the leveling reference. A mentee calibrated only against one mentor's style will reproduce that one style.
- **Escalation path.** If the mentor-mentee relationship needs adjustment (mismatch of style, saturation, change in mentee's or mentor's scope), the mentee can raise it with the manager without penalty. Named up front so it is not a hidden option.

**§6. Explicit "not in scope."**

Named exclusions, mirroring §1 scope. The plan names what it is not:

- **Not performance management.** Not an input to performance review, not a substitute for it. If the manager has performance-management concerns, those go through the performance channel, not the mentorship plan. Mixing the two destroys the trust the mentorship depends on.
- **Not compensation.** Levels in §3 are development targets, not promotion commitments. Reaching the target is a necessary condition for promotion in some organizations; the mentorship plan does not promise promotion.
- **Not therapy.** Career-change conversations, personal-struggle conversations, and conflict-mediation with other engineers are out of scope. The plan names the professional resources for those — EAP, HR partner, skip-level, as applicable.

**§7. Review cadence and exit.**

- **Monthly check-in.** A 60-minute end-of-month review of the plan where the mentor and mentee look at: progress on each §4 objective, evidence accumulated, obstacles, re-plan-if-needed.
- **Quarterly re-sign.** At each quarter boundary, mentor and mentee re-read the plan, update the §2 current read, update §3 targets if scope has shifted, and sign the updated plan. Manager is copied.
- **Re-plan triggers.** Explicit named signals: mentee's scope changes (new project, new surface, team re-org); mentor's capacity changes materially; mentee hits an objective ahead of schedule; mentee misses an objective by > 1 quarter.
- **Exit criteria.** What marks "the mentorship arc for the current targets is complete." Not "the mentee is promoted" (promotion decisions are made by the manager and the organization, not by the mentor), but rather "the mentee can author a roadmap at senior bar without mentor co-authoring, as evidenced by the last two roadmaps in-repo; the mentee leads design reviews without the mentor present, as evidenced by the last three design-review notes; the mentee owns the Model Card refresh for the surface end-to-end, as evidenced by the current Model Card's drafter attribution." Exit is specific and evidence-based.
- **Post-exit.** The mentorship relationship does not end the moment the exit criteria hit. It transitions to a lighter cadence (monthly 1:1 instead of weekly) with a named topic ("broader-scope leadership," "cross-team influence," "next-layer growth") or it closes explicitly.

**§8. Signatures.**

- **Mentee signature.** The mentee signs to attest that the plan reflects their understanding of their current state and their target. A plan unsigned by the mentee is a plan written at the mentee.
- **Mentor signature.** The mentor signs to commit to the coaching tactics in §4 and the mechanisms in §5. A plan unsigned by the mentor is a wish.
- **Manager acknowledgement.** The manager acknowledges the plan (not approves — the mentor and mentee own it) and confirms that the §3 targets are aligned with the organization's leveling expectations. Named date.

### 2.2 Coaching tactic patterns

The §4 "coaching tactics" line is the part of the plan that most differentiates a senior mentor from a well-intentioned one. The following patterns are the moves senior mentors actually make; the plan should name which ones are in scope for which growth area.

- **Pair-authoring, not writing-for.** The mentor sits with the mentee at the artefact and the mentee drives the keyboard. The mentor's job is to ask questions that make the mentee do the thinking ("what would a reviewer ask first about this §1?", "what is the one-sentence why?"). Writing the artefact *for* the mentee produces an artefact; it does not produce a mentee who can write artefacts.
- **Write feedback down; review feedback in person.** Verbal-only feedback is lossy. The mentor writes the feedback (so the mentee can re-read, so the record is observable, so the mentor's thinking is visible), then discusses the feedback in person (so clarifications happen in real time, so tone is readable, so disagreement is safe).
- **Calibrate against peers, not against the mentor.** When a mentee asks "is my draft good?" the mentor's answer is "let's compare it to three roadmaps we both think are senior-level" — the sibling solutions here or team-local artefacts the mentor and mentee both read first. Calibration against artefacts beats calibration against the mentor's taste.
- **Sponsor, don't just mentor.** Mentoring is private — coaching behind a closed door. Sponsorship is public — the mentor advocates for the mentee in rooms the mentee is not in. The plan names at least one sponsorship action per quarter: "mentor proposes mentee as the design-review lead for M2 of the drift-monitoring roadmap at the next cross-team sync." Sponsorship is how mentorship compounds organizationally.
- **Review-and-release.** A pattern where the mentor reviews an artefact, releases comments, and does *not* re-review unless the mentee asks for a re-review. This keeps the mentee from treating the mentor's review as the single blocking step. The mentor's job is to give feedback; the mentee's job is to decide what to do with it. The approval path for the artefact is the normal team review, not a back-channel from the mentor.
- **Protected failure budget.** The mentee is allowed to ship a few drafts that need revision — explicitly, with the mentor's foreknowledge, with the mentor's soft-landing of any team response. Growth happens at the edge of competence; a mentor who protects the mentee from all failure keeps them below the edge. A mentor who exposes them to catastrophic failure has misjudged the edge. Protected failure is between the two.
- **Shift-left on framing.** The expensive coaching happens at the framing stage of an artefact (§1 of a roadmap, §1 of a design doc). A roadmap written on a bad thesis cannot be rescued by review; it can only be rewritten. The mentor's time is best spent at framing, not at the twelfth revision of §5.
- **Model the moves, name them explicitly.** When the mentor is doing something in a shared meeting that is a senior move (reframing a question, naming a non-goal, calling a bet), the mentor flags it afterward: "I did X there because of Y — I want you to notice this pattern so you can do it later." Modeling without naming is invisible training.
- **"What would you do differently" retros.** After an incident, a readout, or a design-review that the mentee led, the first question is not "what went well" and not "what went wrong" — it is "what would you do differently next time?" The question centers the mentee's agency.

### 2.3 Illustrative worked example (not a real individual)

> **Illustrative mentee** — a composite mid-level ML engineer growing into senior-level responsibilities on the `agent-assist/draft` surface. This example is constructed for calibration; do not treat as a real person, performance record, or team.

```markdown
# Mentorship Plan: `agent-assist/draft` surface, 2027-Q1 through 2027-Q4

## §1. Parties and scope
- Mentee: `team-agent-assist` mid-level ML engineer (L25). Target: readiness
  for L30 senior ML engineer responsibilities on this surface.
- Mentor: `team-agent-assist` senior ML engineer (L30), owner of the surface
  roadmap and Model Card. Match rationale: shared surface, mentor can offer
  direct observation and in-flow coaching on the artefacts the mentee is
  growing toward.
- Second mentor (for cross-team RFC capability, which the primary mentor
  is not currently the best-placed to coach): `team-platform-eval` senior
  ML engineer. Scope: RFC-authoring and cross-team negotiation only.
- Manager: `team-agent-assist` manager — acknowledges this plan and will
  calibrate §3 targets against the organization's L30 leveling expectations.
- Scope in: coaching on roadmap authorship, design-review facilitation,
  Model Card ownership, cross-team RFC drafting.
- Scope out: performance-management conversations, compensation
  discussions, pager-rotation adjustments. Those go through the
  manager, not this plan.

## §2. Current read
Evidence basis: the mentee's 2026 commits to `agent-assist/*` repos
(roughly 170 PRs reviewed by the mentor), two design docs the mentee
co-authored (link in-repo), participation in the 2026-Q3 Model Card
refresh (contributor), and one incident the mentee ran as IC
(`IR-2026-087`).

Scope today:
- Sole owner: the feature-normalization layer (`features/numeric_scaler.py`
  and related).
- Co-owner: the offline eval harness (with another mid-level engineer).
- Contributor: the Model Card (contributed §4 metrics table and §7
  fairness section for v4.2).
- Consulted: cross-team RFCs on feature-platform integration.

Strengths:
- Numerical correctness: identified the dtype-promotion regression in
  `make_features.py` during a routine PR review in 2026-Q3 that would
  have gone to production as a silent `float32→float64` cost regression.
- Incident IC: ran `IR-2026-087` (a schema-validity regression) start to
  finish, including the retro — retro is cited in `mod-307/exercise-02`
  style discussions on the team.
- Pair-programming partner: mentee is the first-call for other engineers
  on numerical / dtype questions.

Gaps against senior bar:
- Has not yet authored a multi-quarter roadmap (per `mod-310/exercise-01`).
  Contributed §5 of the 2026 roadmap but did not own the thesis.
- Has not yet owned a Model Card refresh end-to-end (per
  `mod-309/exercise-01`). Contributed sections; has not driven sign-off.
- Has not led a design-review as facilitator; has participated as author
  and reviewer.
- Has not authored a cross-team RFC (per `mod-308/exercise-03`).

Manager calibration: aligned — the gaps named are the standard L30
"owns scope beyond IC delivery" expectations.

## §3. Target responsibilities
1. Author a multi-quarter roadmap (shape: `mod-310/exercise-01`) for a
   named initiative on this surface.
2. Own a Model Card refresh end-to-end (shape: `mod-309/exercise-01`),
   including sign-off coordination.
3. Facilitate the monthly design-review for this surface.
4. Author a cross-team RFC contributing back to `feature-platform` (shape:
   `mod-308/exercise-03`), a gap identified in the 2026 paved-road audit.

(Four targets; roadmap and Model Card are the primary focus for Q1–Q2.)

## §4. Growth areas and objectives

### Growth area A: Multi-quarter roadmap authorship
- Measurable objective: by 2027-Q2 end, mentee is the sole drafter of
  the drift-monitoring initiative's 2027-Q3/Q4 roadmap, passes senior-
  mentor review with ≤ 3 structural comments on the final iteration,
  and signs the roadmap as drafter with mentor as reviewer.
- Coaching tactics:
  - 90-minute pair-authoring session on §1 thesis and §4 assumptions
    once the capability is scoped by the surface team.
  - Written feedback from mentor within 48h of each iteration;
    in-person review for discussion.
  - Mentor-and-mentee review of one peer-team roadmap together as
    calibration (`mod-308/exercise-03` sibling) in Q1.
- Stretch assignment: mentee is the primary author of the drift-
  monitoring roadmap. Mentor is a named reviewer only; mentor does not
  co-draft.
- Sustainment: mentee drafts the next two roadmaps the surface
  produces; mentor transitions to reviewer-only.

### Growth area B: Model Card ownership end-to-end
- Measurable objective: by 2027-Q2 end, mentee has driven a Model Card
  refresh (next scheduled: `assist-llm-v4.3`) from draft through three-
  signature sign-off, with mentor as reviewer.
- Coaching tactics:
  - Review-shadowing: mentee drafts the signature-coordination plan;
    mentor reviews the plan.
  - Pair-authoring on §7 (fairness analysis) only for the first refresh;
    full drafting by mentee for subsequent refreshes.
  - Sponsorship: mentor introduces mentee as the Model Card owner at
    the next governance-reviewer sync.
- Stretch assignment: mentee is the drafter attribution on the v4.3
  Model Card.
- Sustainment: mentee is the Model Card drafter for v5 and v6.

### Growth area C: Design-review facilitation
- Measurable objective: by 2027-Q3 end, mentee has facilitated three
  consecutive monthly design-reviews without the mentor in the room,
  with review notes in-repo that pass the senior-mentor's §2.2 rubric
  of `mod-310/exercise-02`.
- Coaching tactics:
  - Q1: mentee shadows mentor and drafts the review notes; mentor
    reviews.
  - Q2: mentee facilitates with mentor in the room as a backstop;
    mentor debriefs in the next 1:1.
  - Q3: mentee facilitates without mentor present; mentor reads the
    notes afterward.
  - Modeling-and-naming: when mentor reframes a design-review question
    in Q1 or Q2, mentor names the move in the debrief.
- Stretch assignment: Q3 design-reviews are mentee-facilitated.
- Sustainment: mentee is the default facilitator from Q3 onward;
  mentor steps in only on explicit escalation.

### Growth area D: Cross-team RFC drafting
- Measurable objective: by 2027-Q4 end, mentee has authored a
  contribute-back RFC (shape: `mod-308/exercise-03`) that has been
  reviewed by the `feature-platform` team and either merged or
  explicitly declined with a documented decision.
- Coaching tactics (second mentor — `team-platform-eval` senior
  ML engineer — leads this growth area):
  - Pair-authoring on §1 problem statement and §3 proposed design.
  - Second mentor's cross-team relationships used for sponsorship.
  - Primary mentor stays in the loop for calibration against the team's
    other artefact work.
- Stretch assignment: the RFC on prompt-version indexing, which the
  2026 paved-road audit identified (and `mod-308/exercise-01` style
  documents).
- Sustainment: mentee owns the follow-up conversations and the
  implementation if the RFC is accepted.

## §5. Mechanisms
- 1:1 cadence: weekly 30 minutes, Tuesdays 2pm, protected. Half
  mentee-driven, half mentor-driven.
- Pair-authoring sessions: one named 90-minute session per growth area
  per quarter, scheduled at the start of each quarter.
- Review-shadowing: mentee shadows mentor in all design-reviews for
  Q1 and all Model-Card-related meetings for Q1–Q2. Mentee drafts the
  artefact (notes, agenda, action items) and mentor reviews.
- Review-leading: Q2 onward mentee leads one design-review per month.
- Feedback in both directions: end of each monthly check-in, mentor
  asks for feedback on mentoring. Mentor documents the feedback.
- Peer exposure: mentee is introduced to the `team-platform-eval`
  senior ML engineer (second mentor) in Q1. Mentee attends two cross-
  team syncs per quarter where mentor is not present, to build
  independent calibration.
- Escalation path: mentee can raise concerns with manager at any
  time. If the mentor-mentee relationship needs re-plan, the mentor
  facilitates the transition, not the mentee.

## §6. Explicit "not in scope"
- Not performance management. Performance conversations go through
  the manager, not this plan. Mentor and mentee do not discuss
  performance ratings.
- Not compensation. L30 target is a development target; promotion
  decisions are made by the manager and the organization.
- Not conflict mediation. Conflicts with other engineers go through
  the manager or HR partner.

## §7. Review cadence and exit
- Monthly: 60-minute end-of-month plan review on the fourth Tuesday.
- Quarterly re-sign: end of each quarter; updated plan signed by
  mentor, mentee, and manager-acknowledged.
- Re-plan triggers:
  - Mentee's scope changes (new project, surface re-charter, team
    re-org).
  - Mentor's capacity changes materially (> 20% reduction, or scope
    change that moves mentor off this surface).
  - Mentee hits a growth-area objective more than one quarter early
    (re-evaluate targets).
  - Mentee misses a growth-area objective by more than one quarter
    (re-evaluate coaching tactics or stretch assignment — not the
    mentee).
- Exit criteria:
  - Growth area A: mentee has drafted the drift-monitoring roadmap
    (Q2 end) and the next two roadmaps (Q3 / Q4).
  - Growth area B: mentee has driven v4.3 (Q2) and v5 (Q3) Model
    Cards end-to-end.
  - Growth area C: three consecutive mentee-led design-reviews with
    no mentor in the room by Q3 end.
  - Growth area D: one RFC authored, reviewed by the destination
    team, with a documented outcome by Q4 end.
- Post-exit: transition to monthly 1:1 cadence in 2028-Q1; the next
  topic is "broader-scope leadership — influence across teams."
  Re-plan at that point.

## §8. Signatures
- Mentee: (signed, 2026-12-15)
- Mentor: (signed, 2026-12-15)
- Manager acknowledgement: (acknowledged, 2026-12-18) — L30 targets
  confirmed aligned with team leveling expectations.
```

### 2.4 Lightweight plan linter (static checks)

```python
# mentorship_plan_lint.py
"""Lint a mentorship plan for structural completeness.

Enforces: required sections, evidence-anchored current read, measurable
objectives with dates, named sustainment, named exit criteria, and
explicit out-of-scope.
"""

import re
import sys


REQUIRED_SECTIONS = [
    "parties", "current read", "target", "growth areas",
    "mechanisms", "not in scope", "review cadence", "signatures",
]

REQUIRED_COACHING_TOKENS = [
    "pair", "shadow", "feedback", "sponsorship",
]


def lint(text: str) -> list[str]:
    errors: list[str] = []
    lowered = text.lower()
    for s in REQUIRED_SECTIONS:
        if s.lower() not in lowered:
            errors.append(f"missing section: {s}")
    if not re.search(r"evidence|artefact|artifact|pr|design doc", lowered):
        errors.append("current read has no evidence anchor")
    # growth area blocks should name date-bounded objectives
    if not re.search(r"\b202\d-Q[1-4]\b|\bby \d{4}", text):
        errors.append("no dated objective found; growth-area objectives "
                      "must be dated")
    if "sustainment" not in lowered and "sustain" not in lowered:
        errors.append("no sustainment plan named per growth area")
    for t in REQUIRED_COACHING_TOKENS:
        if t not in lowered:
            errors.append(f"coaching tactic not referenced: {t}")
    if "manager" not in lowered:
        errors.append("manager not acknowledged in plan")
    if not re.search(r"re.?plan trigger", lowered):
        errors.append("no re-plan triggers named")
    if "exit criteria" not in lowered:
        errors.append("no exit criteria named")
    if "performance" not in lowered or "not performance" not in lowered:
        errors.append("plan does not name 'not performance management' "
                      "out-of-scope")
    return errors


if __name__ == "__main__":
    errs = lint(open(sys.argv[1]).read())
    for e in errs:
        print(f"mentorship-plan-lint error: {e}")
    sys.exit(1 if errs else 0)
```

## 3. Validation steps

Walk the plan through this checklist before signing:

1. **§1 names mentor, mentee, manager, and scope by role.** A plan missing any party is a plan with hidden dependencies.
2. **§1 scope explicitly names what is out.** Performance management, compensation, conflict mediation are explicitly out.
3. **§2 current read is anchored in named artefacts.** Three to five PRs, design docs, or meeting notes per strength / gap. No "generally strong at X" without evidence.
4. **§2 includes manager calibration.** The manager has confirmed the leveling read is aligned with organizational expectations.
5. **§3 targets are concrete artefact-level responsibilities**, not traits or titles.
6. **§4 growth areas have measurable, dated objectives.** Date + specific artefact + review bar.
7. **§4 coaching tactics are specific.** Pair-authoring sessions named by duration and cadence; written-feedback-within-48h named; calibration-against-peers named.
8. **§4 includes a stretch assignment that is real work the team needs.** Not a synthetic exercise.
9. **§4 includes a sustainment plan.** Growth evaporates without it.
10. **§5 mechanisms name the 1:1 cadence, pair-authoring budget, review-shadowing and review-leading transitions, feedback-in-both-directions, peer exposure, and escalation path.**
11. **§6 names explicit exclusions.** Not performance management; not compensation; not therapy.
12. **§7 names monthly check-in, quarterly re-sign, re-plan triggers, exit criteria, and post-exit transition.** Exit criteria are evidence-based, not title-based.
13. **§8 signatures include mentor, mentee, and manager acknowledgement.** All dated.
14. **Cross-reference pattern.** The plan cites the senior-bar artefacts the mentee is growing toward (`mod-310/exercise-01` roadmap; `mod-310/exercise-02` reviews; `mod-308/exercise-03` RFC; `mod-309/exercise-01` Model Card).
15. **Plan is a joint document.** The mentee has co-edited it; the plan is not one-sided mentor dictation.

Static checks a CI step can enforce:

```bash
grep -ciE "mentor|mentee|manager"                       mentorship-plan.md
grep -ciE "evidence|artefact|artifact|PR"               mentorship-plan.md
grep -ciE "by 202\d-Q[1-4]|by 202\d"                    mentorship-plan.md
grep -ciE "pair|shadow|sponsorship|review-shadow"       mentorship-plan.md
grep -ciE "re-?plan trigger|exit criteria|post-?exit"   mentorship-plan.md
grep -ciE "not performance|not compensation"            mentorship-plan.md
```

`mentorship_plan_lint.py` from §2.4 gives a programmatic version.

## 4. Rubric / review checklist

- [ ] §1 names parties (mentor, mentee, manager, any second mentor) by role; match rationale stated.
- [ ] §1 scope-in and scope-out are both explicit; performance management is explicitly out.
- [ ] §2 current read is anchored in three to five specific artefacts per strength and per gap.
- [ ] §2 manager calibration is captured, including any disagreement and its resolution.
- [ ] §3 targets are concrete artefact-level responsibilities (author, facilitate, review, own), not traits.
- [ ] §3 has three to five targets, ordered by coaching priority.
- [ ] §4 has a growth-area block per target with: name, measurable dated objective, coaching tactics (specific, not generic), stretch assignment (real work), sustainment plan.
- [ ] §4 coaching tactics include at least: pair-authoring, written-feedback-plus-in-person-discussion, peer-calibration, sponsorship, modeling-and-naming, protected failure budget.
- [ ] §5 mechanisms include: 1:1 cadence (protected), pair-authoring budget, review-shadowing to review-leading transition, feedback-in-both-directions norm, peer exposure, named escalation path.
- [ ] §6 names explicit exclusions: not performance management, not compensation, not conflict mediation / therapy.
- [ ] §7 names monthly check-in, quarterly re-sign, re-plan triggers (scope change, capacity change, early hit, miss by > 1 quarter), exit criteria (evidence-based), and post-exit transition.
- [ ] §8 includes mentor signature, mentee signature, and manager acknowledgement — all dated.
- [ ] Plan cites the sibling artefact standards the mentee is growing toward (`mod-308`, `mod-309`, `mod-310`).
- [ ] Plan is co-authored (mentee has visibly edited it), not dictated.
- [ ] NIST AI RMF reference is present where the growth areas touch governance artefacts.

Review-weighted points for grading:

- §2 current read with evidence anchors: 15%
- §3 targets as concrete artefact-level responsibilities: 10%
- §4 growth-area blocks with measurable objectives, named coaching tactics, stretch assignments, sustainment: 30%
- §5 mechanisms (cadence, pair-authoring, shadow-to-lead, upward-feedback, sponsorship, escalation): 15%
- §6 explicit out-of-scope (not-performance, not-compensation, not-therapy): 10%
- §7 cadence, re-plan triggers, evidence-based exit criteria: 10%
- §8 signatures and manager acknowledgement: 5%
- Cross-references to sibling senior-bar artefacts: 5%

## 5. Common mistakes

- **"Develop leadership" as an objective.** Not measurable, not dated, not evidence-anchored. Replace with a specific artefact ("author the Q3 roadmap") and a specific outcome ("passes senior-mentor review with ≤ 3 structural comments").
- **Current read without evidence.** "Strong engineer, needs to grow in scope" is a vibe. Three PRs, two design docs, one incident retro — named — is a read.
- **Targets as titles.** "Become senior." Not a target. "Author a multi-quarter roadmap, own a Model Card refresh, facilitate design-reviews, author a cross-team RFC" is a target set.
- **Too many growth areas.** Eight growth areas means two days per area per quarter. Three or four is the depth that produces change.
- **Generic coaching tactics.** "I'll mentor" is not a tactic. "90-minute pair-authoring session on §4 of the roadmap, mentor-written feedback within 48h, calibration against one peer-team roadmap per quarter" is.
- **Stretch assignment is a synthetic exercise.** The growth evidence is real work. A synthetic exercise produces a synthetic artefact that the team has no reason to invest in.
- **No sustainment plan.** Hitting the objective once does not install the capability; the sustainment plan is what makes the capability the mentee's default practice.
- **No sponsorship action.** Mentorship without sponsorship stays private. The mentor must publicly advocate for the mentee at least once per quarter.
- **Not naming the upward-feedback norm.** Mentors who do not solicit feedback from mentees collect only the signal the mentee volunteers, which is biased toward politeness.
- **Blurring mentorship and performance management.** Performance ratings go through the manager; mentorship is a separate trust surface. Mixing them breaks the trust.
- **No escalation path.** A mentee who wants to change mentors or change the plan but has no named mechanism will either suffer silently or escalate badly. The plan names the mechanism.
- **Exit criteria as "the mentee is promoted."** Promotion is a decision the manager and organization make, not the mentor. Exit is evidence-based: the mentee can do the work at senior bar as shown in-repo.
- **Plan dictated, not co-authored.** A plan the mentee did not edit is a plan the mentee will not execute against.
- **No quarterly re-sign.** Plans decay. Without a re-sign cadence, the plan is a draft the mentor wrote once and both parties stopped reading by month three.
- **Shadowing without drafting.** A mentee who sits in review meetings but never drafts the artefact (review notes, retro summary) learns less than a mentee who drafts and is reviewed on the draft.
- **One mentor for all gaps.** If the mentee needs capability the primary mentor cannot provide (cross-team RFC drafting; a specific technical depth), a second mentor is named. Trying to be the sole mentor across all growth areas is a mentor smell.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds mentorship as the capability-building instrument that transfers ownership of risk artefacts (Model Card, threat model, roadmap) from a senior to the next senior; the plan's §3 targets are the governance ownership transfers the GOVERN function expects to be made legible. MAP / MEASURE / MANAGE ground the specific artefacts the mentee is being grown into owning.
- NIST AI 600-1 (Generative AI Profile) — <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>. Referenced when the growth areas touch generative-surface Model Cards (per `mod-309/exercise-01`).
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Referenced when §3 targets include threat-model ownership (`mod-309/exercise-04`).
- MITRE ATLAS — <https://atlas.mitre.org/>. Referenced when §3 targets include threat-model ownership for generative surfaces.

Practitioner implementation examples:

- VeriSwarm engineering blog — <https://veriswarm.ai>. Shape-of-artefact reference for team-level growth-plan write-ups; not a standard.

Local context (authoritative for the artefact shape and cross-references):

- Paired lesson: `lessons/mod-310-technical-leadership/exercises/exercise-03-mentorship-plan-for-a-mid-level.md`.
- Module README: `modules/mod-310-technical-leadership/README.md`.
- Sibling exercise (roadmap authoring that Growth Area A targets): `modules/mod-310-technical-leadership/exercise-01-roadmap-and-scoping/SOLUTION.md`.
- Sibling exercise (code review / design review that Growth Area C targets): `modules/mod-310-technical-leadership/exercise-02-code-review-and-design-review-at-bar/SOLUTION.md`.
- Sibling exercise (interview loop — where the hiring bar is set that the mentee's targets align with): `modules/mod-310-technical-leadership/exercise-04-rubric-based-ml-interview-loop/SOLUTION.md`.
- Related exercise (RFC authorship that Growth Area D targets): `modules/mod-308-platform-collaboration/exercise-03-contribute-back-rfc-to-platform/SOLUTION.md`.
- Related exercise (paved-road audit that surfaces the RFC need for Growth Area D): `modules/mod-308-platform-collaboration/exercise-01-paved-road-consumption-audit/SOLUTION.md`.
- Related exercise (build-vs-adopt doc that may be a §3 target): `modules/mod-308-platform-collaboration/exercise-02-build-vs-adopt-tradeoff-doc/SOLUTION.md`.
- Related exercise (Model Card authorship that Growth Area B targets): `modules/mod-309-responsible-ai-governance/exercise-01-model-card-authoring/SOLUTION.md`.
- Related exercises (other governance artefacts a senior-level mentee grows into owning): `modules/mod-309-responsible-ai-governance/` (exercises 02, 03, 04).
- Related exercise (SLO doc that may be a §3 target): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (evaluation harness ownership that may be a §3 target): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (experiment design ownership that may be a §3 target): `modules/mod-306-experimentation-at-scale/exercise-01-power-mde-and-ramp-plan/SOLUTION.md`.
