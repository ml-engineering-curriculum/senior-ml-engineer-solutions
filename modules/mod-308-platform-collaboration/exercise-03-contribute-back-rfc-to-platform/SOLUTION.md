# exercise-03 — Contribute-Back RFC to a Peer Platform Team (Reference Solution)

Paired exercise: `lessons/mod-308-platform-collaboration/exercises/exercise-03-contribute-back-rfc-to-platform.md`.

## 1. Solution overview

This exercise produces `rfc-<capability>-<yyyyqq>.md` — a Request For Comments addressed to a peer platform team, proposing that the platform team adopt a capability extension, API change, or new primitive that one or more consumer teams need. The senior ML engineer authors the RFC *on behalf of the consumer*, not on behalf of the platform team: the RFC's job is to make the consumer's need legible and actionable, to propose a shape the platform team can build (or counter-propose), and to pre-answer the questions the platform team will have about compatibility, migration, and ownership.

The artefact has eight parts:

1. A **header** that names the RFC, its authors (roles, not individuals), the paved-road team it is addressed to, the date, and the RFC's current status (`draft` / `review` / `accepted` / `counter-proposed` / `rejected`).
2. A **context** section that pins down the consumer need, including which audit row (from `exercise-01`) or tradeoff-doc outcome (from `exercise-02`) prompted the RFC. The RFC is downstream of the audit and the tradeoff doc.
3. A **problem statement** — a specific consumer-side problem the current paved road cannot solve, with one or two illustrative consumer scenarios that would benefit. Problems are *not* feature requests; they are capability gaps grounded in a user-visible or operator-visible pain.
4. A **goals and non-goals** section — what the RFC explicitly intends to solve and, importantly, what it explicitly does not. Non-goals are the artefact's way of reassuring the platform team that the proposal is not the start of a scope spiral.
5. A **proposed change** section — the API surface, data model changes, configuration shape, and (optionally) a sketch of implementation. The proposal is specific enough that the platform team can estimate it and, if they agree, implement it; it is not a hand-wave.
6. An **alternatives considered** section — other shapes the proposal could take, including: the status quo (why it is insufficient), a smaller change (why it does not suffice), a larger change (why it is out of scope), and, when relevant, an off-platform alternative the consumer team would otherwise pursue. The alternatives section is where the senior engineer demonstrates they thought about the problem from the platform team's perspective, not only the consumer's.
7. A **compatibility and migration** section — backwards compatibility story, deprecation plan for any superseded API, migration path for existing consumers, and the proposed rollout cadence. Platform teams reject RFCs that would break every existing consumer; platform teams merge RFCs that come with a credible migration.
8. An **ownership, rollout, and review** section — who owns the capability after merge, who maintains the first release, what the rollout stages look like, and what the review cadence is after it ships. RFCs without a named post-merge owner tend to become un-maintained platform debt.

The reference solution below shows the structure, works a labeled illustrative example through it (the "prompt-version as a first-class model-registry artefact" RFC that fell out of `exercise-01`'s row 2 and `exercise-02`'s `contribute-back` decision), and gives the grading rubric that calibrates every learner's RFC to the same bar.

The exercise exists because contributing back is where cross-team collaboration becomes load-bearing: an ML surface's deviations from the paved road are either shadows the audit flags and the tradeoff doc records, or they are extensions the surface team proposes and the platform team adopts. The senior ML engineer's job is to turn consumer-side pain into platform-side work in a way that respects the platform team's roadmap, their ownership boundary, and their accountability to other consumers. An RFC that reads like "please build my feature" is not a contribution; an RFC that reads like "here is a capability gap three consumer teams share, here is a shape that solves it, here is why we rejected the alternatives, and here is how you merge it without breaking anyone" is the contribution. NIST AI RMF GOVERN grounds the RFC as the governance instrument that keeps cross-team capability authorship legible; MANAGE grounds the rollout and migration plan as risk control on the change.

## 2. Worked answer / implementation

### 2.1 Header

Six fields, in this order:

- **RFC title.** Short, specific, and in the shape "`<capability> for <platform>`" or "`<change> to <platform>`." Titles that read as feature requests ("add X to Y") are weaker than titles that read as capability statements ("first-class prompt versions in `model-registry`").
- **Authors.** Roles (team + seniority), not individuals. The primary author is typically a senior ML engineer on a consumer team; co-authors from other consumer teams who share the need dramatically increase the RFC's weight.
- **Addressed to.** The paved-road team (role). The RFC's audience is the people who will merge or reject it.
- **Date.** Submission date.
- **Status.** One of `draft`, `review`, `accepted`, `counter-proposed`, `rejected`, with the date of the last status change.
- **Target platform release.** The platform's release train the RFC is proposed against (e.g., `model-registry v3` or "the Q1 release train"). Named explicitly so the platform team can place the proposal on their roadmap.

