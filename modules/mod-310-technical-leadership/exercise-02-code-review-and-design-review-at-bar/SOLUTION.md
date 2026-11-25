# exercise-02 — Code Review And Design Review At Bar (Reference Solution)

Paired exercise: `lessons/mod-310-technical-leadership/exercises/exercise-02-code-review-and-design-review-at-bar.md`.

## 1. Solution overview

This exercise produces two paired artefacts a senior ML engineer uses to raise the review bar on their team:

1. `code-review-rubric.md` — the rubric a senior reviewer applies to any non-trivial ML change (training pipeline, serving code, feature code, eval harness, governance scaffolding). It names what reviewers look for, the order they look for it, and the standard of care that distinguishes an "LGTM" from a review that caught the bug a week before it paged someone.
2. `design-review-rubric.md` — the rubric a design-review meeting (or asynchronous design-doc thread) applies to any ML design doc (new model, new serving topology, new eval harness, new training pipeline, new paved-road integration). It names the shape of a design that can be reviewed, the questions the reviewers ask, and the criteria by which a design is "approved," "approved with changes," or "sent back."

Both artefacts carry one worked example (a code-review example that walks through a realistic diff; a design-review example that walks through a realistic design doc) and the grading rubric that calibrates "senior-level review" across the team.

The two rubrics are paired because they operate on the same system at different altitudes. A senior reviewer who is a terror on code review but passive in design review is a senior reviewer who catches the implementation bugs and misses the architecture bugs. The design is where the expensive mistakes are made; code review is where they are discovered once they have already been paid for. Both reviews exist; the senior ML engineer sets the bar for both.

The reference solution below shows the structure, works a labeled illustrative diff and design doc through each rubric, and gives the grading rubric for assessing the artefacts themselves.

The exercise exists because setting a review bar is one of the main ways a senior ML engineer multiplies their effect across a team without becoming a bottleneck. A rubric that is explicit is a rubric a mid-level engineer can self-apply before sending the review request; a rubric that lives only in the senior's head is a review that stalls when the senior is on leave. NIST AI RMF GOVERN grounds both artefacts as the governance instruments that make "review" a legible and auditable step of the ML lifecycle; MAP grounds the design-review risk framing; MEASURE grounds the exit criteria for approving a design or a change; MANAGE grounds the escalation / pushback / rework loop.

## 2. Worked answer / implementation

### 2.1 The code-review rubric

The rubric reads top-to-bottom and the reviewer applies it in order. Reading order matters: by the time the reviewer is looking at style, they have already decided whether the change is safe to land, and style comments do not block landing.

**§1. Change framing (read before looking at any code).**

- **Does the change have a one-line "why."** The PR description's first line answers *why* the change exists. "Fixes bug" is not a why; "clamps feature `X` to prevent the training/serving skew that surfaced in incident `IR-2026-114`" is. If the why is missing, the reviewer asks for it before continuing — a review without a why is a review where the reviewer and the author have different models of what is being decided.
- **Is the change scoped to one concern.** A diff that fixes a bug, refactors a module, and adds a new feature is three reviews stacked on top of each other, and reviewers approve all three or none. Split-as-a-request is a legitimate outcome of the rubric's §1.
- **Is the change tied to a design doc (if required).** For changes that touch architecture, add a new paved-road dependency, modify a Model Card (`mod-309/exercise-01`), or change an SLO (`mod-307/exercise-01`), a design doc is required — the review process demands an upstream design review before code review (§2.2 below). A sufficiently large change without a design doc is sent back, not approved.
- **Is the test plan explicit.** How the author verified the change — unit tests, integration tests, offline eval delta, A/B shadow, local manual check with repro steps. "I ran the script" is not a test plan. Reviewers should be able to re-run the plan, not just trust it.

**§2. Correctness (the first-order review).**

The reviewer reads the diff hunk-by-hunk, with the following lens. ML-specific correctness is the senior reviewer's specialty and the first place they add value over a generalist reviewer.

