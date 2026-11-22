# exercise-04 — Misuse And Adversarial Threat Model (Reference Solution)

Paired exercise: `lessons/mod-309-responsible-ai-governance/exercises/exercise-04-misuse-and-adversarial-threat-model.md`.

## 1. Solution overview

This exercise produces `threat-model.md` — a written threat model for one production ML surface, enumerating the misuse scenarios and adversarial attacks the surface is exposed to, the mitigations in place, the residual risk after mitigation, and the probe suite that measures whether each mitigation holds release-to-release. The artefact has seven parts:

1. A **system under threat model** section that pins down the surface, its deployment context, its trust boundaries, and the data / tool / model surface the attacker can reach.
2. An **asset and actor inventory** — what the attacker can gain (trained-model IP, training data, downstream user data, abuse-of-outputs leverage, reputational harm) and who the attackers are (external anonymous user, authenticated external user, insider, compromised upstream, compromised downstream).
3. An **attack catalog** — the enumerated misuse / adversarial threats, each tagged to MITRE ATLAS techniques and OWASP ML Top 10 categories, with a short attacker-narrative, the data / API / model path exercised, and the preconditions for success.
4. A **mitigation register** — for every attack row, the in-place controls (model-side, input-side, output-side, policy-side, monitoring-side), the residual risk, and the owner of each control.
5. A **probe suite** — the dated set of adversarial inputs the surface is re-evaluated against on every release and on the nightly safety cadence (see exercise-03). Each probe is linked to one or more attack rows.
6. An **incident and disclosure playbook** — what the team does when a probe-set regression is detected, when an in-the-wild attack is observed, or when a vulnerability is disclosed externally.
7. A **review cadence** — when the threat model is re-run, who signs off, and what triggers (new user surface, new tool / RAG index, new model class, new disclosed technique) force an out-of-cycle re-review.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric that calibrates every learner's threat model to the same bar.

The exercise exists because a Model Card §8 risk table (exercise-01) and a safety slice cadence (exercise-03) are measurements; the threat model is the *reasoning* that chooses which measurements matter. Without a threat model, a probe suite is a bag of tests written against whichever attack came to mind during offline eval. With one, the probe suite covers the attacker's cost–benefit surface — the attacks that are cheap to try, the attacks that are high-value to succeed, the attacks a specific adversary class is likely to attempt. NIST AI RMF MAP function grounds the threat model as the risk-characterization surface for an AI system; MANAGE function grounds the mitigation register as the risk-control surface; GOVERN function grounds the incident / disclosure playbook as the accountability surface. OWASP ML Top 10 and MITRE ATLAS provide the taxonomy; NIST AI 600-1 provides the generative-AI-specific risk framing (information security, information integrity, data privacy, intellectual property, value chain).

## 2. Worked answer / implementation

### 2.1 System under threat model

Six lines, in this order, so a reviewer can situate the threat model before reading the catalog:

- **Surface.** The user-facing endpoint, model version, and primary request shape. "Our LLM" is not a surface; "`assist-llm-v4.2.1` serving draft replies to internal support agents via the `agent-assist/draft` endpoint" is.
- **Deployment context.** Where the surface runs, who can reach it, what authentication / authorization gates sit in front of it. "Internal-only, SSO-gated, available to agents in queues `billing / shipping / account-mgmt` only" is a context; "we have auth" is not.
- **Trust boundaries.** Which parties sit on each side of each boundary. Attacker vs. defender is not binary — a trust boundary sits between the model and the tool-calling layer, between the model and the retrieval corpus, between the serving layer and the training pipeline, between the training pipeline and upstream sources.
- **Data / API / model surface.** The exact components the attacker can interact with — the request body schema, the retrieval index the model reads, the tools the model calls, the training feedback loop (if any) the model's outputs feed.
- **Audit owner.** The engineer running the threat model and the role they hold. A threat model is signed, not anonymous.
- **Last reviewed.** A date, within the declared review cadence (default quarterly; see §2.7).

The surface section must be the *first* thing a reviewer reads; a threat model without a pinned surface is a threat model for no system in particular.

### 2.2 Asset and actor inventory

**Assets** — what an attacker can gain. The threat model names the top-value assets and ranks them; mitigations are prioritized against the most valuable asset the surface exposes.

- **Model IP.** The weights, the fine-tuning dataset, the system prompt / policy prompts, the tool definitions. Extraction has intellectual-property and competitive-advantage value.
- **Training data.** Memorized content the model emits under extraction attacks (OWASP ML03 Model Inversion, ML04 Membership Inference). Training-data-level PII is a specific sub-asset.
- **Downstream user data.** For a surface that reads upstream resources (RAG corpora, tool outputs, feature stores), the data behind those surfaces is reachable through the model if an injection succeeds.
- **Output abuse leverage.** The ability to make the surface produce outputs the attacker can use elsewhere — phishing drafts, toxic content, policy-evasive content, operational instructions. The attacker's gain is downstream, outside the surface.
- **Operational capacity.** Denial-of-ML-service (OWASP ML10): an attacker makes the model expensive to run, exhausting cost budgets or latency SLOs, achieving disruption without needing output-level success.
- **Reputational / regulatory harm.** The attacker's gain is the negative externality on the defender — a surface that is caught producing policy-violating content is a public-trust loss and a potential regulatory trigger.