### 2.2 Context

The context section pins the RFC to the artefacts that produced it. Required lines:

- **Prior-audit linkage.** The specific row from the author's `exercise-01` audit where the deviation that this RFC addresses appears. The audit's deviation-reason cell is often the opening sentence of §2.3.
- **Prior tradeoff-doc linkage, if any.** If `exercise-02` was run and chose `contribute-back`, the RFC is downstream of that decision; the tradeoff doc's recommendation paragraph is a direct input to the RFC's context.
- **Co-sponsoring consumer teams.** Other consumer teams who share the need, with a name and a role per team. If the RFC has only one sponsor, its priority is a conversation between two teams; with three or more sponsors, it is a platform-roadmap conversation. Co-sponsorship is the single strongest signal a platform team weights.
- **Related RFCs or deprecations.** Prior RFCs on adjacent capabilities, or deprecations the proposed change depends on. An RFC that silently depends on a deprecation the platform team has not committed to is an RFC that will be rejected or stalled.
- **Lifecycle of the consumer surface.** Whether the consumer is pre-production, production, or regulated (`mod-307`, `mod-309` referenced). Platform teams calibrate willingness to make breaking changes to the lifecycle of the consumers.

### 2.3 Problem statement

The problem statement is a paragraph (not a bullet list) that names the capability gap the consumer team hits today. Discipline:

- **Grounded in a consumer-visible scenario.** "Model-registry does not support prompt versions as first-class artefacts; today a prompt rollback requires reconciling the registry entry against a Google Doc, which has caused two near-misses in the past quarter where the doc and the registry disagreed." That is a problem statement. "We'd like prompt versions in registry" is a request.
- **Frequency and scope.** How often the problem is hit, by how many teams, in how many scenarios per quarter. A problem that one team hits twice a year is a lower-priority RFC than a problem four teams hit weekly. Frequency is where co-sponsorship becomes legible.
- **Current workaround.** What the consumer teams do today (the `partial`, `bypassed`, or `forked` posture from the audit). The workaround is often itself the problem: a shadow spreadsheet, a hand-rolled script, a forked harness. The cost of the workaround is the cost of *not* adopting the RFC.
- **Why the paved road's current shape does not solve it.** Specific. "The current `model-registry.create()` API allows a single `prompt` string on a model entry, but a production prompt has (prompt text, output schema, safety classifier version, system-message, assistant-prefix) — all five change independently." Vague "the API is too limited" statements are a tell that the author has not done the design work.
- **Illustrative scenarios.** One or two short, realistic consumer scenarios (not full code; a paragraph each) that walk through how the consumer team would use the proposed capability. Scenarios are where the platform team pattern-matches to other consumer teams who would benefit.

### 2.4 Goals and non-goals

Platform teams make merge decisions partly on what the RFC *does not* promise.

**Goals.** The specific capability outcomes the RFC commits to. 3–6 goals is the right range; fewer is probably under-scoped, more is probably a scope spiral. Each goal is testable: "A consumer team can store a prompt version with its (prompt text, output schema, safety classifier version, system-message, assistant-prefix) tuple" is testable. "Prompt versioning should feel natural" is not.

**Non-goals.** Equally specific; 3–6 items. Non-goals are not disclaimers; they are deliberate exclusions that keep the proposal from eating the platform team's quarter. Example non-goals:

- "Not in scope: a full prompt-engineering IDE. The registry stores and lineages prompt versions; the authoring tooling is out of scope and owned by a different consumer-side effort."
- "Not in scope: an LLM-judge evaluation pipeline on registered prompts. Prompt-level evaluation is the subject of a separate RFC on `ml-eval-harness`."
- "Not in scope: automated prompt-regression detection. This RFC does not commit the platform team to building regression detection on top of registered prompts."

The discipline is that the platform team can skim the non-goals and trust that the RFC is the proposal they are being asked to merge — not the beachhead of a quarter-long scope grab.

### 2.5 Proposed change

The proposal is specific. For a paved-road RFC, "specific" usually means one of (or several of):

- **API surface.** The new or modified method signatures / endpoints / CLI subcommands, written in the paved-road's existing style. If the platform uses Python client libraries, the proposal is a Python API sketch; if the platform uses REST, it is an endpoint sketch with a request / response body. Not a full implementation — a shape the platform team can confirm or refine.
- **Data model.** New entities, fields, or relationships. For a model-registry RFC, this is the registry schema change; for a feature-store RFC, this is the `feature_spec.yaml` schema change. Named with types and semantics ("`prompt_version: str` — opaque, platform-generated; semver is not implied") so a reader can model the artefact.
- **Configuration or declarative surface.** If the capability has a declarative config, the config excerpt. If the capability has runtime flags, the flag names.
- **Lifecycle hooks.** The capability's write path, read path, and any audit / logging / governance hook the platform team would plug into it.
- **Illustrative code snippet (optional).** A short code snippet showing one consumer using the proposed API. The snippet is a reference point for the API shape, not an implementation request.

