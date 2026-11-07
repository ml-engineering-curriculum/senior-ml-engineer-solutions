# exercise-03 — Self-Assessment and Growth Plan (Reference Solution)

Paired exercise: `lessons/mod-301-senior-ml-role-scope/exercises/exercise-03-self-assessment-and-growth-plan.md`.
Paired chapter: `05-self-assessment-and-growth-plan.md`.

## 1. Solution overview

This exercise produces `growth-plan.md` — five competency ratings with evidence, a ranked gap list with cost-of-inaction sentences, a module-allocation table across mod-302 → mod-310, a stretch-project pick with reasoning + project ordering + reviewer commitment, a review cadence, and a 3–5 sentence closing reflection.

The artefact is personal — the "right" rating vector differs per learner, and the reference solution cannot make up a learner's actual experience without violating the source policy. What this reference does provide:

1. A **template** `growth-plan.md` with every section the acceptance criteria requires.
2. One **fully worked illustrative plan** built around the chapter 05 example rating vector `A=3, B=2, C=1, D=3, E=2` (the "lopsided senior" the chapter walks through). Clearly labeled as illustrative. Do **not** copy the ratings, because the whole point of the exercise is to produce ratings you can defend with evidence.
3. The validation and rubric sections, which are general and apply to every learner's plan.

The chapter-05 competency map (Architecture A, Evaluation & Experimentation B, Reliability & Cost C, Cross-team & Governance D, Technical Leadership E) is the backbone of the whole artefact.

## 2. Implementation (worked answer)

### 2.1 Template `growth-plan.md`

Use this skeleton; fill in your real numbers and prose.

```markdown
# Growth plan — <your name>, <date started>

## 1. Self-assessment (five rows)

### A — Architecture (mod-302 / mod-303 / mod-304)
Target level: 3
Current level: __
Evidence: <one specific, artefact-grounded sentence>

### B — Evaluation & Experimentation (mod-305 / mod-306)
Target level: 3
Current level: __
Evidence: ...

### C — Reliability & Cost (mod-307)
Target level: 3
Current level: __
Evidence: ...

### D — Cross-team & Governance (mod-308 / mod-309)
Target level: 3
Current level: __
Evidence: ...

### E — Technical Leadership (mod-310)
Target level: 3
Current level: __
Evidence: ...

Rating vector: A=_, B=_, C=_, D=_, E=_. Peak axis: __.

## 2. Ranked gaps and cost of inaction
... (descending by target − current; cost-of-inaction sentence per gap ≥ 1)

## 3. Module allocation (mod-302 → mod-310)
| Module | Planned hours | Your allocation | Reason |

## 4. Stretch project
- Chosen project + 1-paragraph reasoning.
- Ordering of the other two projects (warm-up vs capstone).
- Reviewer / mentor commitment, or named public-artefact commitment.

## 5. Review cadence
- Mid-track review: date / hours-in checkpoint.
- End-of-track review: date / exit criteria.
- Optional: weekly journal, manager check-in.

## 6. Closing reflection (3–5 sentences)
... shape / risk / what-would-change-the-plan
```

### 2.2 Illustrative worked plan (DO NOT COPY — this is calibration, not your plan)

Rating vector used throughout: `A=3, B=2, C=1, D=3, E=2`. This is the exact "lopsided senior" vector from chapter 05, chosen so the reader can verify the derivation step by step.

#### 2.2.1 Self-assessment rows (illustrative)

