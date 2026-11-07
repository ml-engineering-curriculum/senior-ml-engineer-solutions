# exercise-01 — Seniority Verb Audit (Reference Solution)

Paired exercise: `lessons/mod-301-senior-ml-role-scope/exercises/exercise-01-seniority-verb-audit.md`.
Paired chapter: `03-seniority-verbs-in-postings.md`.

## 1. Solution overview

This exercise produces `verb-audit.md` — a per-posting classification plus a compressed senior-verb signature for every L30 posting. The reference solution below shows the intended structure, applies the six-family rubric to one fully-worked posting, and sketches the summary section across the five-posting sample.

The reference cannot ship real URLs — live postings churn and the source policy forbids inventing employer-specific facts. Instead this document contains two things:

1. A **template** you (the learner) fill in with your own five postings.
2. One **fully worked illustrative posting** built from the hypothetical bullets taught in chapter 03, labeled as illustrative so no claim is made about a real employer. Treat it as a graded calibration sample, not as a sixth posting in your audit.

The exercise is learning-authored because every learner will sample a different corpus. The grading rubric at the end is what calibrates all of those different corpora to the same bar.

## 2. Implementation (worked answer)

### 2.1 Template for each posting

Use this header block verbatim, once per posting:

```
Posting #N
Employer:
Exact title:
URL:
Date observed: YYYY-MM-DD
Location / remote?:
Represents: (AI-native / consumer / fintech / other — pick one)

Required bullets (verbatim):
- …
Preferred bullets (verbatim):
- …
```

Then for each posting, three artefacts:

1. The six-family classification table (one row per clause).
2. The counts block (Build+Operate count, lead-family count, ratio, distinct scope qualifiers).
3. The classification verdict + cited bullets + senior verb signature (if L30).

### 2.2 Fully worked posting (illustrative)

Below is **Posting #W** — a hypothetical composite assembled from the example bullets in chapter 03 of the paired lesson. It exists so you can see the graded shape of a correct audit row. It is clearly labeled `illustrative` and is not a sampled posting — do not count it toward your five.

```
Posting #W  (illustrative — composite from chapter 03, not a real employer)
Employer:         <illustrative>
Exact title:      Senior Machine Learning Engineer, Personalisation
URL:              <illustrative>
Date observed:    <illustrative>
Location / remote?: <illustrative>
Represents:       consumer

Required bullets (verbatim, broken into clauses):
R1. Build and deploy production ML models that power personalisation across the app.
R2. Own the technical direction of the personalisation ranking system end-to-end.
R3. Partner with the ML platform team on feature-store adoption.
R4. Raise the bar for evaluation rigour across the ranking pod.

Preferred bullets (verbatim):
P1. Mentor two mid-level ML engineers on the ranking pod.
P2. Set direction for the retraining roadmap through the next half.
```

Six-family classification table:

| # | Bullet (verbatim clause) | Family | Scope qualifier | Comment |
|---|---|---|---|---|
| R1 | Build and deploy production ML models that power personalisation across the app. | Build + Operate | `<no-scope>` | Standard L20 build/operate clause; no system-level qualifier. |
| R2 | Own the technical direction of the personalisation ranking system end-to-end. | Own + Set direction | system | System-scope Own + Set-direction is a genuine L30 signal (ch.03). |
| R3 | Partner with the ML platform team on feature-store adoption. | Lead | cross-team (peer platform) | Explicit peer-track hand-off; L30 cross-team signal. |
| R4 | Raise the bar for evaluation rigour across the ranking pod. | Mentor / raise the bar | programme (pod) | Mentor-family with explicit scope = pod → L30. |
| P1 | Mentor two mid-level ML engineers on the ranking pod. | Mentor / raise the bar | programme (cohort) | Mentor with concrete cohort — passes the "Mentor without scope" anti-pattern check. |
| P2 | Set direction for the retraining roadmap through the next half. | Set direction | programme (roadmap) | Set-direction attached to a decision surface, not a codebase. |

Counts block:

- Build/Operate bullets: 1 (R1).
- Lead-family bullets (Own / Lead / Mentor / Set-direction): 5 (R2, R3, R4, P1, P2).
- Ratio (lead : build): 5 : 1.
- Distinct scope qualifiers: `system`, `programme (pod)`, `programme (cohort)`, `programme (roadmap)`, `cross-team`.

Classification + evidence:

> **L30 (genuine senior).** Five lead-family clauses across four distinct scope qualifiers (R2 system; R3 cross-team; R4 pod; P1 cohort; P2 roadmap). The single Build/Operate clause (R1) is subordinate. No multi-team "across the org" qualifier, so this is not L40-mistitled.

Compressed senior verb signature:

> *Own the personalisation ranking system end-to-end, lead cross-team coordination with the ML platform team on feature-store adoption, mentor two mid-level ML engineers, partner with the ML platform team on feature-store adoption, set direction for the retraining roadmap through the next half.*

Slot fill: 5/5 (Own, Lead, Mentor, Partner, Set-direction). This is the shape of a well-formed L30 posting per chapter 03.

### 2.3 Summary section (sketch)

After the five per-posting writeups, finish with:

- A small table rolling up each posting's classification and the slot-fill count of its signature:

| Posting # | Represents | Classification | Signature slots filled | Most under-filled slot |
|---|---|---|---|---|
| 1 | AI-native | L30 | 5/5 | — |
| 2 | consumer | L20-in-disguise | n/a | Own has no system qualifier |
| 3 | fintech | L30 | 4/5 | `set direction` |
| 4 | health | ambiguous | n/a | copy-paste grab-bag |
| 5 | adtech | L40-mistitled | n/a | "roadmap across the org" |

- A 1–2 paragraph reflection. The chapter 03 hypothesis to test: `mentor` and `set direction` are the two slots the industry most often leaves `<missing>`. Whether your sample confirms or contradicts that is the load-bearing finding.

## 3. Validation steps

Before considering the audit done, walk through this checklist on the artefact:

1. **URLs resolve.** Click every URL in your five header blocks. If any returns 404, re-source or capture an Archive.org snapshot; a dead URL cannot be cited.
2. **Bullets are verbatim.** Diff one bullet at random against the live posting. Paraphrasing is the single most common silent failure — it moves the clause into the wrong family.
3. **Every lead-family bullet has a scope qualifier.** If a row's family is Own/Lead/Mentor/Set-direction and the qualifier column is blank or `<no-scope>`, you have under-classified: either the qualifier is there and you missed it, or the qualifier is genuinely absent and the comment column must flag the anti-pattern.
4. **Classifications cite bullets.** No row says "L30" without listing at least two bullet numbers in the evidence sentence.
5. **Signature slots match the posting.** For each L30 posting, re-read the extracted signature against the bullets you cited. Every slot either maps to a bullet or is marked `<missing>`.
6. **Title-filter was applied.** No posting in the sample has title keywords excluded by the exercise (Staff, Principal, Platform, MLOps, Research, Data Scientist, NLP, LLM, Applied AI). A Staff posting may appear only under the stretch-goal cross-track comparison, clearly labeled.
7. **Variety constraint holds.** The `Represents:` column across the five postings covers at least AI-native, consumer, fintech, and one domain you would not normally sample.

Static check:

```bash
# From the audit file's directory. Quick sanity on the structure.
grep -c "^Posting #" verb-audit.md        # expect 5 (or 6/7 with stretch)
grep -c "Classification:" verb-audit.md   # expect one per posting
grep -c "Signature:"      verb-audit.md   # expect one per L30 posting
```

## 4. Rubric / review checklist

A reviewer (peer or AI-grader) should verify the following. Anything unchecked is a reason to send the artefact back.