For each element, the proposal states: the new shape, how it slots into the existing shape, and (for breaking changes) what it replaces. The proposed change is detailed enough that a reader can estimate the platform team's work; it is not so detailed that the author has pre-written the implementation and the platform team is reduced to a merge request.

### 2.6 Alternatives considered

Four alternatives, at minimum:

- **Status quo (do nothing).** Why it is insufficient — the problem statement in short form. The platform team reads this section expecting to see the problem taken seriously; an "status quo is insufficient because the problem is real" line is thin and tells the platform team the author did not interrogate their own ask.
- **Smaller change.** A narrower version of the proposal — e.g., "store the prompt as an opaque blob rather than a structured tuple." Why it does not suffice. This is where the author demonstrates they understand the minimum viable capability vs. the proposed one.
- **Larger change.** A broader version — e.g., "a full prompt-management service inside `model-registry`." Why it is out of scope. This is where the author demonstrates they are not over-asking.
- **Off-platform alternative.** What the consumer team would do if the RFC is rejected or deferred. Often this is "continue running the shadow" (from the audit) or "build a team-local service" (from the tradeoff doc). Naming the off-platform alternative honestly is not a threat; it is the platform team's input into prioritization — if five teams would otherwise fork, that is a stronger prioritization signal than three teams quietly waiting.

Alternatives considered sections that read as "we considered doing nothing and that was bad" are a tell that the author drafted in one pass. The best alternatives sections list 4–6 alternatives, each with a specific reason for rejection.

### 2.7 Compatibility and migration

Platform teams merge RFCs with a credible migration plan; they reject or stall RFCs without one.

**Backwards compatibility.** Does the proposal break any existing consumer? For each existing consumer class, how is it handled:

- Zero-touch compatible (existing consumers see no change).
- Opt-in (existing consumers see no change unless they adopt the new API).
- Forced migration (existing consumers must change; migration tooling is part of the proposal).

Opt-in is the most common and least contentious. Forced migrations are the costliest and should be reserved for cases where the existing API is structurally broken; the RFC states the reason explicitly.

**Deprecation plan for superseded APIs.** If the RFC replaces an existing API, the deprecation plan names:

- The deprecation period (two release cycles is a common default).
- The deprecation signal (warning log, deprecation header, test-only failure, then removal).
- Named consumers whose migration is co-owned by their teams.
- The RFC's commitment to co-author migration PRs for high-value consumers.

**Migration tooling.** If the migration is non-trivial, the RFC names the tooling the platform team (or the author) will build: a migration script, a one-shot backfill job, a `linter` warning for legacy usage. Platform teams prefer RFCs that come with migration tooling over ones that leave migration as a problem for every consumer team.

**Rollout cadence.** The release train the RFC targets, the number of release cycles the capability spends in `beta` before `GA`, and the gating criteria (what signal says "ready to GA"). A capability going straight to GA is often a tell that the author did not budget for discovery after real consumers hit it.

### 2.8 Ownership, rollout, and review

Three fields that keep the capability from drifting into un-maintained platform debt.

**Ownership after merge.** The role that owns the capability after it ships. Typical shape: the paved-road team owns the capability; the proposing consumer team is on the hook for integration testing and for co-authoring the first two downstream consumer migrations. An RFC whose answer to "who owns this after merge?" is silence is an RFC that produces unmaintained platform code.

**Rollout stages.** Named stages (e.g., `internal-alpha` → `beta-with-2-consumers` → `beta-open` → `GA`) with named entry criteria for each. Entry criteria are objective (e.g., "two consumer integrations running for two weeks with no incidents," "an SLO on the new endpoint held for a release cycle"), not subjective ("feels ready").

**Review cadence.** Post-GA, when is the capability re-reviewed? Common defaults: 90-day post-GA retrospective, quarterly review of adoption and consumer feedback, annual API surface review. The review cadence catches "the RFC shipped but nobody adopted it" quickly enough to iterate.

### 2.9 Illustrative worked example (not a real system)

> **Illustrative RFC** — "first-class prompt versions in `model-registry` v3," co-authored by `team-agent-assist` (lead), `team-support-ops`, and `team-eval`. Addressed to `team-mlops`. Target release train: `model-registry v3` (2027-Q1). This example is a composite for calibration; do not treat it as a real system.

#### Header

- **Title:** First-class prompt versions in `model-registry`
- **Authors:** `team-agent-assist` L30 (primary); `team-support-ops` tech-lead (co-author); `team-eval` tech-lead (co-author)
- **Addressed to:** `team-mlops`
- **Date:** 2026-12-15
- **Status:** `draft` (ready for review; status change log in appendix)
- **Target platform release:** `model-registry v3` (Q1 release train; feature branch to be cut 2027-01-15)