- **Data flow.** Is the shape, dtype, and semantics of every tensor / dataframe / schema preserved through the change? An ML bug almost always lives at a boundary where a `float32` became `int32`, a `(N, K)` became `(K, N)`, a prediction became a probability-of-a-different-class, or a feature name changed and downstream consumers were not re-pointed.
- **Training/serving skew.** Does the change touch feature computation, normalization, or tokenization? If yes, the reviewer asks for proof the training and serving paths produce byte-identical outputs on a fixed set of probe inputs. A change to feature computation without this proof is a change that will cause the next outage.
- **Numerical correctness.** For loss functions, metrics, aggregations, normalization: does the change respect numerical stability (log-sum-exp, epsilon placement, dtype of accumulators)? ML numerical bugs are silent and expensive; a senior reviewer tests these more aggressively than a junior one.
- **Edge cases.** Empty batch, single-sample batch, all-null-feature row, out-of-vocab token, duplicate id in a join, a shard with zero rows. ML pipelines are full of these; a senior reviewer asks the author to show the test case before merging.
- **Reproducibility.** Is a randomness source added, changed, or removed? Seeds documented? If a seed is not controlled, say so and name why it's acceptable. The reviewer is defending the invariant that the eval is re-runnable.
- **Metrics change gates.** If the diff changes a metric definition (not its implementation but its *meaning*), the change is a Model-Card-affecting change, not a code change, and it goes back through §2.2 design review.
- **Rollback.** Can the change be reverted cleanly? A change that cannot be rolled back (schema migration, state-full compute, retroactive data rewrite) is flagged `irreversible` in the review and goes through a higher standard of care (see `mod-310/exercise-01` §7).

**§3. Safety, security, and governance.**

- **Secrets.** The diff does not add secrets, credentials, personal access tokens, or PII to source. The reviewer runs the standard secret-scan grep over the diff; if the pre-commit hook did not run, the reviewer re-runs it.
- **Dependency hygiene.** New dependencies are named, version-pinned, and justified in the PR. A new third-party dependency triggers a license-review question — the licence belongs in a short line of the PR.
- **PII / consent boundary.** If the diff touches training-data construction or feature extraction, the reviewer asks "is the PII register (`mod-309/exercise-02`) still correct for this change?" If no, the Data Statement must be updated before merge.
- **OWASP ML Top 10.** The reviewer scans for known ML-specific threat categories touched by the change: input manipulation (new input path without a validator), data poisoning (new data source), model inversion / membership inference (new output path exposing internals), model theft (new unauthenticated serving edge), supply-chain (new dependency), transfer-learning (new base-model use), model skewing (new feedback-loop training signal), output-integrity (new downstream consumer that trusts model output as-is), denial of ML service (new unbounded compute path).
- **MITRE ATLAS mapping.** For generative surfaces, the reviewer asks whether the change opens a new ATLAS tactic exposure (e.g., prompt injection, data exfiltration via inference API, jailbreak evasion). Changes that touch system prompts, tool-use wiring, or RAG grounding are flagged for cross-review by `team-security` and cross-referenced to `mod-309/exercise-04`.
- **NIST AI 600-1 risks.** For generative surfaces, is the change in scope for the Model Card's §8 risk table? If yes, the Model Card refresh is a required follow-up.
- **Launch-gate alignment.** Does the change move a metric in a way that requires an updated launch gate (`mod-305/exercise-01` harness, `mod-306` experiment)? If yes, the change does not land without the gate being updated.

**§4. Observability and operability.**

- **Logging.** New error paths have named log lines with named severity. Un-logged silent failures are the operational bugs that surface three months later.
- **Metrics.** New behaviour is covered by existing dashboards, or the diff adds the dashboard panel. A change that moves an SLI must be attached to a runbook update.
- **Rate-limited / cost-bounded calls.** New external calls (model API, feature-store RPC, logging sink) are rate-limited and bounded. "What is the p99 cost of this change at 10× traffic" is a reviewer question.
- **On-call runbook.** New failure modes update the runbook. A diff that adds an alert without updating the runbook makes somebody's 2am worse.

**§5. Tests.**

- **Unit tests.** Added where meaningful; the reviewer does not demand 100% coverage but demands coverage of the behaviours the diff is making a claim about. A one-line change that fixes a known bug ships with the regression test.
- **Integration / eval tests.** For changes that touch eval, offline harness or shadow-eval delta is required. "I didn't see the eval move" is not sufficient; "offline eval on `eval-ds-v4` shows schema-validity +0.1%, acceptance-rate within noise band, no slice regression > 2%" is.
- **Flaky tests.** The diff does not introduce a flaky test. If the test is inherently non-deterministic, seed it or mark it `flaky` in the suite config and file the follow-up.
- **No disabled tests silently.** A test disabled without an explanation and a tracking issue is a test disabled forever.

**§6. Readability and local quality.**

Only read this section after §2–§5 are green. Style never blocks landing on its own; it is bundled into the follow-up PR if the correctness review has approved.