```
A — Architecture (mod-302 / mod-303 / mod-304)
Target level: 3
Current level: 3
Evidence: I authored the search-ranker-v3 ML system RFC last quarter; it was reviewed by the architecture reviewers and merged without a rework pass.

B — Evaluation & Experimentation (mod-305 / mod-306)
Target level: 3
Current level: 2
Evidence: I've run two A/B tests on the ranker this year; the second one needed rework because I used the aggregate CTR as the primary metric and missed an SRM that an experimentation specialist caught in review.

C — Reliability & Cost (mod-307)
Target level: 3
Current level: 1
Evidence: I can read an SLO document and follow the argument; I have not authored a freshness or quality SLO for my own system, and I have not been incident commander on an ML-side regression.

D — Cross-team & Governance (mod-308 / mod-309)
Target level: 3
Current level: 3
Evidence: I authored and shipped a contribute-back RFC to the feature-store team last cycle, and I authored the review packet (model card + data statement + threat-model draft) for the last ranker deploy.

E — Technical Leadership (mod-310)
Target level: 3
Current level: 2
Evidence: I mentor one mid-level engineer through regular PR review and 1:1s, but I have not authored a review rubric and I have not run a full interview loop end-to-end.

Rating vector: A=3, B=2, C=1, D=3, E=2. Peak axis: A (and D) at target; no axis at 4 yet.
```

Notes on *how to write* the evidence sentence — the rubric is artefact-based (ch.05):

- Rating `0` means no working mental model — you cannot name the artefact. The evidence sentence in this case is a one-liner on *why* that is, not a defensive non-answer.
- Rating `1` is "aware" — you can read it, not author it.
- Rating `2` is "produced one, needed rework." Name the artefact and the rework signal.
- Rating `3` is "produce regularly, pass review largely as authored." Name one recent artefact.
- Rating `4` is "mentor others in this competency, artefacts used as templates by peers." Name the mentee and the template.

If you cannot write the sentence for rating `N`, drop the rating to `N−1` and note why. That instruction is load-bearing (ch.05).

#### 2.2.2 Ranked gaps and cost of inaction (illustrative)

Gaps (target − current):

1. **C (Reliability & Cost): gap 2.**  Cost of inaction — the ranker's on-call rotation starts in the next quarter; if I am not fluent in SLOs and incident command by then, I will be incident commander for a regression I cannot debug and the pod will lose a quarter of velocity recovering.
2. **B (Evaluation & Experimentation): gap 1.**  Cost of inaction — the next A/B on the long-tail-seller slice needs CUPED to hit power; without it I will either ship a false positive or fail to reject a real win and the ranker roadmap will slip a cycle.
3. **E (Technical Leadership): gap 1.**  Cost of inaction — the mid-level engineer on my pod is heading into their promotion cycle; if I have not authored a review rubric and run a visible mentorship loop, they will not have the sponsorship evidence they need and I will be graded down on People at review.
4. **A (Architecture): gap 0.**  At target; invest reclaimed hours into C/B/E.
5. **D (Cross-team & Governance): gap 0.**  At target; invest reclaimed hours into C/B/E.

Tie-break rationale (per ch.05): Reliability's on-call deadline is nearer than Leadership's promotion-cycle deadline, which is nearer than Evaluation's next-A/B deadline. If the on-call rotation were further out, the ranking could re-sort — the cost-of-inaction sentence is what carries the ordering.

#### 2.2.3 Module allocation (illustrative)

Planned-hour totals come from `CURRICULUM.md` in the learning repo. If your local copy has different totals, use yours — the point is to make the delta visible, not to match these numbers. The totals below are illustrative placeholders.

<!-- Note: the per-module hours below are the chapter-05 example figures used for illustration only; confirm against the learning repo's authoritative module READMEs before using in a real plan. -->

| Module | Planned hours (illustrative) | Your allocation | Reason |
|---|---|---|---|
| mod-302 — ML systems architecture | 16 | 8 | A=3 at target. Skim chapters; run the RFC exercise only if the rubric surprises me. |
| mod-303 — advanced modelling | 14 | 6 | A=3 at target. Focus on cold-start section only; skip warm-ups. |
| mod-304 — production LLM integration | 14 | 8 | A at target, but the ranker is about to add an LLM reranker leg — run the decision-framework exercise in full, skip the rest. |
| mod-305 — advanced evaluation | 12 | 14 | B=2 gap. Read in full, run all exercises, author the slice-definition artefact for my ranker. |
| mod-306 — experimentation at scale | 12 | 14 | B=2 gap. Full read + CUPED lab + the SRM diagnostic exercise, targeted at my ranker's long-tail-seller slice. |
| mod-307 — ML reliability & SLOs | 12 | 20 | C=1 gap of 2. Load-bearing. Full read, author freshness + quality SLO for the ranker, run the incident-command simulation. Google SRE ch.2–6 in parallel. |
| mod-308 — platform collaboration | 10 | 6 | D=3 at target. Skim; re-run the contribute-back RFC exercise only. |
| mod-309 — responsible AI & governance | 12 | 8 | D=3 at target. Focus on the slices covered by the ranker review packet (fairness + robustness). |
| mod-310 — technical leadership | 10 | 14 | E=2 gap. Full read + author one review rubric + run one simulated interview loop with the mid-level engineer. |
| **Total** | **~112** | **~98** | Reclaimed hours from A/D (−14) spent on C (+8), B (+4), E (+4). |