**Actors** — who is attacking. The threat model names the actor classes and the capabilities each has.

- **External anonymous user.** No authentication. Low capability, high volume. For an externally-exposed API, the dominant class.
- **External authenticated user.** Can authenticate, has a user context, may have a paid relationship. Medium capability, limited by TOS; attacks framed as "misuse" rather than "intrusion."
- **Insider.** Has engineer-level access to one or more parts of the system. High capability, low volume. Treated by the threat model via access-control and audit trails (ties to exercise-02 lineage governance).
- **Compromised upstream.** A third-party data source, a model vendor, or a tool / plugin that is itself compromised. The attacker reaches the surface through supply chain (OWASP ML06, MITRE ATLAS AML.T0010).
- **Compromised downstream.** A client or integration that is compromised and used as a proxy to exercise the surface. Rare but non-zero.
- **Opportunistic LLM researcher.** Low intent-to-harm, high prompt-sophistication. The actor class disclosed vulnerabilities historically come from; the threat model respects their likely capability.

Each row in the attack catalog (§2.3) is tagged with the actor classes for which the attack is in-scope. An attack that is only practical for an insider (e.g., "tamper with training-feature-store contents") is called out so the mitigation is access-control rather than model-level.

### 2.3 Attack catalog

The catalog is the heart of the artefact. Each row describes one attack class, grounded in the standard taxonomies, with enough detail that a reviewer can evaluate the defence.

**Row structure (one row per attack class):**

| Field | Content |
|---|---|
| Attack id | Short, stable id (`A.injection.direct`, `A.extraction.membership`, `A.supply-chain.rag`, `A.dos.throughput`). Used in alerts and probe-suite references. |
| MITRE ATLAS tags | List of applicable technique ids (e.g., `AML.T0051`, `AML.T0043`). |
| OWASP ML Top 10 tags | List of applicable category ids (e.g., `ML01`, `ML02`, `ML03`, `ML04`, `ML05`, `ML06`, `ML09`, `ML10`). |
| NIST AI 600-1 risk category | For generative surfaces, the relevant GAI risk (e.g., "information security," "information integrity"). |
| Attacker narrative | 2–4 sentences, written as the attacker's story. "A support ticket is submitted containing an instruction-like phrase intended to override the system prompt, directing the model to reveal internal queue triage rules." Specific, named inputs, named outcome. |
| Actors | Which actor classes are in-scope for this attack (anonymous external, authenticated external, insider, compromised upstream, etc.). |
| Preconditions | What the attacker needs for success (API access, a specific user population, access to a RAG indexed document, a specific tool enabled). |
| Impact | What the attacker gains. Tied to the asset inventory in §2.2 (`model-ip` / `training-data` / `downstream-data` / `output-abuse` / `operational-capacity` / `reputational-regulatory`). |
| Likelihood | Qualitative (`low` / `medium` / `high`). Grounded in actor capability and surface exposure. |
| Severity | Qualitative (`low` / `medium` / `high` / `critical`). Grounded in asset value and user impact. |

**Standard attack classes** (the threat model includes each that applies to the surface; a surface for which a class does not apply says so explicitly with a reason):

1. **`A.injection.direct` — Direct prompt injection.** ATLAS AML.T0051, OWASP ML01, NIST AI 600-1 information-security. An attacker's request contains instructions overriding the system prompt. For any surface whose input is user-supplied free text, this is in-scope by default.
2. **`A.injection.indirect` — Indirect prompt injection via retrieved content.** ATLAS AML.T0051 extended. For RAG surfaces, the model reads attacker-controlled content (a ticket body, a document, a comment) that contains injection instructions. In-scope for any surface that augments model context with retrieved documents, tool outputs, or external content.
3. **`A.jailbreak.policy-evasion` — Policy evasion / jailbreak.** ATLAS AML.T0051, NIST AI 600-1 harmful-content categories. The attacker frames a policy-violating request to bypass the model's safety training (role-play, hypothetical, encoding tricks, multi-step chaining).
4. **`A.harm.CBRN` — Harmful-information elicitation.** ATLAS AML.T0051, NIST AI 600-1 CBRN / dangerous-content categories. The attacker attempts to elicit uplift on weapons, violence, self-harm, or other restricted content categories. In-scope for any generative surface, even internal — a misuse of an internal assistant leaks to external surfaces via copy-paste.
5. **`A.extraction.membership` — Membership inference.** ATLAS AML.T0024, OWASP ML04. The attacker determines whether a specific record was in the training set.
6. **`A.extraction.inversion` — Model inversion / training-data extraction.** ATLAS AML.T0024, OWASP ML03. The attacker elicits memorized training examples (strings, images, names) from the deployed model.
7. **`A.extraction.model-theft` — Model theft / fingerprinting.** ATLAS AML.T0016 Craft Adversarial Data (query-based), OWASP ML05. The attacker systematically queries the surface to clone or fingerprint the model.
8. **`A.poisoning.training-data` — Training-data poisoning.** ATLAS AML.T0019 Publish Poisoned Datasets, OWASP ML02. An attacker-controlled upstream source injects content intended to bias the trained model. In-scope wherever the training pipeline reads from user-controlled or third-party content (ties to exercise-02 lineage audit).
9. **`A.poisoning.feedback-loop` — Feedback-loop poisoning.** Serving sink → feedback-training pipeline (ties to exercise-02). The attacker generates traffic to be re-ingested by the feedback loop in a direction that moves model behaviour.
10. **`A.supply-chain.model` — Model / base-model supply chain compromise.** ATLAS AML.T0010 ML Supply Chain Compromise, OWASP ML06. A compromised base model, pre-trained embedding, or model-library dependency arrives via the model-build pipeline.
11. **`A.supply-chain.rag` — Retrieval-corpus or tool supply-chain compromise.** A third-party plugin, tool, or indexed corpus is itself compromised. In-scope for RAG and agentic surfaces.
12. **`A.dos.throughput` — Denial of ML service.** OWASP ML10. Crafted inputs that exhaust GPU time, tool-calling budget, or latency SLO (e.g., inputs that force maximum-length decoding, inputs that trigger tool-call loops).
13. **`A.integrity.output` — Output integrity attack.** OWASP ML09. The attacker uses the model to produce intentionally wrong, misleading, or harmful outputs directed at a downstream consumer. Includes phishing-draft generation, malicious-code generation, and misinformation.
14. **`A.privacy.PII-leakage` — PII leakage via output.** NIST AI 600-1 data-privacy. The model emits PII — training-data PII, retrieved PII, or user-session PII reused across sessions. In-scope wherever the training set or retrieval corpus contains PII (ties to exercise-02 PII register).