- **Names.** Functions / variables / files have names that read as what they do.
- **Comments.** Follow the project norm: a comment explains *why* (hidden constraint, surprising workaround), not *what*. The reviewer pushes back on `what` comments.
- **Dead code.** The diff does not add commented-out blocks, placeholder `# TODO` stubs without an owner, or unused imports.
- **Scope of helpers.** A helper added for the diff's one use site lives next to the use site; a helper that would be re-used across modules is moved to the module home.
- **Formatting.** CI must be green; reviewer does not hand-check formatting.

**§7. Approval call.**

Reviewers have exactly three options:

- `LGTM` — approve to land. The reviewer attests that §1–§5 are green and §6 is at team standard.
- `Approve with changes` — approve to land conditional on named, listed changes. Each change is specific ("rename `fn_a` to `fn_b`"), not aspirational ("make it cleaner").
- `Changes requested` — the review has found a §1–§5 issue that prevents the change from landing as-is. The comment names the issue and names what would resolve it. A "changes requested" without a resolution path is a review that isn't helping.

An opinionated norm: `Approve with changes` is the common outcome. Both `LGTM` with no comments (did the reviewer actually read?) and `Changes requested` with a single-word "No" (is the author getting actionable feedback?) are review smells. The senior reviewer's goal is to teach the next reviewer how to say "approve with changes" well.

### 2.2 The design-review rubric

Design review runs before code review on the same system. A design doc that passes design review licenses the author to open PRs; a design doc that fails goes back for revision. The rubric reads in the same top-to-bottom style.

**§1. Shape of a reviewable design doc.**

Before the rubric evaluates the design, the reviewer confirms the doc has these parts. A doc missing any of them is sent back for structure, not evaluated on content (the content review wastes everyone's time if the doc's shape is wrong):

- **Problem statement and thesis.** What the design solves and the measurable outcome. Mirrors `mod-310/exercise-01` §1.
- **Context and scope.** The system boundary, what is in-scope, and what is explicitly out-of-scope.
- **Proposed design.** The design itself — architecture diagram, data flow, interfaces, data model, serving topology. ML-specific: training data, features, model, eval, serving, monitoring, governance all appear.
- **Alternatives considered.** At least two realistic alternatives, with a one-paragraph "why not" for each.
- **Risks and mitigations.** The risk register for the design (per `mod-310/exercise-01` §5 categories).
- **Rollout plan.** How the design is delivered incrementally; the rollout mechanism (canary, A/B, progressive) and the measurement plan.
- **Open questions.** Named questions the design does not yet resolve, each with an owner and a target resolution date.

**§2. Problem framing.**

- **Is the thesis specific and falsifiable.** A thesis of "make X better" is not reviewable; a thesis of "move SLI X by Y percentage points by date Z" is.
- **Is the primary outcome tied to an existing SLI or dashboard.** The reviewer confirms the outcome metric exists, is instrumented, and is known-stable at the baseline.
- **Are the non-goals named.** Missing non-goals is a design smell; the review bounces back for them.
- **Is the "do nothing" alternative represented.** Every design is implicitly competing with "do nothing." A design that cannot articulate why "do nothing" loses is a design whose cost has not been argued.

**§3. Correctness of the proposed design (the first-order review).**

- **Data model and interfaces.** Named schemas, named interfaces, named ownership. A design doc with pseudo-code at the interface boundary is a design that has not yet nailed the interface; the review asks for the interface to be pinned.
- **ML specifics.** For ML designs the senior reviewer walks through the Model-Card checklist (`mod-309/exercise-01`) in abbreviated form: what is the model, what is the training data, what is the eval, what are the slices, what are the risks, what is the governance owner, what is the SLO, what is the rollout.
- **Training-serving symmetry.** If the design has training and serving paths, does it name how it enforces feature parity? A shared feature-computation library; a schema that is single-source-of-truth; a drift monitor (tied to `mod-307`) — the reviewer wants the answer explicitly.
- **Data lineage.** A design doc for a new training pipeline cites the planned lineage artefact (`mod-309/exercise-02`). A design that does not pin lineage ships without observability.
- **Dependency graph.** Named upstream services, named versions, named ownership. A design that depends on an unnamed "platform team's thing" is a design that has not located itself in the ecosystem. Cross-reference `mod-308/exercise-01` (paved road audit) and `mod-308/exercise-02` (build-vs-adopt).
- **Capacity and SLO.** Named QPS, named latency targets, named cost envelope. ML designs are expensive at serving time; the design doc declares the serving budget up front. Cross-reference `mod-307/exercise-01`.

**§4. Alternatives.**

- **Two or more realistic alternatives.** "Do nothing" and the design itself is one alternative, not two. A second real option is expected.
- **Alternatives are not strawmen.** A design that presents "do the same thing but worse" as its only alternative is argumentative, not analytical. The reviewer pushes back.
- **Build vs. adopt has been considered.** Where paved-road options exist, the design cites or is paired with a `mod-308/exercise-02`-style build-vs-adopt doc. A design that re-builds a paved-road capability without that discussion is a design that will be challenged at the readout.