#### Context

- **Prior-audit linkage.** `team-agent-assist` paved-road audit (2026-11-15), row 2: `model registry` posture is `partial` because prompt versions are not first-class in the registry; current consumption carries a parallel Google Doc of promoted prompts, which has caused two reconciliation near-misses in the past quarter.
- **Prior tradeoff-doc linkage.** `team-agent-assist` build-vs-adopt doc (2026-12-01) for the "promote prompt version" capability selected `contribute-back` over `fork` and `build`; the chosen option named this RFC as the next-step artefact.
- **Co-sponsoring consumer teams.** `team-support-ops` (same surface; downstream impact); `team-eval` (eval-harness needs prompt version as a first-class lineage field so slice-regression attribution is unambiguous). Letters of support attached as appendix.
- **Related RFCs.** `team-mlops`'s "prompt-version storage proposal" (draft, 2026-Q3) — this RFC is a concrete proposal for the shape sketched in that draft.
- **Lifecycle of consumer surfaces.** `agent-assist/draft` is production, in-regulated-review (per `mod-309`). `team-eval`'s harness is production.

#### Problem statement

`model-registry` v2 stores a single `prompt: str` field on a model entry. Production prompts for `assist-llm-v4` are five-field tuples: `(prompt_text, output_schema_version, safety_classifier_version, system_message, assistant_prefix)`. Each of the five fields changes independently and each is reviewed under a different governance surface (prompt_text is reviewed by product; output_schema_version is reviewed by platform; safety_classifier_version is reviewed by security). Today, three consumer teams (ours, `team-support-ops`, `team-eval`) store the five-field tuple in a parallel Google Doc keyed by registry entry URL, and run a nightly job that checks the doc against the registry. The job has flagged two disagreements in the past quarter, both caused by an engineer updating the doc but not the registry (or vice versa). Each disagreement required a 2–3 hour reconciliation and a stand-down of the rollback-to-last-good-prompt playbook until resolved.

A production prompt version is an *artefact* in the registry's sense: it has an identity, a lineage (which model + which data + which prompt-review packet produced it), and a review chain. The doc is a workaround for the fact that the registry does not yet model this artefact. Three consumer teams share the workaround; a fourth (`team-safety`) has asked us whether they should adopt the same doc for their safety-classifier version tracking — if they do, the doc becomes an un-owned cross-team artefact, which is the moment this problem becomes a platform problem rather than a team problem.

**Illustrative scenario 1 (prompt rollback).** On 2026-10-02, a prompt_text change was promoted to `assist-llm-v4-prompt-v38`. At 2026-10-04 an incident surfaces a regression on a support-ticket slice. On-call pages, consults the audit doc, and attempts to roll back to `v37`. The registry has `v38` as the current promoted prompt but the audit doc has `v37.5` (an interim fix) also promoted. Reconciliation takes 2h 15m; rollback is delayed by the reconciliation time.

**Illustrative scenario 2 (safety-classifier correlation).** When a regression is detected, the eval team's harness attributes slice-level regression to a model version. Attribution is wrong when the model version was held constant but the safety_classifier_version changed — which the registry does not record. The harness attributes the regression to the model when it should attribute to the safety-classifier swap. Fixing attribution requires registering the safety_classifier_version alongside the model.

#### Goals

1. A consumer team can store a prompt version in `model-registry` as a first-class artefact with a stable URI and a semver-free, platform-assigned version ID.
2. A prompt version stores a tuple of `(prompt_text, output_schema_version, safety_classifier_version, system_message, assistant_prefix)`; schema evolution is supported via optional fields added at a minor release.
3. A prompt version has a lineage: it references the review packet(s) that approved each of the five fields; the lineage is read-only after promotion.
4. A prompt version has a status machine (`draft`, `in-review`, `promoted`, `deprecated`, `retired`) matching the model registry's model-status machine.
5. A model entry can reference zero or more prompt versions; a prompt version can be referenced by zero or more model entries (many-to-many).
6. The eval harness can query prompt versions by model entry and by lineage; the governance registry can query prompt versions by review packet.

#### Non-goals

1. **Not a prompt-engineering IDE.** Prompt authoring, versioning workflow, and side-by-side comparison are out of scope; those are consumer-team tooling.
2. **Not an evaluation pipeline on registered prompts.** Prompt-level eval is in scope for `ml-eval-harness` (separate RFC).
3. **Not automated prompt-regression detection.** Regression detection on promoted prompts is a separate capability.
4. **Not retro-active backfill of past prompts.** The RFC does not commit the platform team to backfilling the registry with historical prompts from team Google Docs; a consumer-side backfill tool is in scope for the migration section.
5. **Not a replacement for governance review of prompts.** The registry stores and lineages; governance review (per `mod-309/exercise-01`) remains the review surface.
6. **Not a policy engine.** The RFC does not propose policy evaluation on registered prompts; policy is downstream.

