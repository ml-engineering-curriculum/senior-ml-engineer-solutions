# exercise-02 — Build-vs-Adopt Tradeoff Doc (Reference Solution)

Paired exercise: `lessons/mod-308-platform-collaboration/exercises/exercise-02-build-vs-adopt-tradeoff-doc.md`.

## 1. Solution overview

This exercise produces `build-vs-adopt.md` — a short, decision-record-style document that a senior ML engineer writes when their team is facing one concrete choice: *for this capability, do we build it ourselves, adopt an existing paved road, fork a paved road and modify it, or contribute back to close a paved-road gap?* The document names the options, scores them against a fixed set of dimensions, picks one, and records why the other three were rejected with enough specificity that a reviewer six months later can tell whether the reasoning still holds.

The artefact has seven parts:

1. A **context** section that pins down what capability is being decided, what triggered the decision, and what the "do nothing" counterfactual is.
2. An **options panel** — build, adopt, fork, contribute-back — each with a one-sentence shape, a representative artefact, and the owner it implies.
3. A **decision dimensions table** — one row per option, one column per dimension (time-to-deliver, maintenance cost, team expertise, platform roadmap fit, blast radius, security posture, governance posture, lock-in and exit cost, strategic alignment). Each cell is specific (named weeks, named FTEs, named failure modes), not an adjective.
4. A **weighted score** — explicit weights per dimension, chosen before the scores (so weights are not back-fitted to a desired outcome), and a tallied recommendation.
5. A **recommendation** — the chosen option, followed by a short section titled "why not the others" that walks through each rejected option and names the honest reason it lost.
6. A **rollout and reversibility plan** — how the chosen option is executed, how long until it is live, and what signal would force a revisit (because decisions made in Q3 often look wrong in Q1 of the next year).
7. A **signatures** panel — the product owner, the tech-lead, the paved-road team (reviewer not approver), and any governance / security reviewer whose domain the capability touches.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric that calibrates every learner's tradeoff doc to the same bar.

The exercise exists because "build-vs-buy" (the outside-the-org framing) and "build-vs-adopt" (the inside-the-org framing) are different decisions. Inside an org, the opportunity cost of building locally is the capability the team *does not* build, the paved-road team's roadmap the team *does not* influence, and the technical debt the team *does* commit to maintain. Decisions written down against a fixed dimension set are decisions that can be revisited honestly; decisions made in a hallway are decisions nobody can defend six months later. NIST AI RMF GOVERN grounds this artefact as the governance instrument that makes a capability sourcing decision auditable; MANAGE grounds the rollout-and-reversibility plan as risk control on the decision.

## 2. Worked answer / implementation

### 2.1 Context

Six lines, in this order:

- **Capability under decision.** The single capability the doc is about. Pinned to a name. "Model monitoring" is not a decision; "a drift-monitoring layer that produces per-feature PSI daily, pages on-call when a feature crosses threshold, and writes a quarterly summary to the governance registry" is. If the capability spans multiple concerns, split the doc into one per concern — a doc that decides four things at once makes four decisions badly.
- **Decision trigger.** What forced this doc to be written now — an incident, an audit row (see `exercise-01`), a platform launch, a new regulatory requirement, a new product ask, a cost pressure. Decisions made without a trigger date tend to be made for the wrong reasons.
- **Decision deadline.** When the doc is signed off. A tradeoff doc with no deadline is open indefinitely, and "open indefinitely" ships whichever option the loudest voice preferred that week.
- **"Do nothing" counterfactual.** What happens if no decision is made. Often the option that wins by default in the absence of a doc, which is why the doc matters. If "do nothing" is viable, it belongs in the options panel as its own option, not as a hidden default.
- **Scope boundary.** What is in-scope and what is explicitly deferred. A build-vs-adopt doc that silently expands scope at readout is a doc that is debating four things when it meant to debate one.
- **Prior audit linkage.** The audit row (from `exercise-01`) or incident that identified this capability as in need of a decision. The doc is downstream of the audit.

### 2.2 Options panel

Four options, in a fixed order, so the reviewer can compare docs across teams:

1. **Build.** The team implements the capability itself, inside its own repo and under its own operational ownership. Representative artefact: a service / library / pipeline inside the surface team's repo. Owner: surface team.
2. **Adopt.** The team consumes the paved road as it exists today, through the supported interface. Representative artefact: a registered consumer entry (feature-store spec, model-registry client, eval-harness config). Owner: paved-road team (for the capability); surface team (for the integration).
3. **Fork.** The team copies the paved road's source or spec and modifies it for team-specific needs, maintaining the fork going forward. Representative artefact: a team-local repo of the modified platform. Owner: surface team (for the fork); paved-road team (for the un-forked upstream).
4. **Contribute-back.** The team authors an RFC (see `exercise-03`) and lands the change in the paved-road itself. Representative artefact: the RFC and the merged PR against the paved-road's repo. Owner: paved-road team (after merge); surface team (for the RFC and for the integration test).

Every option appears in the panel, even the obviously bad one. Reviewers reading the doc six months later need to see that the rejected option was *considered* and rejected on specific grounds, not silently omitted. A doc that lists only two options has already pre-biased the decision.

If one of the four is inapplicable — e.g., no paved road exists, so "adopt" and "fork" both reduce to "there is nothing to adopt" — the panel still lists it, with a one-line "inapplicable because" note. That note is often itself the signal that an action out of this doc is to escalate the absence of a paved road (ties to `exercise-01`'s `absent` row and action list).

### 2.3 Decision dimensions

A fixed set of dimensions, scored for each option. The dimensions below are the recommended minimum; teams may add one or two, but never drop one of these:

- **Time-to-first-value.** When the capability is in production and actually producing value — not when the PR is merged. Measured in weeks, with named assumptions about team capacity.
- **Ongoing maintenance cost.** Engineer-days per quarter after launch, including on-call load, dependency upgrades, documentation upkeep, and migration costs when the surrounding ecosystem changes. "Low" is not a cost; "~2 engineer-days/quarter and one on-call page per incident class" is.
- **Team expertise.** Whether the team has the skill to build and operate the chosen option. A "build" option whose team cannot operate it is a doc that will ship a system the team cannot maintain. Expertise is specific (e.g., "the team has one engineer with distributed-training experience") not generic ("we are a strong ML team").
- **Platform roadmap fit.** For `adopt` and `contribute-back`, how well the paved road's roadmap aligns with the team's trajectory — an adopt decision made against a paved road on the deprecation runway is a doc that ships a two-quarter solution. Fit is specific: "`feature-platform-v2` is on track to ship prompt versions in 2027-Q1 per their RFC; our need is 2027-Q2, so adoption leads build by one quarter."
- **Blast radius.** The scope of impact when the capability fails. A drift-monitoring service that pages one team has a smaller blast radius than one that pages ten teams. Blast radius is named in surfaces / users / pager hours.
- **Security posture.** What the option implies for the surface's security review (OWASP ML Top 10 threat classes, MITRE ATLAS adversarial tactics). An option that forks a paved road inherits the paved road's threat model and the team's responsibility for maintaining the fork's security posture. Named explicitly: which threat class does the option change exposure to?
- **Governance posture.** What the option implies for the surface's governance review (NIST AI RMF GOVERN function categories, model-card and data-statement obligations, review cadence). An adopt decision inherits the paved road's governance; a build or fork does not.
- **Lock-in and exit cost.** How hard it is to reverse the decision. A build commits the team to a codebase; an adopt commits the team to a platform's API surface; a fork commits to both. Exit cost is named in engineer-months of migration if the decision is reversed in a year.
- **Strategic alignment.** Whether the option is consistent with the surface team's charter and the platform team's charter. A build decision that duplicates a platform's roadmap is a strategic conflict even if it scores well on other dimensions; the doc names the conflict rather than hiding it.

Each cell is a short sentence, not an adjective. "Fork option, security posture: inherits paved road's threat model; team becomes responsible for tracking upstream CVEs in the forked dependency set — expect one upstream CVE every 1–2 quarters based on paved-road history" is a cell. "High risk" is not a cell.

### 2.4 Weighted score

Weights are declared *before* the scores are written, and ideally before the scores are known. Weights represent the team's stated priorities for this decision and should be defensible without reference to any specific option.

Suggested baseline weights (sum = 100) for a production ML surface's capability-sourcing decision:

- Time-to-first-value: 10
- Ongoing maintenance cost: 20
- Team expertise: 10
- Platform roadmap fit: 15
- Blast radius: 10
- Security posture: 10
- Governance posture: 10
- Lock-in and exit cost: 10
- Strategic alignment: 5

These are starting points. A pre-launch surface weighs time-to-first-value higher; a mature, regulated surface weighs governance and security posture higher; a surface built on a paved road that is close to deprecation weighs platform roadmap fit higher. The doc records the weights and the reason they deviate from the baseline, if they do. Changing the weights at readout to produce the desired recommendation is the primary failure mode here; declare-before-you-score is the discipline.

Scoring per option, per dimension, on a 1–5 scale:

- 5 = this option is a clear best on this dimension.
- 4 = above average.
- 3 = average / acceptable.
- 2 = below average / costly.
- 1 = this option is a clear worst on this dimension.

The weighted score per option = sum over dimensions of (weight × score / 5). Rounding to one decimal is enough; a tenth of a point of difference between options is not the signal, the per-cell reasons are.

Example scoring table (illustrative; see §2.8 for a full worked example):

| Dimension | Weight | Build | Adopt | Fork | Contribute-back |
|---|---|---|---|---|---|
| Time-to-first-value | 10 | 3 | 5 | 4 | 2 |
| Ongoing maintenance | 20 | 1 | 5 | 2 | 4 |
| Team expertise | 10 | 3 | 5 | 3 | 3 |
| Platform roadmap fit | 15 | 2 | 5 | 2 | 4 |
| Blast radius | 10 | 3 | 4 | 3 | 4 |
| Security posture | 10 | 2 | 4 | 2 | 4 |
| Governance posture | 10 | 2 | 5 | 2 | 4 |
| Lock-in / exit cost | 10 | 3 | 4 | 2 | 4 |
| Strategic alignment | 5 | 2 | 4 | 2 | 5 |
| **Total (weighted, out of 100)** | | **48** | **91** | **48** | **76** |

A difference of 15+ points usually names a clear winner; a difference under 5 points usually says the doc is not yet sharp enough — either the weights are wrong, the cells are vague, or the options have collapsed onto each other. A close score is a signal to re-write the dimensions, not to flip a coin.

### 2.5 Recommendation

One option is chosen. The recommendation is stated as a sentence ("We recommend adopting `feature-platform-v2` for this capability as of 2026-11-20 subject to the rollout plan in §2.6").

Followed by **"why not the others"** — a short paragraph per rejected option naming the honest reason it lost on this decision. The failure modes to avoid:

- "Why not Build" that just says "Adopt scored higher." The reason is a sentence from the dimensions: "Build was rejected primarily on maintenance cost (20% weight, score 1): a bespoke drift-monitoring service implies ~1 engineer-day/quarter per feature family and the team already carries two bespoke services."
- "Why not Fork" that reads like a general objection to forks. The reason is specific: "Fork was rejected on platform roadmap fit: upstream is on a monthly release cadence with breaking changes to the harness API in two of the last three quarters; tracking the fork would cost the team ~0.5 FTE per quarter indefinitely."
- "Why not Contribute-back" that silently assumes contribution is unavailable. The reason is often "accepted but out-of-cycle": the paved-road team is open to the contribution but the capability is needed before the next platform release train; contribute-back is the *second-stage* action after this decision is made (see §2.7).

The recommendation section does *not* re-litigate the weights. If the weights were wrong, the correction goes in §2.4 and the scores are redone. Challenging the recommendation without challenging the weights is debating the output while hiding the input.

### 2.6 Rollout and reversibility plan

A decision without a rollout is a wish. The rollout plan names:

- **First milestone.** The smallest deliverable that proves the chosen option works. For an adopt decision: a single consumer of the paved road is wired up and running in staging. For a build decision: the MVP shape of the local implementation is defined and a first slice is in staging.
- **Target date.** When the first milestone is live.
- **Owner.** The role that owns delivery, named in `exercise-01`'s owner column convention.
- **Success criterion.** What signal says the first milestone succeeded. Not "it works"; a specific signal — an SLO holding for a week, a slice-level eval passing, a quarterly cost bound met.
- **Reversibility triggers.** The conditions under which the decision is revisited. For an adopt decision: paved-road team announces deprecation, SLA degrades, the team's capability requirements grow beyond what the paved road can deliver. For a build decision: a new paved road launches that covers the need, the maintenance cost exceeds the budget set in the doc, the team re-organizes.
- **Scheduled re-review.** A date — typically 1–2 quarters out — on which the decision is explicitly re-read. Scheduled re-reviews are rare in practice; they are also where most decisions quietly drift. Scheduling one is cheap insurance.

### 2.7 Signatures and next-step linkage

The doc is signed off by, at minimum:

- **Product owner** of the surface. The capability is being sourced on behalf of the surface's users; the product owner signs the rationale.
- **Tech-lead** of the surface team. The tech-lead is on the hook for the operational cost in §2.3 and the rollout in §2.6.
- **Paved-road team reviewer.** If the decision is `adopt`, `fork`, or `contribute-back`, the paved-road team is a *reviewer* — they see the doc, comment on roadmap fit and interface choices, and sign as "reviewed, not approver." The paved-road team is not an approver of the surface team's sourcing decision; they are an accountable stakeholder for the integration shape.
- **Security review** (if the capability has an attack surface).
- **Governance review** (if the capability has a governance-visible output — model-card data, fairness slice, retention surface).

The signatures block records role + date, not individual + date-of-review. Roles survive rotations; names do not. A decision whose signatures are stale (role has rotated and the new occupant has not acknowledged) is a decision in drift.

If the chosen option is `adopt`, the next-step linkage is the integration backlog in the surface team's tracker. If the chosen option is `contribute-back`, the next step is the RFC authored in `exercise-03` — the tradeoff doc is the preamble of the RFC and should link to it as soon as the RFC is drafted. If the chosen option is `build` or `fork`, the next step is the design doc for the build / the fork-maintenance plan, which lives in the surface team's repo.

### 2.8 Illustrative worked example (not a real system)

> **Illustrative decision** — `team-agent-assist` is deciding how to source a drift-monitoring capability for `agent-assist/draft`. The audit from `exercise-01` flagged the drift-monitoring row as `absent` (no org-wide paved road), with a bespoke nightly drift job currently running. The decision deadline is 2026-12-15. This example is a composite for calibration; do not treat it as a real system.

Context:

- Capability: drift-monitoring for `assist-llm-v4`'s prompt-feature distribution, producing a daily PSI value per feature family, paging on-call when a feature crosses threshold (as defined in `mod-307/exercise-01`'s drift SLI), and writing a quarterly summary to the governance registry.
- Decision trigger: the paved-road audit (2026-11-15) flagged `absent` for drift monitoring and a P2 action to escalate the absence. Separately, `mod-307/exercise-01`'s SLO document declared a drift SLI with a 98% target; the SLI needs a monitoring surface.
- Decision deadline: 2026-12-15 (ahead of the Q1 release train).
- "Do nothing" counterfactual: the bespoke nightly job continues; its output is a single dashboard panel; no org-wide capability emerges.
- Scope: drift-monitoring for the prompt-feature family only; downstream drift on the retrieval index is out of scope for this doc.
- Prior audit linkage: `paved-road-audit.md`, row 5 (`absent`), 2026-11-15.