**§5. Risks.**

- **Risk register has owners, mitigations, and elevation triggers.** Rows missing any of those are prose.
- **Security row references OWASP ML Top 10 / MITRE ATLAS** as applicable.
- **Governance row references NIST AI RMF** (GOVERN / MAP / MEASURE / MANAGE) and NIST AI 600-1 for generative surfaces.
- **Model risks are named.** Quality regression, fairness regression (`mod-309/exercise-03`), robustness regression (`mod-309/exercise-04`), training-data risks (`mod-309/exercise-02`).

**§6. Rollout and reversibility.**

- **Rollout mechanism named.** Canary / A/B / progressive. Un-specified rollout → design incomplete.
- **Measurement plan named.** Statistical power considerations referenced (`mod-306/exercise-01`); measurement window dated.
- **Irreversible commitments flagged.** Schema changes, deprecations, Model Card publications, vendor contracts — all irreversible on a quarterly horizon.
- **Rollback plan.** Named steps; if "no rollback is possible," the design must say why and name what compensating control replaces rollback.

**§7. Open questions and dependencies.**

- **Open questions have owners and dated resolutions.** A design-doc "open question" that exists for a quarter is a design that was approved on vibes. Each open question has a named owner and a target date; an unresolved open question past its date is a re-review trigger.
- **Dependencies on sister teams are confirmed, not assumed.** The design-review process confirms the dependency with the sister team. "Assumed" dependencies in design review become "missing" dependencies in code review.

**§8. Approval call.**

Reviewers of the design doc choose:

- `Approved` — the design is ready for implementation; code review proceeds on the PRs that flow from it.
- `Approved with revisions` — the design is approved conditional on named, dated revisions to the doc; implementation can begin on the parts not affected by the revisions.
- `Revise and re-review` — the design has structural problems; the author revises and re-circulates.
- `Reject` — the design is not a viable approach; an alternative must be proposed. Rejection is rare and is accompanied by named reasons.

The reviewer attests to §1–§7 at the approval level chosen. A design-review approval is a dated, signed artefact; it does not expire with the author's memory.

### 2.3 Worked example — code review of a feature-normalization change

> **Illustrative diff** — a change to `features/numeric_scaler.py` in the hypothetical `agent-assist` codebase, introducing a per-feature clamp on the normalization output. This example is a composite for calibration.

**The diff (summary of the hunk).**

```python
# features/numeric_scaler.py (before)
def scale(x: np.ndarray, mean: float, std: float) -> np.ndarray:
    return (x - mean) / std

# features/numeric_scaler.py (after)
def scale(
    x: np.ndarray, mean: float, std: float, clamp_abs: float = 8.0
) -> np.ndarray:
    z = (x - mean) / max(std, 1e-6)
    return np.clip(z, -clamp_abs, clamp_abs)
```

Serving code calls `scale` identically; the training-time feature build is in `training/make_features.py`.

**Senior reviewer walkthrough.**