#### Proposed change

**API surface (Python client, matching existing style).**

```python
# new: PromptVersion artefact
class PromptVersion:
    uri: str                                 # platform-assigned, opaque
    version_id: str                          # platform-assigned; e.g. 'pv-00f2a3'
    prompt_text: str
    output_schema_version: str               # ref to schema registry
    safety_classifier_version: str           # ref to classifier registry
    system_message: str
    assistant_prefix: str
    lineage: PromptLineage
    status: Status                           # draft | in-review | promoted | deprecated | retired

# new: create a prompt version
client.prompt_versions.create(
    prompt_text=...,
    output_schema_version=...,
    safety_classifier_version=...,
    system_message=...,
    assistant_prefix=...,
    review_packet_uri=...,                   # for lineage
) -> PromptVersion

# new: transition a prompt version's status
client.prompt_versions.transition(
    version_id='pv-00f2a3',
    to_status='promoted',
    actor='team-support-ops/release-manager',
) -> PromptVersion

# changed: models can reference prompt versions
client.models.attach_prompt_version(
    model_entry_id='m-123',
    prompt_version_id='pv-00f2a3',
)
client.models.list_prompt_versions(model_entry_id='m-123') -> list[PromptVersion]

# changed: prompt versions reference review packets
client.review_packets.get_prompt_versions(packet_id='rp-456') -> list[PromptVersion]
```

**Data-model additions.**

- New entity `PromptVersion` (fields above); storage layer: existing registry backend, with an additional table.
- New join table `ModelPromptVersion` (many-to-many between `Model` and `PromptVersion`).
- `PromptVersion.lineage` references `ReviewPacket.id` on the governance registry side (already an entity).

**Lifecycle hooks.** The existing registry's audit log is extended to include `PromptVersion` events (`create`, `transition`, `attach`, `detach`). Existing governance-review hooks fire on `transition to promoted`.

**Configuration.** No new configuration; the capability is on by default in `model-registry v3`.

#### Alternatives considered