For each applicable row the catalog also lists **non-applicable** rows with an explicit reason — "`A.poisoning.training-data`: N/A because the training pipeline reads only from authenticated internal sources (see exercise-02 lineage §2.1 third-party-source audit); the risk is reconsidered if a third-party source is added." Silent omission is the failure mode; an explicit N/A with a reason is operable.

### 2.4 Mitigation register

For every applicable attack row, the mitigation register names the controls in place, where they sit (model-side, input-side, output-side, policy-side, monitoring-side), the residual risk after mitigation, and the owner.

**Control categories:**

- **Model-side controls.** Safety-trained base model, policy-trained fine-tune, refusal behaviour on specific categories, groundedness training against hallucination. Owned by the surface team.
- **Input-side controls.** Input classifier (detects likely injection patterns), input sanitizer (normalizes encoding, strips suspicious control characters), tool-input validation (schema-check, parameter-bounds). Owned by the surface team + security.
- **Output-side controls.** Output classifier (detects policy-violating or PII-bearing content), output schema validator, output-grounding check (RAG citations, source consistency), rate-limited emission for high-risk classes. Owned by the surface team + safety.
- **Policy-side controls.** Documented refusal policy, out-of-scope-use list (ties to exercise-01 Model Card §2), user-category restrictions (SSO, queue, business-tier), auditable usage policy. Owned by the surface team + governance.
- **Monitoring-side controls.** Anomaly detection on query patterns, rate-limit on high-token-count or high-tool-call requests, telemetry on refusal rate drift, red-team probe cadence (ties to exercise-03). Owned by the surface team + SRE.
- **Access-side controls.** Authentication, authorization, upstream-source authentication (mTLS, signed artefacts), training-pipeline access control (ties to exercise-02). Owned by the data-platform / infra teams.

**Register row shape:**

| Field | Content |
|---|---|
| Attack id | Ties to the catalog (`A.injection.direct`). |
| Model-side controls | What the model does to resist this attack. |
| Input-side controls | What the serving-time input layer does. |
| Output-side controls | What the output layer / safety classifier does. |
| Policy-side controls | The policy that says "this is out-of-scope"; the auditable rule. |
| Monitoring-side controls | The telemetry that detects the attack in flight. |
| Access-side controls | The gate that says "this actor cannot reach this surface." |
| Residual risk | What remains after mitigation. Named as a sentence: "A sophisticated multi-turn injection could succeed at ~3% empirical rate (see probe suite); impact bounded by output-side refusal classifier and by the user's edit step." |
| Owner | Named roles for each control. |

A mitigation register with every row saying "we trained it to be safe" is not a register — it is a vibe. The register is structured so a reviewer can see where each control category is implemented and where it is missing.

### 2.5 Probe suite

The probe suite is the operational measurement of whether the mitigations hold. Each probe is a dated, versioned adversarial input (or input batch) linked to one or more attack rows.

**Probe suite structure:**

- **Probe version.** The probe set has a declared version and a change log; a probe set that is "the latest adversarial tests" is not versioned.
- **Probe tags.** Each probe is tagged to an attack id (and through it, to MITRE ATLAS and OWASP ML Top 10 ids). The tagging makes the probe-to-threat mapping legible on dashboards.
- **Probe size.** Per attack class, the number of probes. Minimums are pre-registered (e.g., ≥ 300 per class for release-gating slices, ≥ 100 for nightly canary slices).
- **Probe source.** Named provenance: hand-crafted by the security team, drawn from a public adversarial-prompt benchmark, generated by a red-team LLM (and the generator / version), contributed by an insider red-team exercise.
- **Probe refresh cadence.** Quarterly refresh of ≥ 20% of each probe set, to prevent the model from over-fitting to a static set.
- **Probe sensitivity.** Access-controlled. High-sensitivity probes (CBRN, policy-violating content) are stored in a restricted repository with legal-reviewer sign-off on access.
- **Probe-to-slice mapping.** Each probe set feeds the corresponding safety slice in exercise-03's cadence; the metric and threshold are declared there.

