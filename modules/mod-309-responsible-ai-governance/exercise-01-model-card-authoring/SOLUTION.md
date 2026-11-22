# exercise-01 — Model Card And Data Statement Authoring (Reference Solution)

Paired exercise: `lessons/mod-309-responsible-ai-governance/exercises/exercise-01-model-card-authoring.md`.

## 1. Solution overview

This exercise produces two linked governance artefacts for one production ML surface:

1. `model-card.md` — a Model Card that describes the trained artefact: intended use, performance by population, training data, evaluation, risks, mitigations, and the dated signatures of the owners who stand behind it.
2. `data-statement.md` — a Data Statement that describes the datasets the model was trained and evaluated on: provenance, consent basis, collection methodology, pre-processing, demographics or representational coverage, known limitations, and the retention / deletion policy.

The two artefacts are co-authored and published together because a Model Card without a Data Statement is a claim without a provenance trail, and a Data Statement without a Model Card is a dataset description with no operational consumer.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric that calibrates every learner's artefacts to the same bar.

The exercise exists because a senior ML engineer's second-order output on any production surface is a governance artefact a non-engineer can read: a Model Card is what a legal reviewer, product owner, safety team, or regulator opens to answer "what does this model do, for whom, and what are its known failure modes?"; a Data Statement is what the same reader opens to answer "where did the training data come from, who consented, and what populations are represented?" NIST AI RMF GOVERN grounds both artefacts as the governance instruments that make an AI system's intended use and limitations legible to its stakeholders; MEASURE grounds the performance-by-population and risk tables as the required characterization surface; MAP grounds the intended-use and out-of-scope sections as the context in which risk is assessed. NIST AI 600-1 (Generative AI Profile) extends the Model Card contents for generative systems — the actions in its GAI risk taxonomy (CBRN info, harmful bias, data privacy, obscene content, confabulation, information integrity, information security, intellectual property, value chain) are the categories the Model Card's risk section must explicitly address for any generative surface.

## 2. Worked answer / implementation

### 2.1 Why two artefacts, not one

A Model Card describes the **model**: what it is, how it was built, what it does, what it does poorly, where it should and should not be used, who approved it. The audience is the consumer of the model's outputs (product owner, downstream engineer, safety reviewer, end user if published).

A Data Statement describes the **data**: where it came from, who it describes, how consent was obtained, what preprocessing was applied, what populations are under- or over-represented. The audience is anyone making an *inference* about the model's behaviour from its training distribution — reviewers, auditors, researchers reusing the data.

The two artefacts have different lifecycles. A Data Statement is written once per dataset version and updated when the dataset changes; a Model Card is written once per model version and may cite multiple Data Statements (one for training, one for evaluation, one for a red-team set). Splitting them keeps both honest: a shared governance card where "data" is one bullet turns into a two-sentence summary, and a reviewer cannot tell whether the model was evaluated on a population that was never consented to be in the training set.

### 2.2 Model Card structure

The Model Card has ten required sections. Each section is short, factual, and dated. Prose that could apply to any model ("our model is state-of-the-art and ethical") is a Model Card smell and is rejected on review.

**§1. Model details.**