Totals ≠ planned, which is fine (ch.05). What matters is naming where you are spending more than planned (C, B, E, part of mod-304) and where less (A modules, D modules), with a reason per row.

#### 2.2.4 Stretch project (illustrative)

Reading the three project READMEs in `projects/` and applying the chapter-05 heuristic:

- Gap on Architecture → project-301 (three senior-level RFCs). **Not** my case (A=3).
- Gap on LLM integration + Evaluation → project-302. Partially my case (B gap, A already at target).
- Gap on Leadership + Cross-team → project-303 (tech-lead simulation). Partially my case (E gap, D at target).

**Chosen stretch:** project-303 (tech-lead simulation). My biggest gap by cost-of-inaction is C, but project-303 compounds the E gap with an artefact I am nervous to author (an interview rubric + a simulated roadmap + a mid-cycle calibration). Project-302 would be more comfortable because I already have strong A/D. The point of the stretch is discomfort (ch.05).

**Ordering of the other two:** project-301 first as the warm-up — three RFCs on systems I already understand is calibration work, not stretch. Project-302 last as the capstone — by then I will have closed B via mod-305/306 and the LLM-integration decision from mod-304 will have landed in the real ranker.

**Reviewer commitment:** I will ask my manager to review the project-303 interview rubric, and I will share the simulated roadmap with one peer senior ML engineer at my company for red-team review. If a reviewer is not available, I will publish the growth plan and each project's deliverables as commits in a (private) GitHub repo and share the repo link in my 1:1 — the public commitment is the forcing function.

#### 2.2.5 Review cadence (illustrative)

- **Mid-track review** at ~100 hours in (after mod-306). Re-run the five-point rubric; re-write the ranked-gap list; adjust the allocation for the second half. Specifically — if C has moved from 1 to 3, reallocate mod-307 hours into mod-310. If C has only moved to 2, keep investing.
- **End-of-track review** after all three projects ship. Re-run the rubric a third time. Compare `start → mid → end` for each competency. Note which axis moved most and which moved less than expected. Draft the next-cycle plan — likely either "deepen on the peak axis toward L40" or "invest in the one axis that did not move."
- **Weekly journal:** one line per module exercise completed, including the rating update it caused. Optional; include only if I have shown weekly-journal discipline before (I have not, historically; I will trial it for the first four weeks and drop if I miss two weeks in a row).
- **Manager check-in** at each formal review. Even if the plan does not change, speaking it out loud usually exposes a mis-rating.

#### 2.2.6 Closing reflection (illustrative, 4 sentences)

> The shape is heavily weighted to Reliability and Leadership this cycle — ~35% of my hours on C and E combined, versus ~20% planned. The risk is that I will under-invest in Architecture during a cycle when my team is adding an LLM reranker leg, and I could enter the mod-310 projects architecturally stale; I mitigate by keeping ~8 hours on mod-304's decision-framework exercise even though A is at target. What would change the plan mid-track: if C moves from 1 to 3 after mod-307 alone (plausible — SLO authoring is faster to calibrate than I expect), I will move the C surplus into E rather than letting it sit idle. The 4-across-the-row shape at the end of the track is not the goal; a 3-3-3-3-3 with a credible path to 4 on E by next cycle is.

## 3. Validation steps