- [ ] Five posting header blocks, each with employer, exact title, URL, `date observed: YYYY-MM-DD`, and verbatim required/preferred bullets split into clauses.
- [ ] Title filter respected — no excluded keywords in the five sampled titles.
- [ ] Variety constraint met — at least one AI-native, one consumer, one fintech, one "domain you would not normally look at."
- [ ] Six-family classification table per posting, one row per clause, with the Family, Scope qualifier, and Comment columns populated.
- [ ] Every Own/Lead/Mentor/Set-direction row has a non-empty qualifier or an explicit `<no-scope>` plus an anti-pattern note in Comment.
- [ ] Counts block per posting: Build/Operate count, lead-family count, ratio, distinct scope qualifiers.
- [ ] Classification verdict per posting with ≥2 cited bullets.
- [ ] Compressed senior verb signature for every L30 posting, with `<missing>` markers where slots are unfilled.
- [ ] L20-in-disguise / L40-mistitled postings have a one-sentence note on which slot(s) the posting misuses, in lieu of a signature.
- [ ] Closing reflection (1–2 paragraphs) names which verb families the sample under-specifies and whether the learner would apply, with a stated reason.
- [ ] If stretch goals attempted: the Staff/mid-level comparison postings are clearly labeled and not counted toward the five.

Review-weighted points (for use when grading):

- Rubric application (classification table + qualifiers): 40%
- Classification verdict + cited evidence: 25%
- Senior verb signature correctness: 15%
- Reflection depth and honesty: 15%
- Hygiene (URLs, verbatim bullets, date observed): 5%

## 5. Common mistakes

- **Paraphrasing bullets.** Collapsing "Own the technical direction of the ranking system end-to-end" into "Owns the ranker" erases the scope qualifier (`end-to-end`) that drives the L30 classification. Keep clauses verbatim.
- **Classifying from the title.** A "Senior ML Engineer" title is a hypothesis to be tested by the verbs, not a conclusion. The whole point of the exercise is to catch titles that disagree with their bullets.
- **Dropping the qualifier column.** "Own" with no scope is meaningless (ch.03). If the posting leaves it off, your row must say `<no-scope>` and your Comment must flag it — that is itself the finding.
- **Letting one buzzword promote a posting.** A single "architect" or "mentor" verb does not make a posting L30. The threshold is at least four lead-family clauses across at least two distinct qualifiers.
- **Mentor without scope counted as a Mentor-family signal.** "Mentor junior engineers" with no cohort or ownership context is the anti-pattern — it usually means "answer Slack questions." Classify as `<no-scope>` and note it.
- **Set direction attached to a codebase.** "Set direction for the codebase" is a Build-family bullet in senior clothing. Only count Set-direction when it is attached to a roadmap, standard, or architecture.
- **Treating a grab-bag posting as five seniority signals.** A JD that asks for every verb family across every scope is probably a copy-paste. Mark it `ambiguous`, not L30.
- **Using the same employer-type for three postings.** Audit of five AI-native postings tells you about one slice of the industry; the variety constraint exists to expose that "senior" means different things across employer shapes.
- **Signature slots left blank with no `<missing>` marker.** If the posting does not fill `mentor`, the signature must say `mentor <missing>` — silence hides the finding.

## 6. References

Official standards cited for the overall module's source policy (not specifically load-bearing for this exercise, which is grounded in the paired chapter):

- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>
- MITRE ATLAS — <https://atlas.mitre.org/>
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>

Local context (authoritative for this exercise's rubric and taxonomy):

- Paired lesson: `lessons/mod-301-senior-ml-role-scope/exercises/exercise-01-seniority-verb-audit.md`.
- Chapter 03 — Seniority verbs in postings: `lessons/mod-301-senior-ml-role-scope/03-seniority-verbs-in-postings.md`. Source of the six-family taxonomy, scope-qualifier levels, the anti-pattern list, and the compressed senior verb signature form used throughout this solution.
- Chapter 02 — Altitude and ownership: `lessons/mod-301-senior-ml-role-scope/02-altitude-and-ownership.md`. Source of the L20/L30/L40 ownership axes referenced in the classification verdicts.

External career-ladder references (these are frameworks the learner can read if they want to translate between company-specific level names; cited by chapter 02):

- progression.fyi — public engineering career-ladder catalogue — <https://www.progression.fyi/>
- engineeringladders.com — composite public ladder — <https://www.engineeringladders.com/>