Options panel:

1. **Build** — expand the current bespoke nightly job into a team-grade service with alerting, dashboard, and governance writeback. Owner: `team-agent-assist`.
2. **Adopt** — *(inapplicable: no org-wide paved road exists)*. Listed for completeness; the inapplicability itself is the signal to escalate (see action P2 in §2.7).
3. **Fork** — fork a nearby open-source drift-monitoring project (e.g., an open-source distribution-shift library) and host a team-local instance. Owner: `team-agent-assist` + tracked upstream.
4. **Contribute-back** — none applicable (no paved-road team to contribute to). *(inapplicable; listed for completeness; the inapplicability is the signal to escalate, same as option 2.)*

Weights (as of 2026-11-20; deviation from baseline noted):

- Time-to-first-value: 10 (baseline)
- Ongoing maintenance: 25 (above baseline — this decision commits to a long-lived maintenance surface and the team already carries two bespoke services; cost is weighted higher)
- Team expertise: 10
- Platform roadmap fit: 5 (below baseline — no platform exists, so there is no roadmap to fit to)
- Blast radius: 10
- Security posture: 10
- Governance posture: 15 (above baseline — drift signals feed the governance registry and the governance surface is what makes drift legible to leadership)
- Lock-in / exit cost: 10
- Strategic alignment: 5