1. **§1 framing.** PR description reads: "Clamp numeric-scaler output to ±8σ; motivated by incident `IR-2026-114`, where a training example with `std` of effectively zero produced `inf` features and a NaN training loss. Also added `max(std, 1e-6)` as a floor." — specific why, named incident, named root cause. Pass §1.
2. **§2.1 data flow.** `clamp_abs` default is 8.0; the reviewer checks whether any training-time feature was legitimately beyond 8σ in historical data. A quick offline check (or a reviewer ask for one) confirms: `< 0.01%` of training examples exceed 8σ; the clamp is safe. Pass §2.1.
3. **§2.2 training-serving skew.** The scaler is shared between training (`training/make_features.py`) and serving (`serving/featurizer.py`), via the same `features/numeric_scaler.py` import. The reviewer confirms both paths now use the clamp and both read the same `clamp_abs` config. The reviewer asks: "is `clamp_abs` serialized with the trained model, so that a re-trained model with a different clamp cannot be loaded into a serving version that uses the old clamp?" The author adds the constant to the model metadata and the feature-store spec. Pass §2.2 *after* follow-up.
4. **§2.3 numerical correctness.** The floor `max(std, 1e-6)` is placed after the mean subtraction, not before — this is the correct placement (the division-by-zero is in the denominator, not the numerator). The reviewer asks about `nan` / `inf` inputs: `np.clip` on `nan` preserves `nan`. The reviewer asks for a `nan` test: the author adds a unit test asserting that `nan` inputs produce `nan` outputs (silent masking would be worse than the current behaviour). Pass §2.3 *after* follow-up.
5. **§2.4 edge cases.** Empty-batch (`shape (0,)`) handled correctly by `np.clip`. Single-sample batch likewise. Pass §2.4.
6. **§2.5 reproducibility.** No randomness. Pass §2.5.
7. **§2.6 metrics gate.** The clamp will change feature distributions; the reviewer asks whether this triggers a Model Card refresh. Answer: the feature definitions themselves changed, so Data Statement §7 (pre-processing) and Model Card §6 (training-data summary) are updated. Pass §2.6 *after* follow-up.
8. **§2.7 rollback.** The clamp is a pure-function change in a shared library. Rollback is a one-line revert on the library; cached model weights are compatible since `clamp_abs` is read at inference. Pass §2.7.
9. **§3 safety / security / governance.** No secrets. No new dependency. No PII touched. No OWASP ML Top 10 category exposure changed. NIST AI 600-1 risks unchanged. Pass §3.
10. **§4 observability.** Reviewer asks: "do we log how often the clamp is hit in production?" The author adds a counter metric `scaler_clamp_hits_total{feature}` with the existing feature-metric sink; a runbook line is added noting that `> 1%` clamp hits on any feature is a drift signal (tied to `mod-307`). Pass §4 *after* follow-up.
11. **§5 tests.** Unit tests added: zero-std input; nan input; large positive / negative input; empty batch. Offline-eval delta reported: quality metrics within noise band; one feature's distribution tightened as expected. Pass §5.
12. **§6 readability.** Function named appropriately; comment explains *why* clamp was chosen (one line referencing the incident). Pass §6.
13. **§7 approval call.** `Approve with changes` — three specific follow-ups listed (serialize `clamp_abs` with model metadata; add `nan` test; log clamp-hit counter and runbook line). Follow-ups landed in a second PR within two days; reviewer re-approves on the follow-up. Final state: merged.

The walk-through is the shape of a senior review: §1 framing is unblocking, §2 correctness is where the review's time goes, §3–§5 are mechanical but necessary, §6 is light. The three follow-ups are the review's teaching moments — they are things the author will now do by default on the next PR. That is the review's compounding value.

### 2.4 Worked example — design review of a drift-monitoring layer

> **Illustrative design doc** — a proposed drift-monitoring layer for the `agent-assist/draft` surface. This example is a composite for calibration.

The design doc proposes a per-feature PSI monitor computed nightly, paging on-call when any feature crosses a threshold. The design-review walkthrough:

1. **§1 shape.** All seven required parts present. Pass §1.
2. **§2 problem framing.** Thesis: "detect feature drift within 24 hours of onset, with a false-positive rate below one page per month across the full feature set." Primary outcome: 24-hour detection latency on a synthetic drift injection; FP rate measured on a two-week baseline. Non-goals named (not detecting concept drift at the output; not detecting data-quality faults such as schema violations — those are handled upstream). "Do nothing" alternative: current reactive-only drift detection is the baseline; its cost is measured as "incident `IR-2026-114` went un-caught for 11 days." Pass §2.
3. **§3 correctness.**
   - Data model: PSI per-feature on 1-day and 7-day windows with named bin boundaries. Reviewer confirms bin boundaries are not recomputed per-day (which would hide drift); they are fixed from a training-period reference.
   - ML specifics: tied to Model Card §4 metrics table; drift signal feeds §9 recommendations as an escalation, not an auto-rollback.
   - Training-serving symmetry: features are read from the production feature-store slice, same as serving; no re-computation.
   - Data lineage: cites `mod-309/exercise-02` lineage artefact; the drift signal is itself logged as lineage.
   - Dependency graph: uses `feature-platform-v2` (dependency on `mod-308/exercise-01` audit row); design cites the dependency's own SLOs and acknowledges risk if the feature platform's freshness SLI slips.
   - Capacity and SLO: nightly batch compute envelope sized; the feature-store read is the dominant cost; the monitor's own SLO is 24-hour signal-to-page latency.
   - Pass §3.