- **Status quo (consumer-side Google Doc).** Rejected because the workaround has caused two reconciliation near-misses in the past quarter across one consumer team, and a fourth team is about to adopt the pattern; the cost of the workaround grows superlinearly with consumer count.
- **Store prompt as a JSON blob in an existing `metadata` field.** Rejected because: (a) the blob is unsearchable by lineage ("which prompt was used by `review-packet rp-456`?") without a schema; (b) the registry's existing `metadata` field has no status machine, so prompt promotion / deprecation lose semantic weight; (c) the blob approach replicates the doc-vs-registry failure mode inside the registry itself.
- **Smaller change: single `prompt_text` is replaced by a 5-field struct, no status machine.** Rejected because without a status machine, "which prompt is currently promoted?" has no answer; rollback remains a tribal-knowledge operation.
- **Larger change: a full prompt-management service inside `model-registry` (authoring UI, side-by-side comparison, A/B infrastructure).** Rejected because it would commit the platform team to a quarter of work that is not in the problem statement; the authoring UI is team-side tooling.
- **Off-platform alternative: consumer teams continue the doc, or fork the registry to add prompt versions.** The audit (`exercise-01`, row 2) marked this as a maintenance debt we would retire; `exercise-02` recommended `contribute-back` over `fork` precisely because the fork cost grows with consumer count.
- **Delegate to a prompt-management OSS tool (e.g., an open-source prompt registry).** Rejected on security posture (OWASP ML Top 10 ML06: supply chain risk from an un-vetted OSS registry handling production artefacts) and governance posture (OSS tool's audit log would need to be federated with the governance registry; non-trivial).

#### Compatibility and migration

**Backwards compatibility.** Zero-touch compatible for existing consumers: the existing `model.prompt: str` field is preserved for v3.0 and read-only sourced from the first attached `PromptVersion`'s `prompt_text` for backwards compatibility. Existing consumers see no change until they adopt the new API. Opt-in migration to the new API is encouraged but not required in v3.

**Deprecation plan.** The existing `model.prompt: str` field is marked deprecated in v3.0 (warning log on write) and removed in v4.0 (one major cycle later, approximately 2027-Q3 per the platform team's release cadence). A migration script (in scope — see below) backfills existing `model.prompt` strings into a `PromptVersion` with only `prompt_text` populated; consumer teams retrospectively enrich the other four fields at their convenience.

**Migration tooling.**

- `mlops-cli prompts migrate --source-model-registry` — one-shot: for every model entry with a non-empty `prompt`, create a `PromptVersion` with `prompt_text` populated and `status=promoted`, attach it to the model entry. Dry-run supported.
- `mlops-cli prompts enrich --from-audit-doc <path>` — consumer-side tool for teams with existing audit docs to enrich the auto-created `PromptVersion`s with the four missing fields. Owned by the consumer team that provides the doc; the platform team ships the schema for the doc format.
- Deprecation linter warning on every `model.prompt` write after v3.1.

**Rollout cadence.**

- `internal-alpha` (v3.0, 2027-Q1): capability shipped, used only by the three co-sponsoring teams. Entry criterion: schema review passed; data layer migration tested in staging.
- `beta` (v3.1, 2027-Q2): capability open to all consumers. Entry criterion: three co-sponsoring teams running in production for 4+ weeks without a capability-attributed incident; the migration tooling running clean on a sample of 20 existing models.
- `GA` (v3.2, 2027-Q3): the deprecated `model.prompt: str` field is removed. Entry criterion: >90% of production model entries have at least one attached `PromptVersion`; the deprecation linter has been in place for a release cycle.

#### Ownership, rollout, and review

- **Ownership after merge.** `team-mlops` owns the capability (code, operations, SLA on the new endpoints). The three co-sponsoring teams are on the hook for integration-testing coverage of the new API against their production workflows. `team-agent-assist` L30 commits to co-authoring the first two downstream consumer migrations (pick two consumer teams outside the three sponsors, co-author their migration PRs).
- **Rollout stages.** See compatibility section above.
- **Review cadence.** 90-day post-GA retrospective (jointly owned by `team-mlops` and `team-agent-assist`); quarterly adoption review for the first year post-GA; annual API surface review afterwards.

#### Signatures

The RFC is circulated for review on 2026-12-16 with a comment window of two weeks (deadline 2026-12-30). Required reviewers:

- `team-mlops` (addressed-to; decision authority).
- `team-governance` (reviewer, because prompt versions feed the governance registry).
- `team-security` (reviewer, because safety_classifier_version is covered).
- Co-sponsoring consumer teams (each team's tech-lead).

Comments collected in the RFC's comment log; decision recorded as a status transition in the header.

### 2.10 Optional template: `rfc-template.md`

A recommended on-disk template that teams start from. Platform teams should maintain their own RFC template in their repo; this is a suggested canonical shape for cross-cutting ML platform RFCs.

```markdown
# <RFC title>

| Field | Value |
|---|---|
| Authors | <role (team + seniority)>, <co-author role>, ... |
| Addressed to | <platform team role> |
| Date | <yyyy-mm-dd> |
| Status | draft \| review \| accepted \| counter-proposed \| rejected |
| Target platform release | <release train / version> |

## 1. Context
- Prior-audit linkage: <audit filename + row>
- Prior tradeoff-doc linkage: <filename + recommendation>
- Co-sponsoring consumer teams: <team, role>, ...
- Related RFCs or deprecations: <list>
- Lifecycle of consumer surfaces: <pre-prod / prod / regulated>

## 2. Problem statement
<paragraph; grounded in a consumer-visible scenario, frequency, current workaround, why paved road does not solve it>

### Illustrative scenarios
- Scenario 1: ...
- Scenario 2: ...

## 3. Goals and non-goals
### Goals
1. ...
2. ...

### Non-goals
1. ...
2. ...

## 4. Proposed change
### API surface
<code-style sketch>

### Data model
<entities, fields, relationships>

### Lifecycle hooks
<hooks the platform team plugs into>

### Configuration (if any)
<config shape>

## 5. Alternatives considered
- Status quo: ... rejected because ...
- Smaller change: ... rejected because ...
- Larger change: ... rejected because ...
- Off-platform alternative: ... rejected because ...

## 6. Compatibility and migration
- Backwards compatibility: <zero-touch / opt-in / forced>
- Deprecation plan: ...
- Migration tooling: ...
- Rollout cadence (alpha → beta → GA with entry criteria): ...

## 7. Ownership, rollout, and review
- Ownership after merge: ...
- Rollout stages: ...
- Review cadence: ...

## 8. Appendix
- Co-sponsor letters of support
- Comment log
- Status transition log
```

## 3. Validation steps

Walk the RFC through this checklist before submitting it for review:

1. **Header is complete** — title specific, authors as roles, addressed-to named, date set, status set, target release named.
2. **Context links to prior audit and tradeoff doc.** RFCs that do not reference the audit row or tradeoff-doc outcome that produced them are RFCs made in a vacuum.
3. **At least one co-sponsoring consumer team is named.** A single-sponsor RFC is a team-to-team conversation; a multi-sponsor RFC is a platform-roadmap conversation.
4. **Problem statement is a paragraph, not a bullet list**, and includes frequency, current workaround, and why the paved road's current shape does not solve it.
5. **Two or more illustrative scenarios are described**, in prose, grounded in consumer workflows.
6. **Goals are 3–6, testable, and explicit.** "Should feel natural" is not a goal.
7. **Non-goals are 3–6, specific, and bound the scope.** Non-goals are deliberate exclusions, not disclaimers.
8. **Proposed change includes API surface, data model, and lifecycle hooks** in code-level detail.
9. **Alternatives considered has at least 4 options** — status quo, smaller, larger, off-platform — each rejected with a specific reason.
10. **Compatibility section names zero-touch / opt-in / forced migration** for every existing consumer class.
11. **Deprecation plan** (if any API is being replaced) names deprecation period, signal, and migration tooling.
12. **Rollout cadence has named stages** (`alpha / beta / GA`) with objective entry criteria.
13. **Ownership after merge is named** (role, team).
14. **Review cadence is named** (90-day post-GA retrospective at minimum).
15. **NIST AI RMF GOVERN cited** for the RFC-as-governance framing; MANAGE cited for the rollout-and-migration risk-control framing.
16. **OWASP ML Top 10 and MITRE ATLAS cited** in the alternatives or security-review context if the capability has a security surface.
17. **Cross-reference to `exercise-01`'s audit row and `exercise-02`'s tradeoff doc** are explicit; the RFC is downstream of both.

Static checks a CI step can enforce on an RFC file:

```bash
grep -ciE "authors|addressed to|target.*release|status"    rfc-*.md   # header complete
grep -ciE "prior audit|prior tradeoff|co-sponsor"          rfc-*.md   # context grounded
grep -ciE "goals|non-goals|alternatives considered"        rfc-*.md   # core sections
grep -ciE "backwards compatib|deprecation|migration"       rfc-*.md   # compatibility story
grep -ciE "alpha|beta|GA|rollout"                          rfc-*.md   # rollout cadence
grep -ciE "ownership|review cadence"                       rfc-*.md   # post-merge
grep -ciE "nist ai rmf|owasp|mitre atlas"                  rfc-*.md   # standards cited
```

## 4. Rubric / review checklist

- [ ] Header has title, authors (roles), addressed-to, date, status, target-release.
- [ ] Context links prior audit row (`exercise-01`), prior tradeoff doc (`exercise-02`), co-sponsoring teams, related RFCs, and consumer lifecycle.
- [ ] At least one co-sponsoring consumer team is named with a role.
- [ ] Problem statement is grounded in a consumer-visible scenario, with frequency, current workaround, and why paved road's shape does not solve it.
- [ ] Two or more illustrative scenarios are described in prose.
- [ ] Goals are 3–6 and testable.
- [ ] Non-goals are 3–6, specific, and bound scope.
- [ ] Proposed change includes API surface (code-level), data model, and lifecycle hooks.
- [ ] Alternatives considered has at least 4 options with specific rejection reasons (status quo, smaller, larger, off-platform).
- [ ] Compatibility names zero-touch / opt-in / forced migration for every existing consumer class.
- [ ] Deprecation plan has period, signal, and migration tooling.
- [ ] Rollout cadence has named stages with objective entry criteria.
- [ ] Ownership after merge is named by role.
- [ ] Review cadence is named (90-day post-GA retrospective minimum).
- [ ] NIST AI RMF GOVERN and MANAGE cited for framing.
- [ ] OWASP ML Top 10 and MITRE ATLAS cited if the capability has a security surface.
- [ ] Linkage to `exercise-01` and `exercise-02` is explicit.
- [ ] RFC is addressed to the paved-road team as *reviewers*; the proposing engineer is not the merge authority.

Review-weighted points for grading:

- Header and context discipline (prior-audit linkage, co-sponsors, lifecycle framing): 10%
- Problem statement grounding (frequency, scenarios, workaround cost): 20%
- Goals and non-goals (testable, bounded, specific): 15%
- Proposed change (API surface, data model, lifecycle hooks at code level): 20%
- Alternatives considered (≥4 options, each with specific rejection reason): 10%
- Compatibility and migration (backwards-compat framing, deprecation plan, migration tooling): 10%
- Ownership, rollout, and review (post-merge ownership named, rollout stages with entry criteria): 10%
- Standards alignment (NIST AI RMF GOVERN / MANAGE, OWASP ML Top 10 / MITRE ATLAS if applicable): 5%

## 5. Common mistakes

- **RFC as feature request.** "Please add prompt versions" is a feature request; "here is a capability gap three teams hit weekly, here is a specific shape, here is why the alternatives lose, here is how to migrate" is an RFC. The difference is the problem statement, the alternatives considered, and the migration plan.
- **Single-sponsor RFC on an org-wide platform.** Platform teams calibrate priority partly on co-sponsorship. An RFC with one sponsor says "one team wants this"; an RFC with three sponsors says "the platform has a gap." Recruit co-sponsors before drafting.
- **Problem statement as feature description.** "We need X" is not a problem; "when X does not exist, Y happens, which costs Z per quarter" is. Platform teams can estimate the value of fixing a problem; they cannot estimate the value of a feature in isolation.
- **Goals that are not testable.** "Should be easy to use" is a goal only if "easy" is measurable. "A consumer team can author and promote a prompt version with no more than three API calls" is testable.
- **No non-goals.** Platform teams read non-goals carefully; they are the artefact's commitment not to scope-creep. An RFC without non-goals is an RFC that reserves the right to expand.
- **Hand-waving the API surface.** "A new endpoint for prompt versions" is not a proposal; the method signature, input types, and return type is. Platform teams cannot estimate a hand-wave.
- **Alternatives section that only mentions the status quo.** Four alternatives, four specific rejection reasons. An alternatives section that reads "we considered doing nothing and rejected it" is one that did not do the design work.
- **Forced migration with no tooling.** Platform teams reject RFCs that would break every existing consumer unless the RFC comes with a migration script, a linter, and a committed deprecation window.
- **No ownership after merge.** The question "who maintains this after Q1?" must have an answer. An RFC whose answer is implicit (the platform team, presumably) is an RFC that produces un-maintained code.
- **No rollout stages.** A capability going straight to GA is a capability that will discover bugs on real consumers. `alpha / beta / GA` with named entry criteria is the discipline.
- **No review cadence.** A capability that ships and is never re-reviewed is a capability that may have no adopters at all. A 90-day post-GA retrospective is the cheap insurance.
- **Scope grab inside the non-goals.** Writing "the registry will not, but could" in a non-goal is a soft scope grab. If the capability is in scope, say so and own it in the goals.
- **Ignoring the platform team's roadmap.** An RFC that proposes work against a release train the platform team does not own is an RFC that will be stalled. Check the platform team's roadmap and target an acknowledged train.
- **"Co-sponsored" by teams that have not read the RFC.** Co-sponsorship is signed, not assumed. The appendix carries a short letter of support from each co-sponsor.
- **Submitting without a comment window.** RFCs are proposals, not announcements. Give the platform team and other reviewers two weeks to comment before escalating.
- **Treating rejection as failure.** A rejected RFC is often a signal that the design is wrong, the timing is wrong, or the capability belongs elsewhere — all valuable outcomes. Rejected RFCs that come back with a better shape are how platform teams and consumer teams learn each other's constraints.
- **Treating "counter-proposed" as rejection.** A counter-proposal is the platform team engaging with the RFC. The right response is usually to revise the RFC to reflect the counter-proposal, not to re-litigate the original.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds the RFC as a governance instrument that keeps cross-team capability authorship legible to leadership and reviewers. MANAGE function grounds the compatibility, migration, and review-cadence sections as risk control on the change.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Cited in the alternatives-considered section when an off-platform OSS alternative widens supply-chain surface (ML06 AI supply chain attacks), and in the compatibility section when a forced migration affects data-poisoning (ML02) or output-integrity (ML09) exposure for existing consumers.
- MITRE ATLAS — <https://atlas.mitre.org/>. Cited when the proposed capability touches surfaces exposed to the adversarial tactics catalogued in ATLAS (prompt injection, model extraction, data poisoning); the platform-team security reviewer uses ATLAS references to scope the threat-model impact of the change.

Local context (authoritative for the RFC's shape and for the cross-references):

- Paired lesson: `lessons/mod-308-platform-collaboration/exercises/exercise-03-contribute-back-rfc-to-platform.md`.
- Module README: `modules/mod-308-platform-collaboration/README.md`.
- Related exercise (paved-road audit whose `partial` and `forked` rows typically produce contribute-back candidates): `modules/mod-308-platform-collaboration/exercise-01-paved-road-consumption-audit/SOLUTION.md`.
- Related exercise (build-vs-adopt tradeoff doc whose `contribute-back` recommendation produces an RFC): `modules/mod-308-platform-collaboration/exercise-02-build-vs-adopt-tradeoff-doc/SOLUTION.md`.
- Related exercise (role hand-off map whose peer-platform category is the structural counterpart to the RFC's addressed-to field): `modules/mod-301-senior-ml-role-scope/exercise-02-role-hand-off-map/SOLUTION.md`.
- Related exercise (SLO document whose monitoring contract any capability-addition RFC typically plugs new metrics into): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (model-card and governance-review workflow whose registry a prompt-version RFC must feed into): module `mod-309-responsible-ai-governance` (exercises pending; the RFC's governance-posture framing follows the NIST AI RMF GOVERN function categories).
- Related exercise (offline evaluation harness whose lineage requirements the RFC's data model must satisfy): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