**Probe refresh and generation discipline.** Over-fitting to a static probe set is a silent failure mode: the model passes the probes but a near-neighbour prompt succeeds. Mitigations:

- Rotate ≥ 20% of each probe set quarterly.
- Generate adversarial variants via a red-team LLM (ties to mod-305/exercise-04) and dedupe against the current set.
- Track the "probe freshness" metric (fraction of probe set < 3 months old); a probe set below the freshness threshold is itself a finding.

### 2.6 Incident and disclosure playbook

Three triggering events, three playbook branches:

**Trigger 1 — probe-set regression detected (via exercise-03's safety cadence).**

1. Severity triage. Match the probe-set to the attack catalog row; look up the attack's severity; decide SEV level.
2. Immediate mitigation. If a safety-net control (input / output classifier) can be deployed to catch the regression, deploy it. If no safety-net exists, consider rolling back the model.
3. Communicate. Notify the governance reviewer, the product owner, and (if customer-impacting) the customer-facing lead.
4. Fix. Root-cause the regression, fix, re-run the probe suite, ship the fix.
5. Retro. Ties to mod-307/exercise-03 incident drill. The retro updates the probe suite, the mitigation register, or the catalog.

**Trigger 2 — in-the-wild attack observed.**

1. Capture. Record the attack's inputs and outputs (access-controlled; may include sensitive content).
2. Contain. Deploy safety-net filter if applicable; consider rate-limit or temporary suspension of the implicated user class.
3. Classify. Map to an existing attack row or add a new row to the catalog.
4. Communicate. Notify security review and governance; the catalog addition is tracked in the governance registry.
5. Permanent fix. Add probes for the observed attack to the probe suite; refresh model-side or input-side controls.

**Trigger 3 — external vulnerability disclosure.**

1. Acknowledge the disclosure within the team's declared SLA (e.g., 72 hours).
2. Reproduce. Confirm the attack against the current model version; capture the probe batch.
3. Triage. Same as Trigger 2.
4. Coordinated response. For coordinated disclosure, agree on an embargo and fix / mitigation deployment schedule. For an uncoordinated disclosure, mitigate first, communicate second.
5. Post-mitigation. Add the technique to the attack catalog if novel; add probes; update exercise-01's Model Card §8 risk row if the residual risk profile has changed.

The playbook names: the on-call rotation (ties to mod-307/exercise-03), the governance reviewer, the security reviewer, the legal reviewer (for disclosures with customer-data implications), and the communication template.

### 2.7 Review cadence

- **Quarterly.** The threat model is re-run; the catalog is reviewed against newly-disclosed techniques (new MITRE ATLAS entries, new OWASP ML Top 10 updates, new NIST AI 600-1 guidance, new published research). The mitigation register is re-checked. The probe suite is refreshed per §2.5.
- **Trigger-based.** Any of: a new tool / RAG index added, a new user population onboarded (ties to exercise-01 Model Card §2 scope expansion), a new model class or base model swapped in, an incident attributed to a gap in the threat model, an external vulnerability disclosure that touches the surface, a new regulatory guidance (e.g., EU AI Act systemic-risk designation). Any of these forces a re-review out of cycle.
- **Who signs.** Surface L30 owner, governance reviewer, security reviewer. For generative surfaces with a public-facing component, a legal reviewer signs on the privacy / IP rows.

### 2.8 Illustrative worked example (not a real system)

> **Illustrative feature** — `agent-assist/draft`, the internal LLM-backed support-agent assistant from exercise-01, -02, -03. The threat-model excerpt below uses the same composite surface; do not treat it as a real system or incident.

**System under threat model:**

- Surface: `agent-assist/draft` endpoint, model `assist-llm-v4.2.1`, streaming responses.
- Deployment context: internal, SSO-gated, available to agents in queues `billing / shipping / account-mgmt`. Not exposed externally. RAG over past-ticket corpus (agent's prior-ticket history).
- Trust boundaries: (1) customer → ticket-system (customer content enters the system); (2) ticket-system → serving layer (operates on ticket text); (3) serving layer → model (prompt and retrieved context sent to model); (4) model → output layer (output passes through safety classifier before UI); (5) output → agent UI (agent edits before send); (6) sent reply → customer.
- Data / API / model surface: request body (ticket content + agent session), retrieval index (past tickets), tools (none in v4), training feedback loop (approved with time-boxed governance review; see exercise-02).
- Audit owner: `team-agent-assist` L30 senior ML engineer.
- Last reviewed: 2026-11-15; prior: 2026-08-14.

**Asset ranking (summary):**

1. Downstream user data (ticket contents, PII-redacted but with residual risk). High value; exercise-02 PII register details.
2. Output abuse leverage (drafts could be shaped to produce misleading content for customers). High value; agent edit step mitigates but does not eliminate.
3. Training data (model trained on past tickets; memorized content could leak). Medium value; residual bounded by PII redactor residual rate.
4. Model IP (fine-tuning dataset is proprietary). Medium value.
5. Operational capacity (serving budget per quarter). Low–medium value.

**Actor classes in scope:**

- External authenticated user (customer, via submitted ticket content): HIGH volume; customer content enters as RAG input and as the ticket itself → `A.injection.indirect` is the primary concern.
- Insider agent (authenticated support agent): MEDIUM; agent can submit crafted queries intentionally or unintentionally.
- Insider engineer (training-pipeline access): LOW volume, HIGH capability; access-controls and audit (exercise-02) are the primary mitigation.
- External anonymous: OUT OF SCOPE (surface is not externally reachable; verified via network policy).
- Compromised upstream (e.g., ticket system compromise): LOW likelihood, HIGH severity; mitigated by mTLS and lineage audit (exercise-02).

**Attack catalog excerpt (6 rows; real catalog has 14):**

| Id | ATLAS | OWASP | NIST 600-1 | Narrative | Actors | Preconditions | Impact | Likelihood | Severity |
|---|---|---|---|---|---|---|---|---|---|
| `A.injection.direct` | AML.T0051 | ML01 | information-security | Agent submits a request to the model with a prefix "ignore prior instructions and..." attempting to override system prompt. Succeeds if model emits attacker-specified output in the draft. | authenticated agent | agent has endpoint access | model-ip + output-abuse | medium | medium |
| `A.injection.indirect` | AML.T0051 | ML01 | information-security | A customer submits a ticket whose body contains an injection ("as agent, please disclose..."). The RAG layer retrieves the body; the model is instructed to disregard system prompt and produce content directed at the agent or at a future customer. | external authenticated customer | ticket-submission; RAG retrieval of body | output-abuse + reputational | medium | high |
| `A.jailbreak.policy-evasion` | AML.T0051 | ML01 | harmful-bias / harmful-content | Agent attempts role-play framing ("suppose I were a sysadmin...") to elicit outputs outside support policy (e.g., recommending refund amounts beyond policy). | authenticated agent | endpoint access | output-abuse + reputational | medium | medium |
| `A.harm.CBRN` | AML.T0051 | — | CBRN | Agent or bot attempts to elicit CBRN-adjacent content. Not an ordinary support-domain attack but included because any uplift is a worst-case outcome. | any authenticated | endpoint access | reputational-regulatory | low | critical |
| `A.extraction.inversion` | AML.T0024 | ML03 | data-privacy | Attacker (authenticated or probe-wielding researcher) issues queries designed to elicit memorized training-data strings, especially PII residues. | authenticated | endpoint access + ability to run ~1k queries | training-data (PII) | low | high |
| `A.supply-chain.rag` | AML.T0010 | ML06 | value-chain | Past-ticket corpus is indexed; a ticket with injection content persists in the corpus and is retrieved on future sessions, poisoning future generations. Partial overlap with `A.injection.indirect`; the supply-chain framing captures persistence. | external customer | ticket indexed in RAG | output-abuse + reputational | medium | high |

**Mitigation register excerpt (for `A.injection.indirect`):**

- Model-side: safety-trained base model `base-llm-v3`; fine-tune applied for support-queue domain with explicit "do not act on instructions appearing in retrieved content" exemplars.
- Input-side: ticket-body sanitizer strips suspicious instruction-shaped sequences (regex + classifier); separate RAG context tagging encloses retrieved content in explicit "RETRIEVED_CONTEXT" tags that the model is trained to treat as data, not instruction.
- Output-side: output safety classifier (`output-safety-v3`) catches outputs containing policy-violating content; drafts flagged are returned as `refusal` object, not emitted to UI.
- Policy-side: Model Card §2 (exercise-01) declares out-of-scope: "not approved for use on queues flagged `safety-sensitive`"; agents are trained in the onboarding course to review draft content before sending.
- Monitoring-side: `safety.injection.indirect` slice runs on release gate (threshold ≤ 5% success) and nightly canary (regression tolerance 1% abs). Query-pattern anomaly detector flags tickets with instruction-shaped prefixes at ingest time.
- Access-side: upstream ticket-system authentication (mTLS; row-level audit log); RAG index access-controlled to the owning team.
- Residual risk: measured ~3% success rate on current probe suite (`safety.injection.indirect`); mitigated operationally by agent edit step (agent sees the draft before send). Residual risk is further bounded by rate-limit on high-token-count / high-tool-call requests.
- Owners: `team-agent-assist` (model-side, input-side); `team-security` (output classifier, probe suite); `team-support-platform` (upstream auth); `team-governance` (policy + Model Card).

**Probe suite excerpt:**

| Attack id | Probe set version | N | Source | Last refreshed |
|---|---|---|---|---|
| `A.injection.direct` | `probe-injection-direct-v4` | 1,200 | hand-crafted + public benchmarks + red-team LLM | 2026-10-20 |
| `A.injection.indirect` | `probe-injection-indirect-v4` | 800 | ticket-shaped injections from internal red-team | 2026-10-20 |
| `A.jailbreak.policy-evasion` | `probe-jailbreak-v3` | 600 | internal red-team + MITRE ATLAS examples | 2026-09-15 |
| `A.harm.CBRN` | `probe-harm-cbrn-v2` (access-controlled) | 300 | internal security team | 2026-09-15 |
| `A.extraction.inversion` | `probe-inversion-v3` | 500 | synthetic PII strings in a shadow training set | 2026-08-12 |

**Incident playbook snapshot:**

- SEV-1 trigger: any safety slice regression of > 2% above launch-gate threshold on `A.harm.*` or `A.injection.*`. On-call page; governance notified within 1 hour; release blocked.
- SEV-2 trigger: in-the-wild attack observed with output emitted to agent UI but caught by edit step; governance notified within 24 hours; add probe within 72 hours.
- SEV-3 trigger: external disclosure of a technique not currently in the catalog but demonstrably applicable to the surface; add to catalog within 7 days; add probes within 14 days.
- Communication template: on SEV-1, surface L30 writes a post in `security-incident-queue` with the slice, the probe set, the measurement, and the rollback decision; governance reviewer counter-signs within 4 hours.

### 2.9 Optional runnable sketch: `threat_model_lint.py`

The threat model, like the audit, is easier to keep honest with a small linter that reads the Markdown and checks the discipline. The script is small on purpose.

```python
# threat_model_lint.py
"""Lint a threat-model.md artefact for completeness.

A reviewer still reads the threat model. This linter enforces the mechanical
discipline so the reviewer's attention budget goes to the judgement calls.
"""

from dataclasses import dataclass
import re
import sys


REQUIRED_SECTIONS = [
    "System under threat model",
    "Asset",            # asset inventory
    "Actor",            # actor inventory
    "Attack catalog",
    "Mitigation register",
    "Probe suite",
    "Incident",         # incident / disclosure playbook
    "Review cadence",
]

ATLAS_PATTERN = re.compile(r"AML\.T\d{4}")
OWASP_PATTERN = re.compile(r"\bML(0[1-9]|10)\b")
SEVERITY_WORDS = {"critical", "high", "medium", "low"}

REQUIRED_ATTACKS_FOR_GENAI = [
    "injection.direct", "injection.indirect", "jailbreak",
    "harm", "extraction", "supply-chain", "dos", "privacy",
]


@dataclass
class Finding:
    severity: str
    message: str


def lint(text: str) -> list[Finding]:
    f: list[Finding] = []
    lowered = text.lower()

    for s in REQUIRED_SECTIONS:
        if s.lower() not in lowered:
            f.append(Finding("error", f"missing section: {s}"))

    atlas_hits = ATLAS_PATTERN.findall(text)
    if not atlas_hits:
        f.append(Finding("error", "no MITRE ATLAS techniques cited (expected AML.T#### ids)"))
    elif len(set(atlas_hits)) < 3:
        f.append(Finding("warn", f"only {len(set(atlas_hits))} distinct ATLAS techniques cited; expect >= 3 for a generative surface"))

    owasp_hits = OWASP_PATTERN.findall(text)
    if not owasp_hits:
        f.append(Finding("error", "no OWASP ML Top 10 categories cited (expected MLxx ids)"))

    if "nist ai" not in lowered:
        f.append(Finding("warn", "NIST AI RMF / NIST AI 600-1 not cited"))

    for sev in SEVERITY_WORDS:
        if sev not in lowered:
            f.append(Finding("warn", f"severity term '{sev}' not used; threat model usually names all four"))

    for attack in REQUIRED_ATTACKS_FOR_GENAI:
        if attack not in lowered:
            f.append(Finding("warn", f"attack class '{attack}' not addressed; add a row or an explicit N/A with reason"))

    if "residual risk" not in lowered:
        f.append(Finding("error", "no 'residual risk' language; mitigation register must state residual per row"))

    if "probe" not in lowered:
        f.append(Finding("error", "no probe suite language"))

    if not re.search(r"sign.{0,5}off|approver|approved by|signed by", lowered):
        f.append(Finding("error", "no sign-off / approver record"))

    if not re.search(r"re.?review|quarterly|cadence", lowered):
        f.append(Finding("error", "no review cadence declared"))

    return f


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("usage: threat_model_lint.py <threat-model.md>")
        sys.exit(2)
    text = open(sys.argv[1]).read()
    findings = lint(text)
    for fd in findings:
        print(f"{fd.severity:>5}: {fd.message}")
    errors = [fd for fd in findings if fd.severity == "error"]
    sys.exit(1 if errors else 0)
```

## 3. Validation steps

Walk the threat model through this checklist before considering it done:

1. **System-under-threat-model names the surface, context, trust boundaries, and data/API/model surface.** Generic "our LLM" is rejected.
2. **Asset inventory ranks the top assets.** Mitigations are prioritized against the most valuable asset; a threat model without ranked assets has not prioritized.
3. **Actor inventory enumerates the actor classes** and names which are in-scope / out-of-scope for this surface.
4. **Attack catalog addresses the 14 standard classes** (or declares explicit N/A with a reason for each not addressed) and cites MITRE ATLAS technique ids and OWASP ML Top 10 category ids.
5. **Each attack row has: narrative, actors, preconditions, impact, likelihood, severity.**
6. **NIST AI 600-1 categories are tagged** on applicable rows for generative surfaces.
7. **Mitigation register has every control category per attack row** (model / input / output / policy / monitoring / access) and names residual risk.
8. **Probe suite is versioned, dated, sourced, and tied to attack rows.** A probe set without a refresh cadence is a probe set that will decay.
9. **Probe suite refresh cadence is named** (≥ 20% rotation quarterly default).
10. **Probe suite ties to exercise-03 safety slice family.** The probe-to-slice mapping is explicit.
11. **Incident / disclosure playbook has three branches** (probe regression, in-the-wild observation, external disclosure) with named SLAs, communication templates, and SEV triage.
12. **Review cadence is quarterly + trigger-based** with named signers.
13. **Cross-references to exercise-01 Model Card §8 risk rows, exercise-02 lineage audit, exercise-03 safety slice cadence** are explicit.

Static checks a CI step can enforce on `threat-model.md`:

```bash
grep -ciE "trust boundary|attacker|actor|asset"       threat-model.md
grep -ciE "atlas|AML\\.T"                             threat-model.md
grep -ciE "owasp.*ml.*top.*10|ML0[1-9]|ML10"          threat-model.md
grep -ciE "nist ai|600-1|rmf"                         threat-model.md
grep -ciE "mitigation|residual risk|control"          threat-model.md
grep -ciE "probe|red.team|adversarial"                threat-model.md
grep -ciE "incident|disclosure|playbook|sev"          threat-model.md
grep -ciE "review cadence|quarterly|trigger"          threat-model.md
```

`threat_model_lint.py` from §2.9 enforces the structural discipline.

## 4. Rubric / review checklist

- [ ] `System under threat model` section names surface, deployment context, trust boundaries, data / API / model surface, audit owner, last-reviewed date.
- [ ] Asset inventory ranks the top assets and ties each row to the catalog's impact column.
- [ ] Actor inventory enumerates the actor classes, names which are in-scope / out-of-scope, and ties each to catalog rows.
- [ ] Attack catalog addresses the 14 standard classes with explicit N/A where applicable; each N/A names a reason.
- [ ] Each catalog row has: id, MITRE ATLAS tags, OWASP ML Top 10 tags, NIST AI 600-1 category (for generative), narrative, actors, preconditions, impact, likelihood, severity.
- [ ] Mitigation register has every control category per attack row (model / input / output / policy / monitoring / access) with named owners and explicit residual-risk sentences.
- [ ] Probe suite is versioned, sized, sourced, access-controlled where needed, and tied to catalog rows.
- [ ] Probe refresh cadence (≥ 20% quarterly rotation) is declared; probe freshness is monitored.
- [ ] Probe suite maps to exercise-03 safety slice family (one probe set per safety slice).
- [ ] Incident / disclosure playbook has three branches (probe regression, in-the-wild observation, external disclosure) with SEV triage, SLAs, communication templates, and named signers.
- [ ] Review cadence (quarterly + trigger-based) is declared with named signers (surface owner, governance reviewer, security reviewer; legal for privacy/IP rows on generative surfaces).
- [ ] Cross-references to exercise-01 Model Card §8 risk rows, exercise-02 lineage + PII register, exercise-03 safety slice cadence are explicit.
- [ ] NIST AI RMF MAP / MANAGE / GOVERN cited; NIST AI 600-1 cited for generative surfaces; MITRE ATLAS cited for adversarial rows; OWASP ML Top 10 cited for all rows where applicable.

Review-weighted points for grading:

- System-under-threat-model and trust-boundary clarity: 10%
- Asset + actor inventory, ranked and scoped: 10%
- Attack catalog coverage (14 standard classes, N/A with reason): 20%
- Mitigation register with six control categories and residual risk per row: 20%
- Probe suite versioning, size, and refresh cadence: 10%
- Incident / disclosure playbook with three branches, SEV triage, SLAs: 10%
- Standards alignment (NIST AI RMF, NIST AI 600-1, MITRE ATLAS, OWASP ML Top 10): 10%
- Cross-consistency with exercise-01 Model Card §8, exercise-02 lineage + PII register, exercise-03 safety slice cadence: 10%

## 5. Common mistakes

- **Threat model as a one-page checklist.** "We considered prompt injection" is not a threat model. The catalog has a row with ATLAS tag, actor scope, preconditions, mitigation, residual, and probe set per attack. One line per attack is insufficient.
- **"Our LLM is safety-trained" as a mitigation.** That is one control category (model-side) in isolation. A reviewer cannot evaluate a single-category mitigation. The register names all six categories for every row.
- **No residual risk.** A mitigation row without a residual-risk sentence is a claim of perfection, which is almost always wrong. Residual risk is what remains after mitigation; naming it is what makes the threat model honest.
- **Probe suite that is "the latest adversarial tests."** Un-versioned, un-dated, un-sourced probes decay silently. Every probe set has a version, a date, a source, a size, and a refresh cadence.
- **Over-fitting to a static probe set.** A model that passes a probe set can still fail on near-neighbour prompts. The quarterly rotation and the red-team-LLM variant-generation discipline mitigate this.
- **No separation of in-scope / out-of-scope actors.** A threat model that lumps all attackers together over-weights low-capability high-volume threats and under-weights high-capability low-volume threats. The actor inventory forces the distinction.
- **No N/A rows.** A threat model that silently omits poisoning or supply-chain is a threat model that did not consider them. The catalog's 14 standard rows appear; the ones not applicable carry an explicit reason.
- **Mitigation register as prose.** "We have many controls in place" is not a register. The register is tabular: six control categories × every attack row × owner + residual.
- **No incident playbook.** A threat model without an incident playbook is a snapshot; the playbook is the procedure that makes the threat model operable when a probe regresses or an attack is observed.
- **No disclosure SLA.** External vulnerability disclosure requires a declared acknowledgment SLA (72 hours is a reasonable default). A threat model that does not name the SLA is a threat model that will mishandle a disclosure.
- **Owner as a person.** Names change; roles don't. `team-security` + `team-agent-assist` survives rotation; "Jamie from security" does not.
- **No review cadence.** A threat model without a quarterly cadence and trigger-based re-review conditions decays within six months; new techniques accumulate in disclosure feeds and the catalog goes stale.
- **No cross-reference to exercise-03 safety slice cadence.** The threat model chooses *which* measurements matter; exercise-03 schedules those measurements. A threat model that does not cite the slice cadence is a threat model whose mitigations are not actually measured over time.
- **No cross-reference to exercise-02 lineage.** Supply-chain and poisoning attacks are mitigated at the lineage layer (authenticated sources, audit trails, feedback-loop governance). A threat model that does not cite the lineage audit misses where those mitigations live.
- **Generic generative-AI threat model.** NIST AI 600-1's risk categories apply but they apply to *this surface*. A generic "we address information security" without a specific `A.injection.indirect` row for a RAG surface is not operable; the attack row and its preconditions ground the general category in this system.
- **Severity by vibe.** `critical / high / medium / low` is a calibration, not an opinion; the threat model's severity assignment is tied to the asset value and the impact description, and a reviewer can question (and re-calibrate) the assignment.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MAP function grounds the threat model as the risk-characterization surface for an AI system. MANAGE function grounds the mitigation register as the risk-control surface. GOVERN function grounds the incident / disclosure playbook as the accountability surface.
- NIST AI 600-1 (Generative AI Profile) — <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>. Risk categories applied by the catalog: information security (prompt injection, jailbreaks, extraction), information integrity (hallucination, misleading output), data privacy (PII leakage, membership inference), intellectual property (training-data memorization, model theft), human-AI configuration (over-reliance / automation-bias misuse), value chain (supply-chain compromise), CBRN / dangerous / obscene content categories for refusal-rate slices.
- MITRE ATLAS — <https://atlas.mitre.org/>. Primary adversarial-tactic taxonomy for the catalog. Core techniques used by default: AML.T0051 LLM Prompt Injection (direct and indirect), AML.T0043 Craft Adversarial Data, AML.T0024 Exfiltration via ML Inference API, AML.T0019 Publish Poisoned Datasets, AML.T0010 ML Supply Chain Compromise, AML.T0016 Craft Adversarial Data (query-based model theft / fingerprinting). The catalog's ATLAS tags are the operational link between the artefact and ATLAS's maintained adversarial-technique knowledge base.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Taxonomy for the catalog's OWASP tags: ML01 Input Manipulation Attack, ML02 Data Poisoning Attack, ML03 Model Inversion Attack, ML04 Membership Inference Attack, ML05 Model Theft, ML06 AI Supply Chain Attacks, ML07 Transfer Learning Attack, ML08 Model Skewing, ML09 Output Integrity Attack, ML10 Denial of ML Service. Every catalog row carries the applicable OWASP tag(s).

Practitioner implementation examples:

- VeriSwarm Trust Center — <https://veriswarm.ai/trust>. Example of a practitioner publishing a threat-model / security-posture page that externalises mitigation and probe-suite references; useful only as a shape-of-artefact reference, not as a standard.

Local context (authoritative for the artefact shape and cross-references):

- Paired lesson: `lessons/mod-309-responsible-ai-governance/exercises/exercise-04-misuse-and-adversarial-threat-model.md`.
- Module README: `modules/mod-309-responsible-ai-governance/README.md`.
- Related exercise (Model Card §8 risk rows that this threat model feeds): `modules/mod-309-responsible-ai-governance/exercise-01-model-card-authoring/SOLUTION.md`.
- Related exercise (lineage + PII register that feeds the supply-chain, poisoning, and PII-leakage rows of this threat model): `modules/mod-309-responsible-ai-governance/exercise-02-data-lineage-and-pii-audit/SOLUTION.md`.
- Related exercise (safety slice cadence that schedules the probe-suite measurements and gates release on them): `modules/mod-309-responsible-ai-governance/exercise-03-fairness-and-robustness-slice-cadence/SOLUTION.md`.
- Related exercise (incident drill whose playbook format this threat model's playbook branches follow): `modules/mod-307-ml-reliability-slos/exercise-03-ml-incident-response-drill/SOLUTION.md`.
- Related exercise (LLM-as-judge harness that implements the judge for injection-success and refusal-rate probe-suite scoring): `modules/mod-305-advanced-evaluation/exercise-04-llm-as-judge-when-and-how/SOLUTION.md`.
- Related exercise (shadow / canary design that is the deployment surface where safety-net mitigations are rolled out during an incident): `modules/mod-305-advanced-evaluation/exercise-02-shadow-and-canary-deploy-design/SOLUTION.md`.