(Weights sum to 100. Deviations argued explicitly.)

Scoring:

| Dimension | Weight | Build | Fork |
|---|---|---|---|
| Time-to-first-value (4 weeks vs 6 weeks) | 10 | 4 | 3 |
| Ongoing maintenance | 25 | 2 (1 engineer-day/quarter) | 2 (upstream drift + local mod maintenance ~0.5 FTE/quarter) |
| Team expertise (team has one engineer with drift experience) | 10 | 3 | 2 |
| Platform roadmap fit | 5 | 3 (no platform exists; neutral) | 3 (no platform exists; neutral) |
| Blast radius (single team) | 10 | 3 | 3 |
| Security posture (team-owned data exposure) | 10 | 3 (fully internal) | 2 (OSS dependency widens supply-chain surface per OWASP ML Top 10 ML06) |
| Governance posture (feeds governance registry) | 15 | 3 (team-authored writeback) | 2 (OSS writeback schema diverges from org standard) |
| Lock-in / exit cost | 10 | 3 (replaceable when a platform emerges) | 2 (forked code is harder to retire than team code) |
| Strategic alignment (matches team charter?) | 5 | 3 | 2 |
| **Total (weighted)** | | **56** | **46** |

Options 2 and 4 are inapplicable (no paved road exists) and are recorded as inapplicable; the doc notes this inapplicability as the key input to the escalation action.