- **Name and version.** The exact artefact identifier used in the model registry: `assist-llm-v4.2.1`. No marketing names.
- **Owners.** Primary owner (role and team), secondary stakeholder (product owner's team), and the governance reviewer who signs the card. Named as roles, not people, so the card survives rotation.
- **Dates.** Trained-on date, release date, card last-reviewed date. Release date is the only one the user sees; the other two are what an auditor reads.
- **Model type.** Architecture family (e.g., "decoder-only transformer, 8B parameters, fine-tuned from a frozen base"), training objective, and the licensing of any base model or pretrained weights. The licence is non-negotiable: a card that omits base-model licensing cannot be approved for release.
- **Inputs and outputs.** Input modality and schema, output modality and schema, including refusal / error shapes. A reviewer must be able to understand the I/O contract without reading the serving code.
- **Links.** Model registry entry, training run id, evaluation run id, Data Statement(s) for training and evaluation, security-review artefact, fairness-slice report (exercise-03), threat-model (exercise-04).

**§2. Intended use.**

- **Primary intended uses.** Named scenarios. "Suggest a draft reply for a support agent to edit before sending to a customer" is a primary use; "customer support" is not.
- **Primary intended users.** Who the users are. "Support agents who have completed the onboarding course and are logged in through SSO" is specific; "our team" is not.
- **Out-of-scope uses.** The uses the model is *not* approved for, written as prohibitions: "Not approved for drafting medical, legal, or financial advice to end customers." "Not approved for use with PII categories listed as `restricted` in the data catalog." Out-of-scope lists are load-bearing: downstream approvals lean on them, and a vague out-of-scope is a loophole.
- **Out-of-scope users.** Who must not use the model, usually a mirror of the user population's consent boundary. "Not approved for use on EU residents until the EU-AI-Act conformity review is complete" is an out-of-scope-user line.

**§3. Factors.**

The dimensions along which performance may vary. A reviewer uses the factor list to decide which slices the performance table (§4) must break out.

- **Relevant demographic / representational groups** (if applicable to the task and if consent allows inferring or measuring these). Typical axes: language, region/locale, device type, age-range proxies.
- **Instrumental factors.** Request size, session context, time-of-day load. These are not demographics but they shape user experience; a p95-latency SLO (see `mod-307/exercise-01`) stratified by session size is a factor.
- **Environmental factors.** Deployment surface (web vs. mobile vs. internal API), upstream feature freshness, guardrail configuration.

The factor list anchors §4. A factor that appears in §3 but has no row in §4 is a card smell; §4 is where the factors are *measured*.

**§4. Metrics and performance.**

A table, one row per factor-slice × metric. Columns:

| Metric | Slice | Dataset | N | Point estimate | 95% CI | Baseline / threshold | Notes |

- **Metric** is a defined SLI (availability, latency, quality, fairness metric from exercise-03, robustness score from exercise-04).
- **Slice** is the factor value ("en-US", "pt-BR", "mobile", "session_size > 50").
- **Dataset** names the evaluation set and cites the Data Statement for it.
- **N** is the slice sample size. A row where N is below the pre-registered minimum (e.g., 500) is flagged as low-power rather than suppressed.
- **Point estimate** and **95% CI** are both required for any metric reported. A point estimate without a CI is a vibe, not a measurement.
- **Baseline / threshold** cites the launch gate or prior release the slice is compared against.
- **Notes** record anything the reviewer must know: "slice excludes requests flagged by upstream PII filter," "metric from offline eval only, not live traffic."

A performance section without CIs, without slices, or with only an overall number is not a Model Card performance section — it is marketing.

**§5. Evaluation details.**

- **Evaluation data.** Names the dataset(s) and cites the Data Statement(s) for each.
- **Evaluation methodology.** Point-in-time correctness, how the eval set was constructed (sampled from traffic? labelled by humans? synthetic?), how labels were created, and the inter-annotator agreement if human-labelled.
- **Hardware / software.** Model server version, inference-time configuration, batching, hardware class. Different inference configs can move latency by a factor; if the eval ran on an experimental config, say so.
- **Reproducibility.** Evaluation run id and the command/pipeline-name that re-produces it. An un-reproducible eval is an un-review-able eval.

**§6. Training data.**

A two- to four-paragraph summary of the training data — enough for a reviewer to form a mental model, with a prominent pointer to the full Data Statement. This section is a summary of the Data Statement, not a replacement. Include:

- Scope (what domain, what time window).
- Approximate size (number of examples, approximate token count for text).
- Provenance and consent basis (one-line; the Data Statement elaborates).
- Known gaps (languages, locales, user cohorts the dataset under-represents).

**§7. Quantitative analyses / fairness.**

- **Slice-level performance** from §4, interpreted: are there slices with materially worse performance than the overall? Materially worse is defined (e.g., "> 20% relative gap on the primary quality SLI" is the threshold used in exercise-03's cadence).
- **Fairness metrics** reported on the slices where they apply. The choice of fairness metric (demographic parity, equalized odds, equal opportunity, calibration within groups, generalized entropy) is justified with reference to the task and the harm model — a classifier whose error distribution matters most (e.g., false negatives on a safety filter) uses a fairness metric that measures that error, not an unrelated one.
- **Robustness metrics.** Adversarial-input success rate on the MITRE ATLAS-aligned probe set (see exercise-04), prompt-injection success rate for generative surfaces, distribution-shift performance degradation.

**§8. Ethical considerations and risks.**

A short section that names, in plain language, the risks the model carries and the mitigations in place. For generative surfaces, each of the NIST AI 600-1 GAI risks is addressed or explicitly ruled out-of-scope with a reason:

- CBRN information or capability uplift risk.
- Confabulation (hallucination of facts or fabricated references).
- Dangerous, violent, or hateful content.
- Data privacy / training-data memorization.
- Environmental impact (compute cost and energy of training / serving if material).
- Harmful bias and homogenization.
- Human-AI configuration risks (automation bias, over-reliance, inappropriate anthropomorphism).
- Information integrity (misleading content, disinformation support).
- Information security (prompt injection, jailbreaks, extraction attacks — cross-reference MITRE ATLAS tactics used in exercise-04).
- Intellectual property (copyrighted material in training data, licence leakage).
- Obscene, degrading, or abusive content.
- Value-chain and component integration risks (upstream model licence, downstream misuse).

For a non-generative surface, use the OWASP ML Top 10 as the anchor taxonomy: input manipulation, data poisoning, model inversion, membership inference, model theft, supply-chain, transfer-learning risks, model skewing, output integrity attacks, denial of ML service.

Each risk row has: a one-line description, the mitigation in place (model-side control, input filter, monitoring, human review), the residual risk after mitigation, and the owner.

**§9. Caveats and recommendations.**

Human-readable guidance for downstream consumers. "Do not surface model outputs to end users without an edit step." "Do not use for high-stakes automated decisions without the human-in-the-loop configuration." These are not re-prints of §2's out-of-scope list — they are the recommendations a knowledgeable consumer gives a less-knowledgeable one.

**§10. Governance and sign-off.**

- **Approvers.** Three dated signatures: the model owner, the product owner, the governance reviewer. A card with fewer than three signatures is a draft.
- **Approval scope.** Which deployment surface the approval covers (e.g., "internal agent-assist draft-reply surface only; not approved for externally-facing user flows"). An approval is not globally valid; it is scoped.
- **Expiry / re-review trigger.** The date by which the card is re-reviewed, and the triggers that force out-of-cycle re-review: model retrain, dataset refresh, production incident, new risk identified, new user population onboarded, new regulatory guidance. A card with no expiry is a card the team will forget to re-visit.
- **Change log.** A dated list of material changes to the card since v1: which rows changed, who approved, why.

### 2.3 Data Statement structure

The Data Statement describes each dataset the model was trained or evaluated on. One Data Statement per dataset version; a model with a training set, an evaluation set, and a red-team set cites three Data Statements.

**§1. Curation rationale.** Why the dataset exists, what question it was built to answer, who requested it. A line of intent so a future reviewer can see whether the dataset is being reused for a purpose outside its original mandate (which should trigger a re-review).

**§2. Language variety / content scope.** For text: the language codes represented (e.g., `en-US`, `pt-BR`, mixed-code), the registers (formal, informal, slang), the topical domains. For non-text: the content modality, the subject matter, the environmental distribution (daylight vs. nighttime images, studio vs. in-the-wild audio). Scope informs factor selection in the Model Card (§3); a dataset that is English-only must appear as a factor in the Model Card and as an out-of-scope language note.

**§3. Source and provenance.**

- **Original source.** First-party collection, third-party licensed, scraped from public web, user-contributed, synthetic / model-generated. Be exact. "Public web" is not a provenance; "pages crawled under the Common Crawl 2024-Q4 snapshot, filtered to the news domain" is.
- **Access path and licence.** How the data was obtained, under what licence, and what redistribution rights the licence grants. The licence constrains what the model may or may not do downstream.
- **Date range of collection.** When the data was collected; this bounds what the model knows.
- **Chain of custody.** The internal pipeline that moved the data from source to training feature store; the pipeline's lineage artefact (see exercise-02) is cited here.

**§4. Speaker / subject demographics (where applicable and where consent allows inferring).** For user-generated data: the population distribution, including known under-represented groups. If demographics were not collected or inferred, say so explicitly ("no demographic annotation available for this dataset"); silence is confusing, honesty is operable.

**§5. Annotator demographics (if the data was human-labelled).** The annotator population, the training the annotators received, the pay rate and the vendor contract if applicable, the inter-annotator agreement on the label task. Annotator demographics are a known source of annotation bias; a Data Statement that omits them cannot be reviewed for that bias.

**§6. Speech situation / context of collection.** The conditions under which the data was produced: in-product user interactions, structured elicitation, scraping of pre-existing content, synthetic generation from another model. Each has different consent and representativeness implications.

**§7. Text / content characteristics.** Pre-processing applied (deduplication, filtering, normalization, PII redaction method and residual rate from exercise-02), the fraction removed by each step, the resulting dataset size.

**§8. Consent and legal basis.**

- **Consent basis for user-sourced data.** Terms-of-service consent language, specific opt-in, legitimate-interest claim, exemption basis. Named by jurisdiction where relevant.
- **Purpose limitation.** The purposes the dataset is approved for; the Data Statement's reuse for a model outside those purposes requires re-review.
- **Retention policy.** How long the raw dataset is retained, under what conditions it is destroyed, and the schedule for review.
- **Deletion mechanism.** How a user's data is removed from the dataset on deletion request (full-dataset re-materialization, soft-delete flag plus re-training exclusion, filtering at use-time).
- **Jurisdictional considerations.** EU / UK / US / other region-specific requirements that apply.

**§9. Recommendations for use.** The uses the dataset is appropriate for and the uses it is not. A dataset built for intent classification should state whether it is appropriate for sentiment classification or not. Match the shape of the Model Card §2 "intended use" section; the two cross-check each other.

**§10. Known limitations.** Representativeness gaps, known noise patterns, known label errors, known distribution-shift properties. A Data Statement without a limitations section has not been written honestly.

**§11. Governance.** Owner, last-reviewed date, next-review trigger, change log. Mirrors Model Card §10.

### 2.4 How the two artefacts cross-check

Several contents of the Model Card are derived from the Data Statement, and the artefacts must be internally consistent:

- Model Card §2 out-of-scope uses must include populations not represented in the training set (per Data Statement §4). An English-only training set means "not approved for non-English content" is a required out-of-scope line.
- Model Card §3 factors must include any axis where the Data Statement §4 or §5 identifies under-representation.
- Model Card §4 metrics must report on the factors from §3 — the Data Statement feeds which slices matter.
- Model Card §7 fairness analysis cites the dataset(s) and their Data Statements.
- Model Card §8 risk rows for data-privacy and training-data memorization must align with the Data Statement §8 consent basis and retention policy.

A card-and-statement pair where the Model Card's §3 factor list does not match the Data Statement's §4 subject distribution is internally inconsistent and is rejected on review. A CI lint can enforce this (see §3 validation steps).

### 2.5 Illustrative worked example (not a real system)

> **Illustrative feature** — `agent-assist/draft`, an internal LLM-backed support-agent assistant. This example is a composite for calibration; do not treat it as a real system or incident.

#### Model Card excerpt

```markdown
# Model Card: assist-llm-v4.2.1

## §1. Model details
- Name/version: `assist-llm-v4.2.1`
- Owner: `team-agent-assist` (L30 senior ML engineer, primary); `team-support-ops` (product owner)
- Governance reviewer: `team-governance`
- Trained: 2026-10-14; Released: 2026-11-02; Card last reviewed: 2026-11-15
- Model type: decoder-only transformer, 8B parameters, fine-tuned from licensed
  base model `base-llm-v3` (licence: internal commercial use, no redistribution).
- Inputs: support-conversation context (JSON schema `assist/draft_v4.input`),
  max 8k tokens. Outputs: draft-reply text (schema `assist/draft_v4.output`),
  plus `refusal` object if safety classifier triggers.
- Links: registry `assist-llm-v4.2.1`; training run `tr-20261014-...`; eval run
  `ev-20261101-...`; Data Statements: `training-ds-v4.md`, `eval-ds-v4.md`,
  `red-team-ds-v4.md`; threat model `exercise-04`; fairness report `exercise-03`.

## §2. Intended use
- Primary use: draft a reply a support agent edits before sending to a customer.
- Primary users: support agents in queues `billing`, `shipping`, `account-mgmt`
  who have completed the onboarding course.
- Out-of-scope uses: NOT approved for drafting medical, legal, or financial
  advice; NOT approved for direct-to-customer autonomous reply; NOT approved
  for use on queues flagged `safety-sensitive`.
- Out-of-scope users: NOT approved for use on EU-resident tickets until the
  EU-AI-Act conformity review (tracked in `governance-registry`) completes.

## §3. Factors
- Language: en-US (primary), pt-BR (secondary), es-MX (secondary). All others
  out of scope.
- Queue type: billing, shipping, account-mgmt.
- Session size: short (< 5 turns), medium (5–20 turns), long (> 20 turns).
- Model config: default decoding; chain-of-thought disabled; temperature 0.2.

## §4. Metrics and performance (excerpt)
| Metric | Slice | Dataset | N | Est. | 95% CI | Threshold | Notes |
|---|---|---|---|---|---|---|---|
| Agent-acceptance rate | en-US, billing | `eval-ds-v4.md` | 4,812 | 0.72 | [0.70, 0.74] | ≥ 0.65 | — |
| Agent-acceptance rate | pt-BR, billing | `eval-ds-v4.md` | 711 | 0.58 | [0.54, 0.62] | ≥ 0.65 | Below threshold; see §9 |
| Schema-validity rate | all | `eval-ds-v4.md` | 12,340 | 0.996 | [0.994, 0.997] | ≥ 0.99 | — |
| Prompt-injection success | adversarial | `red-team-ds-v4.md` | 1,200 | 0.03 | [0.02, 0.04] | ≤ 0.05 | MITRE ATLAS AML.T0051 |
| Refusal rate, safety probes | safety-probe set | `red-team-ds-v4.md` | 500 | 0.98 | [0.97, 0.99] | ≥ 0.97 | — |

(Full table has 24 rows: 4 primary metrics × 6 slices.)

## §7. Quantitative analyses / fairness (excerpt)
- Slice gap: pt-BR acceptance rate is 19% relative worse than en-US on billing.
  This is below the launch gate; a mitigation action is open (expand pt-BR
  training coverage; see Data Statement `training-ds-v4.md` §10). Deployment
  for pt-BR is kept under a time-boxed waiver through 2027-Q1 pending
  mitigation; en-US remains the default for cross-locale fall-through.
- Fairness metric choice: agent-acceptance rate parity across language slices,
  because the harm model is "agents spend more edit time on worse drafts"; an
  equal-opportunity metric is reported for the safety-probe subset only.

## §8. Ethical considerations and risks (excerpt; GAI taxonomy)
| Risk | Mitigation | Residual | Owner |
|---|---|---|---|
| Confabulation | RAG grounding on ticket history; classifier flags unsupported claims; agents edit before send. | Low — agent-mediated. | `team-agent-assist` |
| Harmful bias | Fairness slices enforced at release; pt-BR waiver tracked. | Medium (pt-BR gap). | `team-agent-assist` + `team-governance` |
| Data privacy / memorization | Training data PII-redacted (see `training-ds-v4.md` §7); membership-inference probe shows < random+ε attack success. | Low. | `team-agent-assist` + `team-data-platform` |
| Information security (prompt injection) | Input sanitizer; adversarial-probe gate at 5% success rate (see §4). | Low — below threshold. | `team-agent-assist` + `team-security` |
| Human-AI configuration (over-reliance) | Agent edit step is a required UX element; UI shows diff. | Medium — mitigated by UX, not technology. | `team-support-ops` |
| CBRN uplift | Out of scope: not a knowledge-domain surface. | Not applicable. | — |

(Full table has 12 rows, one per NIST AI 600-1 GAI risk; items marked N/A carry a reason.)

## §10. Governance and sign-off
- Approvers: `team-agent-assist` tech-lead (2026-11-02); `team-support-ops` PD
  (2026-11-02); `team-governance` reviewer (2026-11-04).
- Approval scope: internal agent-assist draft-reply surface, en-US primary.
- Expiry: 2027-02-02 (90 days) OR earlier on any of: retrain, dataset refresh,
  incident with governance attribution, new user population, new GAI regulation.
- Change log: v4.2.1 – added pt-BR slice row and waiver; v4.2.0 – initial release.
```

#### Data Statement excerpt

```markdown
# Data Statement: training-ds-v4

## §1. Curation rationale
Dataset built to fine-tune `assist-llm-v4` on support-queue draft replies. Built
on request from `team-agent-assist` in 2026-Q2, under governance approval
`gov-req-2026-04-ds-v4`.

## §2. Language variety / content scope
Languages: en-US (87%), pt-BR (9%), es-MX (3%), other (<1%, dropped in
preprocessing). Registers: semi-formal customer-support text. Topical scope:
billing, shipping, account management; explicitly excluded: medical, legal,
financial advice.

## §3. Source and provenance
First-party collection from the support-ticketing system, tickets closed
between 2024-01-01 and 2026-06-30. Access path: `data-platform`
pipeline `support-ticket-export → draft-training-v4`, lineage artefact
`lineage-training-ds-v4.yaml` (see exercise-02).

## §4. Subject demographics (customer-side)
No explicit demographic annotation. Known representational gaps:
- Pt-BR and es-MX are under-represented relative to production traffic
  (prod: ~17% pt-BR, ~6% es-MX; dataset: 9% / 3%).
- Accessibility-flagged tickets (screen-reader, keyboard-only indicators)
  under-represented by ~40% vs. production.

## §5. Annotator demographics
Labels are implicit (agent-sent reply = positive). No external annotators used
for v4 training set. Red-team set (`red-team-ds-v4.md`) used external
annotators; see that statement.

## §7. Content characteristics / preprocessing
- Deduplication: template-reply hash dedup → removed 11%.
- PII redaction: `pii-redactor-v3` applied to customer messages and agent
  replies. Residual PII rate on sample audit: 0.3% (target: < 0.5%); see
  exercise-02 PII register for the categories covered.
- Tickets flagged `restricted` or `safety-sensitive` excluded.
- Final size: 1.8M examples; approximate 620M tokens.

## §8. Consent and legal basis
- Consent basis: ToS §7.4 (2023-05 revision), legitimate-interest claim for
  internal product improvement, documented in `legal-register/entry-0412`.
- Purpose limitation: internal support-queue assistants and related
  auxiliary models; NOT approved for external-facing LLM services or
  model-sale / model-licensing.
- Retention: raw tickets retained 36 months per data-retention policy;
  training-dataset snapshots retained 12 months and recompiled on each
  model retrain.
- Deletion: user deletion requests propagate within 30 days via
  `user-deletion-service → training-pipeline-exclusion-list`; the next
  training recompile honours the exclusion. See exercise-02 for the lineage.
- Jurisdictional: EU-resident tickets excluded from v4 training per legal
  guidance pending EU-AI-Act review.

## §10. Known limitations
- Language distribution skew noted in §4.
- Temporal skew: 2024 tickets over-represented before dedup; after dedup
  the distribution is approximately uniform across the window.
- Label noise: implicit-label scheme means "agent sent this reply" is the
  positive signal; sent-but-edited-heavily drafts are counted as positive.
  A quality label study (2026-Q3) estimated 8% of positives had material
  edits; mitigation is deferred to v5 with explicit edit-distance labelling.

## §11. Governance
Owner: `team-data-platform` (data owner); `team-agent-assist` (consumer).
Last reviewed: 2026-10-20. Next review trigger: v5 training-set build, or
any change to retention / deletion policy, or any legal-register update.
```

### 2.6 Lightweight linter sketch

Keep the artefacts honest with a small linter that reads both Markdown files and asserts the discipline. The linter runs in CI and is not a substitute for a human reviewer.

```python
# card_lint.py
"""Lint a Model Card and its paired Data Statement(s) for structural completeness.

A reviewer still reads both artefacts. This script enforces the mechanical
discipline so the reviewer's attention goes to the judgement calls.
"""

from dataclasses import dataclass
import re
import sys


MODEL_CARD_SECTIONS = [
    "Model details", "Intended use", "Factors", "Metrics and performance",
    "Evaluation details", "Training data", "Quantitative analyses",
    "Ethical considerations", "Caveats", "Governance",
]

DATA_STATEMENT_SECTIONS = [
    "Curation rationale", "Language variety", "Source and provenance",
    "demographics", "Annotator", "Speech situation", "characteristics",
    "Consent and legal basis", "Recommendations for use",
    "Known limitations", "Governance",
]

GAI_RISKS = [
    "CBRN", "confabulation", "dangerous", "data privacy", "environmental",
    "harmful bias", "human-AI configuration", "information integrity",
    "information security", "intellectual property", "obscene",
    "value chain",
]


@dataclass
class Finding:
    severity: str
    message: str


def lint_model_card(text: str) -> list[Finding]:
    f: list[Finding] = []
    lowered = text.lower()
    for s in MODEL_CARD_SECTIONS:
        if s.lower() not in lowered:
            f.append(Finding("error", f"missing section: {s}"))
    if not re.search(r"version[:\s]+\S", text, re.I):
        f.append(Finding("error", "no model version declared"))
    if "out of scope" not in lowered and "out-of-scope" not in lowered:
        f.append(Finding("error", "no out-of-scope use declared"))
    if "ci" in lowered or "95%" in lowered or "confidence interval" in lowered:
        pass
    else:
        f.append(Finding("warn", "no confidence intervals found in metrics table"))
    if not re.search(r"expir", lowered) and "re-review" not in lowered:
        f.append(Finding("error", "no card expiry or re-review trigger"))
    if not re.search(r"sign.{0,5}off|approver|approved by", lowered):
        f.append(Finding("error", "no sign-off / approver record"))
    missing_risks = [r for r in GAI_RISKS if r.lower() not in lowered]
    if missing_risks:
        f.append(Finding("warn", f"GAI risks not referenced: {missing_risks}"))
    if "nist ai rmf" not in lowered and "nist.ai" not in lowered:
        f.append(Finding("warn", "NIST AI RMF not cited"))
    return f


def lint_data_statement(text: str) -> list[Finding]:
    f: list[Finding] = []
    lowered = text.lower()
    for s in DATA_STATEMENT_SECTIONS:
        if s.lower() not in lowered:
            f.append(Finding("error", f"missing section: {s}"))
    if "retention" not in lowered:
        f.append(Finding("error", "no retention policy declared"))
    if "delet" not in lowered:
        f.append(Finding("error", "no deletion / removal mechanism declared"))
    if "consent" not in lowered and "legal basis" not in lowered:
        f.append(Finding("error", "no consent basis or legal basis declared"))
    return f


if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("usage: card_lint.py <model-card.md> <data-statement.md> [more-ds.md ...]")
        sys.exit(2)
    mc_text = open(sys.argv[1]).read()
    mc_findings = lint_model_card(mc_text)
    ds_findings: list[Finding] = []
    for ds_path in sys.argv[2:]:
        ds_findings += lint_data_statement(open(ds_path).read())

    for f in mc_findings:
        print(f"model-card {f.severity:>5}: {f.message}")
    for f in ds_findings:
        print(f"data-stmt  {f.severity:>5}: {f.message}")
    errors = [f for f in (mc_findings + ds_findings) if f.severity == "error"]
    sys.exit(1 if errors else 0)
```

## 3. Validation steps

Walk the artefacts through this checklist before considering them done:

1. **Model Card §1 pins the exact artefact.** Named version, dated, signed, scoped to a surface. "Our model" is not acceptable.
2. **Model Card §2 has an out-of-scope list written as prohibitions.** Vague aspirational language ("be careful when") is not an out-of-scope list.
3. **Every factor in §3 appears as a slice in §4.** A listed factor with no measurement is a card smell.
4. **Metrics table has N, point estimate, 95% CI, and threshold.** No CI → the number is a vibe.
5. **Low-power rows are flagged, not suppressed.** A slice below the pre-registered N minimum is marked low-power with a next-step; it does not vanish.
6. **§7 fairness analysis justifies the choice of fairness metric** in terms of the task's harm model, not generic parity language.
7. **§8 risk section addresses every GAI risk** from NIST AI 600-1 (for generative) or every category from OWASP ML Top 10 (for classical ML), with explicit N/A rationales.
8. **§10 sign-off has three dated signatures** and a named expiry / re-review trigger.
9. **Data Statement names the source, licence, consent basis, retention, and deletion mechanism.** All five are required; missing any is an error.
10. **Data Statement §4 subject distribution matches the Model Card §3 factor list.** A locale in the training data must appear as a factor or as an explicit out-of-scope locale.
11. **Data Statement §10 limitations is non-empty.** Every dataset has limitations; an empty limitations section is a card smell.
12. **Artefacts cross-reference.** The Model Card cites the Data Statement(s) by filename and the Data Statements cite the Model Card as a consumer.
13. **Lineage is traceable.** The Data Statement's §3 cites a lineage artefact (exercise-02); the Model Card's §1 cites the training-run and eval-run IDs.
14. **NIST AI RMF is cited** (GOVERN / MEASURE / MAP functions) in the Model Card. NIST AI 600-1 is cited on the Model Card §8 risk section for generative surfaces.

Static checks a CI step can enforce:

```bash
grep -ciE "intended use|out.of.scope|factors|metrics" model-card.md
grep -ciE "95%|confidence interval|\[0\.|CI" model-card.md
grep -ciE "nist ai rmf|govern|measure|map"            model-card.md
grep -ciE "retention|consent|legal basis|delet"       data-statement.md
grep -ciE "limitations|known gaps|under.represent"    data-statement.md
```

`card_lint.py` from §2.6 gives a programmatic version of the same checks.

## 4. Rubric / review checklist

- [ ] Model Card has all ten required sections, each non-generic and non-empty.
- [ ] §1 names version, trained-on / released / last-reviewed dates, model type, base-model licence, inputs/outputs, and registry link.
- [ ] §2 names primary use, primary users, out-of-scope uses, out-of-scope users — all specific.
- [ ] §3 factor list anchors §4; every factor has a slice in the metrics table.
- [ ] §4 reports each metric × slice with N, point estimate, 95% CI, and launch-gate threshold.
- [ ] §4 flags low-power rows rather than suppressing them.
- [ ] §5 evaluation details include methodology, inter-annotator agreement (if applicable), and reproducibility pointers.
- [ ] §6 training-data summary is a summary of the Data Statement and cites it by filename.
- [ ] §7 fairness choice is justified by the task's harm model; robustness metrics cross-reference MITRE ATLAS tactics from exercise-04.
- [ ] §8 risk table addresses all NIST AI 600-1 GAI risks (for generative) or all OWASP ML Top 10 categories (for classical ML), with explicit N/A rationales.
- [ ] §9 caveats are human-readable guidance, not a reprint of §2.
- [ ] §10 has three dated signatures (model owner, product owner, governance reviewer), named approval scope, expiry / re-review triggers, and a change log.
- [ ] Data Statement has all eleven required sections, each non-empty.
- [ ] §3 provenance names source, licence, access path, and lineage artefact (exercise-02).
- [ ] §4 subject distribution and §5 annotator distribution are declared, including "not collected" where applicable.
- [ ] §7 pre-processing names the PII redactor and the residual-PII sample-audit rate.
- [ ] §8 consent and legal basis names the basis, the purpose limitation, the retention schedule, the deletion mechanism, and jurisdictional notes.
- [ ] §10 limitations is non-empty and ties to known representational gaps.
- [ ] §11 governance has an owner, a last-reviewed date, and a next-review trigger.
- [ ] Artefacts are internally consistent: Model Card §3 factors match Data Statement §4 subject distribution; Model Card §8 data-privacy row aligns with Data Statement §8 consent / retention / deletion.
- [ ] NIST AI RMF (GOVERN / MEASURE / MAP) is cited in the Model Card; NIST AI 600-1 is cited for generative surfaces.
- [ ] OWASP ML Top 10 is cited for classical-ML surfaces or on security rows of generative surfaces.
- [ ] Cross-reference to exercise-02 (lineage artefact), exercise-03 (fairness slice cadence), exercise-04 (threat model).

Review-weighted points for grading:

- Model Card §1–§3 scope/intended-use/factors clarity: 15%
- Model Card §4 metrics table with CIs and slices: 15%
- Model Card §7 fairness analysis with justified metric choice: 10%
- Model Card §8 risk coverage against NIST AI 600-1 / OWASP ML Top 10: 15%
- Model Card §10 governance (signatures, scope, expiry): 10%
- Data Statement §3 provenance and §8 consent/retention/deletion: 15%
- Data Statement §4 / §5 / §10 representational honesty and limitations: 10%
- Cross-consistency between Model Card and Data Statement(s): 10%

## 5. Common mistakes

- **"State-of-the-art and ethical" prose.** A Model Card is a factual document; marketing adjectives are rejected on review. The question is "what does this model do, for whom, and what are its known failure modes?" — answered in sentences a non-engineer can audit.
- **Overall metrics only, no slices.** An aggregate accuracy hides the slice where the model fails the hardest. Every factor in §3 must have a row in §4.
- **No confidence intervals.** A point estimate without a CI is a vibe. A 0.72 agent-acceptance rate on an N=4,000 sample is different from an N=40 sample; the CI is what makes the difference legible.
- **Out-of-scope list as aspiration.** "We hope this won't be used for medical advice" is not out-of-scope; "NOT approved for drafting medical advice" is.
- **Risk section that says "we considered bias."** Each GAI risk (or OWASP category) is a row with a mitigation, a residual, and an owner. A paragraph that lists categories without rows is a Model Card smell.
- **Fairness metric unjustified.** Equalized odds, demographic parity, and calibration are not interchangeable. The choice is driven by the harm model and must be justified in the card.
- **Base-model licence omitted.** A fine-tuned Model Card must declare the base-model licence and the redistribution rights. A card without this is not approvable.
- **Data Statement without limitations.** Every dataset has limitations; an empty §10 signals that the data owner did not look.
- **Consent basis missing.** A Data Statement without a consent basis cannot be used to justify training. "Internal data" is not a consent basis; the terms-of-service clause, legal basis, or exemption is.
- **No deletion mechanism.** A Data Statement must name how a user's data leaves the dataset on deletion request. "We delete it" is not a mechanism; "exclusion-list propagation to next training recompile" is.
- **Model Card § and Data Statement § inconsistent.** A pt-BR slice in the Model Card §4 is incompatible with a Data Statement §4 that says "no Portuguese-language data" — the two artefacts audit each other.
- **No signatures or expiry.** An unsigned card is a draft; a card without expiry is a card the team will forget to re-review. Both are rejected.
- **No cross-reference to downstream artefacts.** The fairness report (exercise-03) and the threat model (exercise-04) belong in the Model Card's §1 links. A card that stands alone is a card without a system behind it.
- **"Lessons from" copy-paste.** A card for a new model that re-uses the risk table verbatim from a sibling model has not been re-reasoned. Risks are revisited per model version.
- **Generic generative-AI boilerplate.** NIST AI 600-1's GAI risks apply but they apply *to this model*. "Harmful bias — mitigated by training on diverse data" without a slice table in §4 is boilerplate; "Harmful bias — pt-BR acceptance 19% below en-US; time-boxed waiver through 2027-Q1" is the operable form.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds the Model Card and Data Statement as governance instruments that keep intended use, limitations, and accountability legible. MAP function grounds the intended-use and factor sections as the context in which risk is characterized. MEASURE function grounds the performance-by-population table as the required characterization surface.
- NIST AI 600-1 (Generative AI Profile) — <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>. The GAI risk taxonomy (CBRN info, confabulation, dangerous/violent/hateful content, data privacy, environmental, harmful bias, human-AI configuration, information integrity, information security, intellectual property, obscenity/degradation, value chain) is the required set of risk categories the Model Card §8 addresses for generative surfaces.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Used as the risk taxonomy for the Model Card §8 risk table on classical-ML surfaces (input manipulation, data poisoning, model inversion, membership inference, model theft, supply-chain attacks, transfer-learning risks, model skewing, output-integrity attacks, denial of ML service) and as a cross-check on the security rows of generative surfaces.
- MITRE ATLAS — <https://atlas.mitre.org/>. Cited on §4 robustness rows and §8 information-security rows for the adversarial-tactic taxonomy against which probe sets are built (e.g., AML.T0051 LLM Prompt Injection, AML.T0043 Craft Adversarial Data, AML.T0024 Exfiltration via ML Inference API). See exercise-04 for the threat-model artefact.

Practitioner implementation examples:

- VeriSwarm Trust Center — <https://veriswarm.ai/trust>. Example of how a practitioner publishes a model card / trust page for external stakeholders; useful only as a shape-of-artefact reference, not as a standard.

Local context (authoritative for the artefact shape and cross-references):

- Paired lesson: `lessons/mod-309-responsible-ai-governance/exercises/exercise-01-model-card-authoring.md`.
- Module README: `modules/mod-309-responsible-ai-governance/README.md`.
- Related exercise (data lineage graph + PII register the Data Statement §3 and §7 cite): `modules/mod-309-responsible-ai-governance/exercise-02-data-lineage-and-pii-audit/SOLUTION.md`.
- Related exercise (fairness / robustness slice cadence that feeds Model Card §4 and §7): `modules/mod-309-responsible-ai-governance/exercise-03-fairness-and-robustness-slice-cadence/SOLUTION.md`.
- Related exercise (misuse and adversarial threat model that feeds Model Card §8 information-security rows and §4 robustness rows): `modules/mod-309-responsible-ai-governance/exercise-04-misuse-and-adversarial-threat-model/SOLUTION.md`.
- Related exercise (offline evaluation harness whose slice definitions populate Model Card §4): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (SLO document whose quality SLIs the Model Card §4 "online" metrics align with): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