Before submitting the plan, walk through this checklist.

1. **Every rating has artefact-grounded evidence.** The rating column of each row maps to a sentence that names an artefact (RFC, A/B writeup, SLO document, review packet, mentorship outcome). "I read X" is not evidence (ch.05). If you can't write the evidence sentence, drop the rating by one.
2. **Gap list is ranked and tie-broken.** Every competency with gap ≥ 1 has a cost-of-inaction sentence that names a specific near-term event (next on-call rotation, next A/B, next promotion cycle). The ranking visibly follows the gaps and is tie-broken by those events.
3. **Allocation table sums are visible.** The total row names your allocation total, the delta from planned, and which rows you are spending more / less on. If the total equals planned, you have not actually reallocated.
4. **No module with gap ≥ 2 is skipped.** Chapter-05 rule. Grep the "Your allocation" column against the gap list — if `C` has gap 2 and `mod-307` is allocated <planned, fix the plan.
5. **Stretch project is the uncomfortable one.** Ask yourself honestly: of the three, which would you be most nervous to show to a reviewer? That is your stretch, per ch.05. The heuristic from the chapter is a prior, not an answer.
6. **Reviewer / public commitment is named.** A plan that says "I will publish eventually" is not a commitment. A named person or a named repo is.
7. **Cadence has concrete checkpoints.** "Mid-track review" is not a cadence; "at ~100 hours in, after mod-306, I will re-run the rubric" is.
8. **Reflection answers the three required questions.** Shape of the plan, risk of the plan, what would change it mid-track. All three must appear in 3–5 sentences.
9. **Document fits under 4 pages.** Chapter 05: if it is longer, you are over-planning.
10. **Document is under version control.** Commit the plan at `v1`. The mid-track and end-of-track reviews are future commits. The delta is the evidence.

Static check:

```bash
# From the artefact's directory.
grep -c "Current level:" growth-plan.md    # expect 5
grep -c "Cost of inaction" growth-plan.md  # expect ≥ (#rows with gap ≥ 1)
grep -c "^| mod-" growth-plan.md           # expect 9 (one per module mod-302..mod-310)
wc -w growth-plan.md                       # ~1500-2500 words is in range; much higher = over-planning
```

## 4. Rubric / review checklist

- [ ] Five self-assessment rows, each with competency, target, current, and an artefact-grounded evidence sentence.
- [ ] Ratings use the chapter-05 rubric semantics (0 novice / 1 aware / 2 emerging / 3 solid / 4 excellent), not a 10-point scale.
- [ ] Rating vector summary line (`A=_, B=_, C=_, D=_, E=_`) present.
- [ ] Peak axis named (where the learner is closest to a 4, per ch.05 — even a "strong-across" learner has one).
- [ ] Ranked gap list, descending by gap size, with tie-break by cost of inaction.
- [ ] Cost-of-inaction sentence for every competency with gap ≥ 1, naming a specific near-term event.
- [ ] Module allocation table covers mod-302 through mod-310 with four columns (module / planned / your allocation / reason).
- [ ] No module with gap ≥ 2 has reduced allocation.
- [ ] Allocation total visible with delta from planned and direction of reallocation.
- [ ] Stretch project picked with a 1-paragraph reasoning that names discomfort, not comfort.
- [ ] Ordering of the other two projects specified (warm-up vs capstone) with a reason for each.
- [ ] Reviewer or public-artefact commitment named.
- [ ] Review cadence with explicit mid-track and end-of-track checkpoints, each with a concrete hour-in or module-completed trigger.
- [ ] Optional cadence items (journal, manager check-in) marked optional and committed to or declined, with a reason.
- [ ] Closing reflection (3–5 sentences) answers shape / risk / what-would-change-the-plan.
- [ ] Document ≤ 4 pages (~2500 words).
- [ ] Document committed as `v1` to version control.
- [ ] If stretch goals attempted: hand-off map (exercise-02) contracts are tagged by competency and the ranking is cross-checked against them.
- [ ] If stretch goals attempted: rating vector compared against an L30 posting's signature (exercise-01) with under-qualified-for-today slots called out.