Recommendation:

> We recommend **Build** for this capability as of 2026-12-15, subject to the rollout plan below. The decision is paired with an **escalation action** (P2 from the audit): the team sponsors a proposal to the platform-review council for an org-wide drift-monitoring paved road, with the current build to serve as a candidate reference implementation. If a paved road emerges in 2027, the team re-runs this doc and expects the recommendation to flip to `adopt`.
>
> **Why not Adopt or Contribute-back.** Both are inapplicable because no paved road exists. The inapplicability is itself the signal that leadership action is required; the escalation is paired with the build decision so that the local capability does not quietly become a permanent substitute for platform work.
>
> **Why not Fork.** Fork lost primarily on security posture (OSS dependency increases supply-chain surface per OWASP ML Top 10 ML06) and governance posture (the OSS writeback schema diverges from the org's governance-registry contract). The marginal time-to-first-value gain is not worth the ongoing supply-chain and governance tax, and the fork's upstream has a monthly breaking-change cadence that would consume the team's maintenance budget faster than a team-authored build.

Rollout and reversibility plan:

- **First milestone.** A single feature-family drift signal (prompt-length PSI) computed daily, written to the SLO dashboard from `mod-307/exercise-01`, and firing a test alert through the paved-road paging platform.
- **Target date.** 2027-02-01.
- **Owner.** `team-agent-assist` L30.
- **Success criterion.** The drift SLI is observable on the dashboard for 14 consecutive days, with at least one synthetic drift test that correctly triggers a (silent) alert.
- **Reversibility triggers.** A paved-road drift-monitoring platform launches, the maintenance cost exceeds 1.5 engineer-days/quarter (the budget declared in §2.3's dimensions), or the surface changes materially (new feature family type, new model class).
- **Scheduled re-review.** 2027-05-15 (end of Q1), coinciding with the next paved-road audit.

Signatures:

- Product owner (`team-support-ops` lead): — signed 2026-12-14
- Surface tech-lead (`team-agent-assist` lead): — signed 2026-12-14
- Security review (`team-security`): — reviewed (not approver) 2026-12-13
- Governance review (`team-governance`): — reviewed (not approver) 2026-12-13
- Platform-review council sponsor (for the paired escalation): — ack'd 2026-12-12

(Roles in the illustrative sign-off block are stand-ins; a real doc records the role names current in the org.)

### 2.9 Optional template: `build-vs-adopt-template.md`

A recommended on-disk template that teams start from. The template is the artefact that keeps the per-team decisions comparable; variance in format invites variance in rigour.

```markdown
# <Capability name>: Build-vs-Adopt Decision

## 1. Context
- Capability under decision:
- Decision trigger (incident / audit row / platform launch / regulatory / product ask):
- Decision deadline:
- "Do nothing" counterfactual:
- Scope boundary:
- Prior audit / incident linkage:

## 2. Options panel
- Build: <one-sentence shape, representative artefact, owner>
- Adopt: <one-sentence shape, representative artefact, owner>
- Fork: <one-sentence shape, representative artefact, owner>
- Contribute-back: <one-sentence shape, representative artefact, owner>

(For any inapplicable option, write "inapplicable because <reason>" — do not omit.)

## 3. Decision dimensions
<table — dimensions as rows, options as columns, each cell a specific sentence>

## 4. Weights and scoring
Weights declared on <date>, before scoring.

<table — dimension × weight × per-option 1–5 score; final weighted totals>

## 5. Recommendation
We recommend <option> as of <date>.

### Why not the others
- Why not Build: <specific reason from dimensions>
- Why not Adopt: <specific reason from dimensions>
- Why not Fork: <specific reason from dimensions>
- Why not Contribute-back: <specific reason from dimensions>

## 6. Rollout and reversibility
- First milestone:
- Target date:
- Owner:
- Success criterion:
- Reversibility triggers:
- Scheduled re-review:

## 7. Signatures
- Product owner (role):
- Surface tech-lead (role):
- Paved-road team reviewer (role, if applicable):
- Security review (role, if applicable):
- Governance review (role, if applicable):

## 8. References
- Paved-road audit row: <link>
- NIST AI RMF function reference (GOVERN / MANAGE):
- OWASP ML Top 10 / MITRE ATLAS references (if security dimension applies):
- Paved-road team RFC if `contribute-back` or if `adopt` depends on an un-shipped RFC:
```

## 3. Validation steps

Walk the tradeoff doc through this checklist before considering it signed off:

1. **Context names a single capability.** If the capability has three parts, the doc is three docs.
2. **Decision trigger and deadline are dated.** A tradeoff doc with no deadline drifts; one with no trigger was probably written for the wrong reason.
3. **"Do nothing" counterfactual is listed** — often in the options panel as a fifth option when it is genuinely viable.
4. **All four options appear.** Inapplicable options are listed with a one-line inapplicability note, not omitted.
5. **Decision dimensions include all nine recommended rows** (or team-added dimensions clearly argued in the weights section).
6. **Each cell is a specific sentence**, not an adjective. "Low maintenance" fails this check; "~0.5 engineer-day/quarter after launch" passes.
7. **Weights sum to 100** and were declared before scoring. Deviations from the baseline are argued explicitly.
8. **Scoring uses a 1–5 scale** with the per-cell reason underlying the score visible.
9. **Weighted totals are computed** (`sum(weight × score / 5)`) and the margin of victory is meaningful (>5 points).
10. **Recommendation is stated as a sentence** with a date and a subjection to the rollout plan.
11. **"Why not" paragraphs are specific to the dimensions**, not generic objections.
12. **Rollout plan has first milestone, target date, owner, success criterion, reversibility triggers, and scheduled re-review.**
13. **Signatures list roles, not individuals.** Dated per signer.
14. **NIST AI RMF GOVERN cited** for the artefact-as-governance framing; MANAGE cited for the rollout-and-reversibility-as-risk-control framing.
15. **OWASP ML Top 10 and MITRE ATLAS cited** on the security-posture dimension, if the capability has an attack surface.
16. **Cross-reference to `exercise-01`'s audit row** that triggered this decision, and to `exercise-03` if the chosen option is `contribute-back`.

Static checks a CI step can enforce on `build-vs-adopt.md`:

```bash
grep -ciE "build|adopt|fork|contribute"                    build-vs-adopt.md   # options named
grep -ciE "time-to|maintenance|expertise|roadmap|blast|security|governance|lock-in"  build-vs-adopt.md   # dimensions
grep -ciE "weight"                                         build-vs-adopt.md   # weights declared
grep -ciE "recommend|why not"                              build-vs-adopt.md   # recommendation + rejected
grep -ciE "rollout|milestone|re-review|reversib"           build-vs-adopt.md   # rollout + reversibility
grep -ciE "nist ai rmf|govern|manage"                      build-vs-adopt.md   # standards cited
```

## 4. Rubric / review checklist

- [ ] Context names capability, trigger, deadline, do-nothing counterfactual, scope, and prior-audit linkage.
- [ ] Options panel lists all four options with one-sentence shapes and owners — inapplicable options labeled and reasoned, not omitted.
- [ ] Decision-dimensions table has all nine recommended rows (or team-added dimensions argued in the weights section).
- [ ] Every cell is a specific sentence with named weeks, FTEs, failure modes, or risk classes — no adjective-only cells.
- [ ] Weights sum to 100, are declared before scoring, and deviations from the baseline weights are argued explicitly.
- [ ] Scores on a 1–5 scale with per-cell reasons legible; weighted totals computed; margin of victory ≥ 5 points or the doc is re-sharpened.
- [ ] Recommendation stated as a dated sentence with reference to the rollout plan.
- [ ] "Why not" paragraphs are specific to the dimensions that caused each option to lose.
- [ ] Rollout and reversibility plan has first milestone, target date, role-owner, success criterion, reversibility triggers, and a scheduled re-review date.
- [ ] Signatures record roles (product owner, surface tech-lead, paved-road reviewer, security reviewer, governance reviewer) with dates.
- [ ] If `contribute-back` is chosen, linked to the RFC in `exercise-03`; if `build` is chosen, linked to a design doc; if `adopt` is chosen, linked to the integration backlog item.
- [ ] NIST AI RMF GOVERN cited for the artefact framing; MANAGE cited for the rollout and reversibility risk-control framing.
- [ ] OWASP ML Top 10 and MITRE ATLAS cited on the security-posture dimension where the capability has an attack surface.
- [ ] Prior-audit linkage (`exercise-01`) is explicit — the doc is downstream of the audit, not standalone.

Review-weighted points for grading:

- Context and option-panel discipline (single capability, four options, no silent omissions): 15%
- Decision-dimensions coverage and per-cell specificity: 25%
- Weights (declared pre-scoring, sum to 100, deviations argued): 15%
- Recommendation (stated as sentence) and "why not" per rejected option: 15%
- Rollout and reversibility plan (first milestone, owner, success criterion, triggers, re-review): 15%
- Standards alignment (NIST AI RMF GOVERN / MANAGE, OWASP ML Top 10 / MITRE ATLAS if applicable) and signatures: 15%

## 5. Common mistakes

- **Two options, not four.** A doc that silently drops `fork` or `contribute-back` has pre-biased the decision. The honest version lists all four, even if two are inapplicable.
- **Adjective-only cells.** "High cost" is not a cell; "~2 engineer-days/quarter and one on-call page per release" is a cell. Adjectives are where the decision-doc discipline evaporates.
- **Weights set after the scores.** The whole point of weights is to pre-commit to what matters; back-fitting weights to produce the preferred option is the decision-doc anti-pattern. Declare weights with a date and keep the git history.
- **Close scores decided by vote.** A < 5-point margin is a signal that the dimensions are blurry or the weights are wrong, not that the group should vote. Re-sharpen first.
- **"Why not" paragraphs that re-litigate the weights.** The weights were declared. If they were wrong, redo §2.4; do not argue the output while hiding the input.
- **Rollout plan that is a wish.** "We'll adopt in Q1" is not a rollout. First milestone + date + owner + success criterion + reversibility triggers is a rollout.
- **No reversibility plan.** Every decision will look different in six months. A doc with no reversibility trigger is a doc that is permanent by default, which is almost never what the team wants.
- **Scheduled re-review that never happens.** The re-review date is on someone's calendar, or it did not happen. The doc records who owns the re-review.
- **Signatures on individuals, not roles.** Individuals move; roles rotate. "Product owner (role: `team-support-ops` lead)" is durable.
- **"Build" that silently assumes team expertise.** A build decision is a maintenance commitment. If the team's expertise cell says "one engineer with drift experience," the maintenance cost cell must reflect that one engineer's departure risk.
- **"Adopt" without the paved road's roadmap.** Adopting a platform on its deprecation runway is a doc that will be rewritten in two quarters. The platform roadmap fit dimension is where that failure mode is caught.
- **Ignoring strategic alignment.** A build decision that duplicates a paved-road team's roadmap is a strategic conflict even if it scores well elsewhere. The strategic-alignment dimension is where that conflict is surfaced rather than hidden.
- **No linkage back to the audit.** The audit (`exercise-01`) identified the decision; the audit's row is the doc's preamble. A tradeoff doc that stands alone was probably written for the wrong reason.
- **Forgetting the escalation when all options are inapplicable.** If `adopt` and `contribute-back` are both inapplicable because no paved road exists, the doc pairs the recommendation with an escalation. Otherwise the local build silently becomes a permanent substitute for platform work.
- **No governance review on a governance-visible capability.** Capabilities whose outputs appear in the governance registry (model cards, drift summaries, fairness slices) need a governance-reviewer signature, not an "FYI."
- **No security review on an attack surface.** Capabilities that touch user data, model weights, or external I/O need a security-reviewer signature. "It's internal-only" is not a reason to skip the signature.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds the tradeoff doc as the governance instrument that makes a capability sourcing decision auditable. MANAGE function grounds the rollout-and-reversibility plan as risk control on the decision, including the scheduled re-review.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Cited on the security-posture dimension for the threat classes each option exposes or mitigates (e.g., ML06 AI supply-chain risk for `fork` of OSS; ML04 membership inference for exposures a `build` option carries).
- MITRE ATLAS — <https://atlas.mitre.org/>. Cited on the security-posture dimension for the adversarial tactics (model extraction, poisoning, prompt injection) each option exposes or mitigates.

Local context (authoritative for the doc's shape and for the cross-references):

- Paired lesson: `lessons/mod-308-platform-collaboration/exercises/exercise-02-build-vs-adopt-tradeoff-doc.md`.
- Module README: `modules/mod-308-platform-collaboration/README.md`.
- Related exercise (paved-road audit whose `absent`, `bypassed`, `forked`, and `partial` rows feed this decision): `modules/mod-308-platform-collaboration/exercise-01-paved-road-consumption-audit/SOLUTION.md`.
- Related exercise (contribute-back RFC that is the follow-on when the recommendation is `contribute-back`): `modules/mod-308-platform-collaboration/exercise-03-contribute-back-rfc-to-platform/SOLUTION.md`.
- Related exercise (role hand-off map whose peer-platform category is the structural counterpart to the options panel): `modules/mod-301-senior-ml-role-scope/exercise-02-role-hand-off-map/SOLUTION.md`.
- Related exercise (SLO document whose monitoring contract many capability-sourcing decisions sit on top of): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (cost-budgets artefact whose cost / quota implications shape the maintenance-cost dimension): `modules/mod-307-ml-reliability-slos/exercise-02-cost-budgets-and-quotas/SOLUTION.md`.
- Related exercise (offline evaluation harness that is the typical "adopt" candidate for the evaluation capability): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