4. **§4 alternatives.**
   - Alt-1: KL divergence instead of PSI. Why not: on sparse categorical features (which 20% of this surface's features are), PSI is more stable; KL requires careful smoothing.
   - Alt-2: use a hosted drift-monitoring product. Why not: build-vs-adopt doc (`mod-308/exercise-02`-style) was written; adopt loses on provenance-of-signal and data-platform integration.
   - Alt-3: per-feature-group aggregate instead of per-feature. Why not: aggregates hide the feature causing the drift; the on-call page is less actionable.
   - Pass §4.
5. **§5 risks.** Risk register covers: false-positive alert fatigue (operational risk), bin-boundary staleness (model risk), feature-store freshness slip (dependency risk), PSI definition change over time (governance risk — the PSI threshold is a published number and changes to it go through Model Card re-review). Each row has owner, mitigation, elevation trigger. Pass §5.
6. **§6 rollout.** Rollout plan: shadow mode (compute PSI, do not page) for one week to calibrate the FP rate; then page-at-warning for another week; then page-at-critical. Measurement: `mod-306/exercise-01`-style FP rate estimate with a 95% CI across the shadow window. Rollback: the pager rule is a one-liner revert; the compute job is idempotent and can be paused. Irreversible commitments: the PSI thresholds, once published to the Model Card, are a dated commitment. Pass §6.
7. **§7 open questions.** Three named questions with owners and dated resolutions: "what to do on a one-off drift spike that resolves itself" (owner: `team-agent-assist` tech-lead; resolve by week 1); "which features are silver vs. gold tier for pager routing" (owner: `team-support-ops`; resolve by week 1); "how are new features onboarded into the monitor" (owner: `team-agent-assist` tech-lead; resolve by week 2). Pass §7.
8. **§8 approval call.** `Approved with revisions` — approve the design, revisions are the three open questions above resolved before code review of the PRs begins. One reviewer signature (senior ML engineer owning the surface) plus one from `team-data-platform` (owner of the feature-store dependency) plus one from `team-support-ops` (owner of the pager routing).

The walk-through is a design review that improved the design: the shadow-mode calibration period came out of the review; the escalation-vs-auto-rollback decision came out of the review; the pager-tier question came out of the review. None of these would have been caught at code review; all three would have surfaced in production.

### 2.5 Lightweight rubric linter (static checks)

```python
# rubric_lint.py
"""Lint a code-review or design-review rubric for structural completeness."""

import re
import sys


CODE_REVIEW_SECTIONS = [
    "framing", "correctness", "safety", "observability", "tests",
    "readability", "approval",
]
DESIGN_REVIEW_SECTIONS = [
    "shape", "problem framing", "correctness", "alternatives",
    "risks", "rollout", "open questions", "approval",
]


def lint_rubric(text: str, kind: str) -> list[str]:
    errors: list[str] = []
    lowered = text.lower()
    sections = CODE_REVIEW_SECTIONS if kind == "code" else DESIGN_REVIEW_SECTIONS
    for s in sections:
        if s.lower() not in lowered:
            errors.append(f"missing section: {s}")
    if not re.search(r"owasp ml top", lowered):
        errors.append("OWASP ML Top 10 not referenced")
    if not re.search(r"mitre atlas|atlas", lowered):
        errors.append("MITRE ATLAS not referenced")
    if not re.search(r"nist ai rmf|govern|measure|manage|map", lowered):
        errors.append("NIST AI RMF not referenced")
    if "approve with changes" not in lowered and "approved with" not in lowered:
        errors.append("no 'approve with changes' / 'approved with revisions' "
                      "outcome documented")
    return errors


if __name__ == "__main__":
    path, kind = sys.argv[1], sys.argv[2]  # kind in {code, design}
    errs = lint_rubric(open(path).read(), kind)
    for e in errs:
        print(f"rubric-lint error: {e}")
    sys.exit(1 if errs else 0)
```

## 3. Validation steps

Walk the rubrics through this checklist before publishing them to the team:

1. **Each rubric has sections in the required order.** Framing / shape → correctness → safety → observability → tests → readability → approval (code); shape → framing → correctness → alternatives → risks → rollout → open-questions → approval (design).
2. **Each section names what a reviewer looks for and what a "pass" looks like.** A rubric that says "check correctness" without enumerating the sub-checks is a vibe, not a rubric.
3. **The code-review rubric names an ML-specific correctness lens** (data flow, training/serving skew, numerical correctness, reproducibility). A rubric that reduces ML correctness to "the tests pass" is incomplete.
4. **The design-review rubric names a required shape** before evaluating content. Shape-checks before content-checks save review time.
5. **Both rubrics reference OWASP ML Top 10 and (where generative surfaces are in scope) MITRE ATLAS.**
6. **Both rubrics cite NIST AI RMF** (GOVERN / MAP / MEASURE / MANAGE) in the sections they ground.
7. **Approval calls are named and bounded.** Three code-review outcomes (`LGTM`, `approve with changes`, `changes requested`). Four design-review outcomes (`approved`, `approved with revisions`, `revise and re-review`, `reject`). Fewer than that collapses nuance; more fragments the review culture.
8. **Each rubric includes a worked example** showing the review in action. A rubric without a worked example is a document the team will read once and forget.
9. **The rubrics cross-reference the other `mod-310` solutions** (roadmap, mentorship, interview) and the governance / evaluation / SLO modules.
10. **"Review as teaching" is named.** A senior reviewer's goal is to teach the next reviewer; the rubric must say so, or new reviewers will not know it is part of the job.

Static checks a CI step can enforce:

```bash
grep -ciE "framing|correctness|tests|approval"          code-review-rubric.md
grep -ciE "shape|alternatives|rollout|open question"    design-review-rubric.md
grep -ciE "owasp ml top|atlas|nist ai rmf"              *.md
grep -ciE "approve with changes|approved with revis"    *.md
```

`rubric_lint.py` from §2.5 gives a programmatic version.

## 4. Rubric / review checklist

- [ ] Code-review rubric has §1–§7 in order: framing → correctness → safety/security/governance → observability → tests → readability → approval.
- [ ] Framing (§1) asks for the "why," single-concern scope, upstream design doc if required, and an explicit test plan.
- [ ] Correctness (§2) names ML-specific checks: data flow, training/serving skew, numerical correctness, edge cases, reproducibility, metrics-change gate, rollback.
- [ ] Safety/security/governance (§3) references OWASP ML Top 10 and MITRE ATLAS for generative surfaces, NIST AI 600-1 risks, PII register, and launch-gate alignment.
- [ ] Observability (§4) covers logging, metrics, rate-limit / cost, and runbook updates.
- [ ] Tests (§5) covers unit, integration / eval delta, flaky-test prohibition, disabled-tests prohibition.
- [ ] Readability (§6) is bundled under "never blocks landing alone"; style follow-ups are allowed as separate PRs.
- [ ] Approval call (§7) names three outcomes: LGTM, approve-with-changes, changes-requested; each with the kind of comment that goes with it.
- [ ] Design-review rubric has §1–§8 in order: shape → problem framing → correctness → alternatives → risks → rollout → open questions → approval.
- [ ] Shape (§1) enumerates the seven required parts of a design doc.
- [ ] Problem framing (§2) demands a measurable thesis, SLI anchor, non-goals, and "do nothing" alternative.
- [ ] Correctness (§3) walks the Model-Card checklist in design-review form, names training-serving symmetry, lineage, dependency graph, SLO envelope.
- [ ] Alternatives (§4) requires at least two realistic alternatives and cites the build-vs-adopt discussion where relevant.
- [ ] Risks (§5) requires a register with owners / mitigations / elevation triggers; references OWASP / ATLAS / NIST AI RMF.
- [ ] Rollout (§6) requires a named rollout mechanism, measurement plan, irreversible commitments flagged, and rollback plan.
- [ ] Open questions (§7) each have an owner and a dated resolution.
- [ ] Approval call (§8) names four outcomes.
- [ ] Worked example present for both rubrics.
- [ ] Cross-references: `mod-310/exercise-01`, `mod-310/exercise-03`, `mod-305`, `mod-306`, `mod-307`, `mod-308`, `mod-309` as applicable.

Review-weighted points for grading:

- Code-review rubric correctness lens (§2 ML-specifics): 20%
- Code-review rubric safety / security / governance coverage (§3): 15%
- Code-review rubric approval-call discipline (§7): 10%
- Design-review rubric shape (§1) and framing (§2): 15%
- Design-review rubric correctness (§3) and alternatives (§4): 15%
- Design-review rubric rollout and open-questions discipline (§6 / §7): 10%
- Worked examples (code and design): 10%
- Cross-references to standards and sibling solutions: 5%

## 5. Common mistakes

- **Rubric as paragraph.** A rubric written as a wall of text is a rubric nobody skims during review. Lists, orderings, pass-criteria per bullet.
- **Correctness checklist without ML specifics.** "Check that the tests pass" is not an ML correctness check. Training/serving skew, numerical correctness, data-flow shapes, and reproducibility are the ML-specific lenses.
- **Style first, correctness second.** A rubric that leads with naming and formatting trains reviewers to spend attention on the cheap questions. Correctness first.
- **No "approve with changes" option.** When the only options are LGTM and reject, reviewers become reluctant to request changes (afraid of being seen as blocking) and merge marginal PRs instead. The three-outcome structure defuses this.
- **Design-review rubric without shape.** A rubric that evaluates content on a doc that has no "alternatives considered" section wastes the reviewer's time before anyone realizes the doc is shaped wrong.
- **"Alternatives" as strawman.** A design that lists a self-evidently-worse alternative as its only counterpoint is a design that has not been pressure-tested. The rubric must ask for a realistic alternative.
- **Risks as prose.** Risk sections written as paragraphs are not triageable. The register is a table: description | likelihood | impact | owner | mitigation | elevation trigger.
- **"Rollback possible" asserted but not described.** A rollback plan is a sequence of steps, not a claim. "We can revert the PR" is not a rollback plan if the PR migrated a schema.
- **Open questions deferred without owners.** "Open question: think about this more" is a design that approved on vibes.
- **No worked example.** Rubrics without an example are rubrics the team will apply inconsistently.
- **Rubric that doesn't reference OWASP / ATLAS / NIST.** For ML changes, the review surface overlaps with three published taxonomies; a rubric that ignores them re-invents a worse version of each.
- **Rubric that doesn't teach.** A senior reviewer's job is to raise the bar for the next reviewer. If the rubric does not name "this review is also teaching," it under-weights the compounding value of review.
- **One rubric, two levels.** Trying to grade a junior-level change and a senior-level change with the same bar either lets juniors through on changes they do not understand or lets seniors through on changes a junior would be rejected for. A rubric names the level it is applied at.
- **Rubric with no "when to escalate."** A reviewer who finds a cross-team security issue must know when to pull in `team-security`; a reviewer who finds a cross-team governance issue must know when to pull in `team-governance`. The rubric names these escalation paths.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds the rubric as the governance instrument that makes review a legible step of the ML lifecycle; MAP grounds the design-review risk framing (§5); MEASURE grounds the exit / approval criteria (§7–§8); MANAGE grounds the escalation and rework loops (§3 of the code rubric, §6 of the design rubric).
- NIST AI 600-1 (Generative AI Profile) — <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>. The GAI risk taxonomy is referenced in §3 (code-review safety/security/governance) and §5 (design-review risk) for generative surfaces; changes touching system prompts, tool-use wiring, or RAG grounding are mapped to the taxonomy.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Used as the threat taxonomy for §3 (code review) and §5 (design review) rows on ML-specific exposure (input manipulation, data poisoning, model inversion, membership inference, model theft, supply-chain, transfer-learning, model skewing, output integrity, denial of ML service).
- MITRE ATLAS — <https://atlas.mitre.org/>. Used as the adversarial-tactic taxonomy against which §3 (code review) and §5 (design review) rows are framed for generative surfaces (prompt injection, exfiltration via inference, jailbreak evasion).

Practitioner implementation examples:

- VeriSwarm engineering blog — <https://veriswarm.ai>. Shape-of-artefact reference for team-level review-rubric write-ups; not a standard.

Local context (authoritative for the artefact shape and cross-references):

- Paired lesson: `lessons/mod-310-technical-leadership/exercises/exercise-02-code-review-and-design-review-at-bar.md`.
- Module README: `modules/mod-310-technical-leadership/README.md`.
- Sibling exercise (roadmap, whose milestones go through these reviews): `modules/mod-310-technical-leadership/exercise-01-roadmap-and-scoping/SOLUTION.md`.
- Sibling exercise (mentorship plan — review as teaching): `modules/mod-310-technical-leadership/exercise-03-mentorship-plan-for-a-mid-level/SOLUTION.md`.
- Sibling exercise (interview loop — the hiring bar aligned with the review bar): `modules/mod-310-technical-leadership/exercise-04-rubric-based-ml-interview-loop/SOLUTION.md`.
- Related exercise (eval harness whose delta is the primary §5 test artefact): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (experiment design the design-review §6 measurement plan cites): `modules/mod-306-experimentation-at-scale/exercise-01-power-mde-and-ramp-plan/SOLUTION.md`.
- Related exercise (SLO targets the code-review §4 and design-review §3 capacity sections reference): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (paved-road audit that design-review §3 dependency graphs cite): `modules/mod-308-platform-collaboration/exercise-01-paved-road-consumption-audit/SOLUTION.md`.
- Related exercise (build-vs-adopt doc design-review §4 alternatives references): `modules/mod-308-platform-collaboration/exercise-02-build-vs-adopt-tradeoff-doc/SOLUTION.md`.
- Related exercise (contribute-back RFC shape the design-review §4 cites where applicable): `modules/mod-308-platform-collaboration/exercise-03-contribute-back-rfc-to-platform/SOLUTION.md`.
- Related exercises (governance artefacts the review-rubric §3 and §5 reference): `modules/mod-309-responsible-ai-governance/` (all four exercises, in particular Model Card and threat model).