Review-weighted points (for use when grading):

- Evidence quality on the five ratings: 30%
- Gap ranking + cost-of-inaction concreteness: 15%
- Module allocation realism (totals, reallocation, skip-rule compliance): 20%
- Stretch project pick + ordering + reviewer commitment: 15%
- Cadence concreteness: 10%
- Reflection honesty and risk-naming: 10%

## 5. Common mistakes

- **Rating yourself on knowledge, not artefacts.** "I know what CUPED is" is a 1 (ch.05). A 2 or 3 requires an artefact you authored and the review outcome on it.
- **Giving every competency a 3.** The straight 3-3-3-3-3 row is suspicious unless every evidence sentence names a different recent artefact. Chapter 05 says real seniors are lopsided; a flat row usually means the learner has not been honest with themselves.
- **Giving yourself a 4 without naming a mentee or a template.** A 4 means your artefacts are used as templates by peers, not that you are good at the competency. If you cannot name the peer, you are a 3.
- **Ranking gaps by gap size only and ignoring cost of inaction.** A gap of 2 on Leadership with no near-term mentorship expectation is less urgent than a gap of 1 on Reliability with an on-call rotation next quarter. The tie-break is mandatory (ch.05).
- **Reallocating hours by vibes rather than by gap size.** Every row of the allocation table needs a one-sentence reason that points at the gap. "Skim this" is not a reason; "At target, skim chapters and run only the RFC exercise" is.
- **Skipping a module with gap ≥ 2.** Chapter-05 rule. Reclaiming hours comes from the at-target modules, not the gap modules.
- **Picking the comfortable project as the stretch.** The stretch is the one you are nervous to author in front of a reviewer. If the pick feels safe, re-pick.
- **Running the stretch project first or last with no stated reason.** Chapter 05: default to running the stretch second. Any other ordering must be explicitly justified.
- **No reviewer named.** "I will share with a mentor" without a name is not a commitment. Either name a person or name a public artefact.
- **Vague cadence.** "Review as I go" is not a cadence. The mid-track review must have a concrete trigger (hours-in, module completed, calendar date).
- **Reflection that praises the plan.** The reflection's job is to name the risk in the plan and the thing that would change it. A reflection that says "this plan is well-balanced" is not calibrated.
- **Document over 4 pages.** Chapter 05 explicitly caps this — long plans are over-planning. Cut rationale, keep decisions.

## 6. References

Official standards cited for the module's source policy (not directly load-bearing for this personal-planning exercise, but Competency D ratings rest on these):

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — <https://atlas.mitre.org/>
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. Competency D evidence that touches governance review packets is expected to tie back to the AI RMF function categories.

Local context (authoritative for this exercise's rubric and allocation rules):

- Paired lesson: `lessons/mod-301-senior-ml-role-scope/exercises/exercise-03-self-assessment-and-growth-plan.md`.
- Chapter 05 — Self-assessment and growth plan: `lessons/mod-301-senior-ml-role-scope/05-self-assessment-and-growth-plan.md`. Source of the five-competency map (A–E), the five-point rubric (0–4), the four-part growth-plan structure (ranking / allocation / stretch / cadence), the "do not skip modules with gap ≥ 2" rule, the stretch-project heuristic, and the lopsided-senior / strong-across-senior worked examples.
- Chapter 02 — Altitude and ownership: `lessons/mod-301-senior-ml-role-scope/02-altitude-and-ownership.md`. Source of the five ownership axes the competency map rests on.
- Paired upstream solutions: exercises 01 and 02 feed this plan — the verb audit's L30 signature calibrates target levels, and the hand-off map surfaces which contracts imply Competency D or E gaps.
- Module READMEs: `modules/mod-302-ml-systems-architecture/README.md` through `modules/mod-310-technical-leadership/README.md` (and the paired learning repo) — authoritative source of per-module planned hours; use the live values rather than the illustrative ones in this document.
- Learning-repo `CURRICULUM.md` — authoritative total hour budget; per-module planned hours are canonical in each module README in the learning repo.
