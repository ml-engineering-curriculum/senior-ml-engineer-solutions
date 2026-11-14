# exercise-01 — Prompt vs RAG vs Fine-Tune Decision (Reference Solution)

Paired exercise: `lessons/mod-304-production-llm-integration/exercises/exercise-01-prompt-vs-rag-vs-finetune-decision.md`.

## 1. Solution overview

This exercise produces `llm-regime-decision.md` — a written decision that says, for one proposed LLM-backed feature, whether the team will go with **prompt-only**, **retrieval-augmented generation (RAG)**, **fine-tuning**, or an **explicit hybrid**, and why. The artefact has four parts:

1. A **feature frame** that names the decision the LLM is in service of — the task, the user, the input distribution, the output contract, and the cost of being wrong in each direction.
2. A **regime catalogue** that enumerates prompt-only, RAG, parameter-efficient fine-tuning (PEFT / LoRA), full fine-tuning, continued pre-training, and hybrid compositions — and what each is good at and bad at for *this* feature.
3. A **decision rubric** — the axes you score the feature on, the collapse rules that pick a regime, and the tie-break ordering.
4. A **worked application** of the rubric to the team's actual candidate feature, complete with rule trace, the picked regime, the explicit rollback story, and the measurable success criteria that define "this was the right pick."

The reference solution below shows the structure, works a fully-labeled illustrative example through it, and gives the grading rubric that calibrates every learner's decision document to the same bar.

The exercise exists because the three regimes have **very different supply-chain surfaces, operational postures, and failure modes** — and most teams pick between them by vibe or by what the loudest engineer built last. This artefact makes the decision explicit *before* a model ships, and makes it reviewable.

## 2. Worked answer / implementation

### 2.1 Feature frame

Before scoring axes, write the frame. Six lines are enough:

- **Task statement.** One sentence, in the form "given X, produce Y under constraint Z." If the task cannot be stated this way, it is not a task yet — it is a wish.
- **User and surface.** Who sees the output? A user in a chat window, a downstream service, an analyst in a dashboard, an automated workflow with no human in the loop? Hallucination tolerance varies by three orders of magnitude between these.
- **Input distribution.** What inputs can arrive? Natural language, forms, code, mixed-modality? Adversarial, benign, or both?
- **Output contract.** Free text, structured JSON validated against a schema, a scalar score, a classification label with a rationale? A regime that cannot reliably produce the output contract is disqualified.
- **Failure cost (per direction).** What does a wrong-and-confident answer cost? What does a refusal cost? These are rarely symmetric. The OWASP ML Top 10 and NIST AI RMF both put "harm per direction" on the designer, not the vendor.
- **Latency and cost envelope.** The p95 end-to-end latency and the per-request cost ceiling the feature must live within.

### 2.2 Regime catalogue (for the LLM regime set)

For each regime a credible candidate, capture:

- **One-sentence description.**
- **Minimum viable team shape and tooling.**
- **What data it requires (and in what form).**
- **Typical serving cost shape.**
- **Where its ground truth lives and how it goes stale.**
- **Threat surface** (MITRE ATLAS tactics it is most exposed to; OWASP ML top-10 items it inherits).

A starter catalogue (adjust to your stack):

| Regime | One-line description | Required data | Serving cost | Primary failure mode | Top threat lens |
|---|---|---|---|---|---|
| **Prompt-only (zero/few-shot)** | Hosted or local LLM with a frozen instruction prompt; no retrieval, no fine-tuning. | A prompt and a golden eval set. | Per-call (tokens in + out). | Hallucination on anything not in the model's training data; distribution drifts silently when the vendor ships a new model version. | OWASP ML — prompt injection; MITRE ATLAS — supply chain / vendor model swap. |
| **RAG (retrieval-augmented generation)** | Retrieve top-k documents from your corpus, stuff them into the prompt, let the LLM compose an answer. | A corpus, an embedder, a vector store, a prompt that cites. | Per-call + retrieval index + re-embedding cost on corpus updates. | Retrieval quality **is** the product; a wrong top-k produces a confidently-wrong answer with citations. | OWASP ML — data poisoning (of the corpus); MITRE ATLAS — context manipulation via retrieved content. |
| **PEFT / LoRA fine-tune** | Small adapter weights trained on task-specific labeled data on top of a frozen base model. | Hundreds-to-tens-of-thousands of task-labeled examples. | Base-model serving + adapter swap (cheap). | Over-fits narrow task; loses robustness on inputs outside the training distribution. | MITRE ATLAS — training-data poisoning; OWASP ML — insufficient training data governance. |
| **Full fine-tune / instruction-tune** | Full weights updated on task + instruction data. | Large labeled dataset; GPU budget; eval plan that does not just use the training distribution. | Dedicated serving; large storage; costly rollout. | Catastrophic forgetting of general capabilities; licence/lineage obligations inherited from the base checkpoint. | NIST AI RMF — MAP/MEASURE for model-lineage risk; MITRE ATLAS — supply-chain and model-reuse. |
| **Continued pre-training** | More unsupervised training on domain text before any instruction-tuning. | Very large in-domain corpus; serious infra. | Highest. | Large time-to-value; hard to prove the pre-training lift beat the cheaper regimes. | NIST AI RMF — GOVERN around training data provenance. |
| **Hybrid (RAG + PEFT, RAG + output validator, etc.)** | Compose two of the above behind one interface. | Varies. | Mixed. | Operational complexity: more moving parts, each with its own monitoring. | All of the above compose. |

### 2.3 Decision rubric — axes

Score the **feature**, not the regime, on seven axes. Each axis is scored 1–5 (1 = cheap/easy, 5 = expensive/hard):

1. **Knowledge location (K).** Where does the knowledge the answer requires live?
   - 1 = entirely inside the base LLM's training data (general-purpose task).
   - 5 = entirely in private documents the LLM has never seen.
2. **Knowledge freshness (F).** How fresh must the knowledge be?
   - 1 = stable over years.
   - 5 = minutes-to-hours (ticket status, inventory, outage state).
3. **Output structure strictness (O).** How strict is the output contract?
   - 1 = free text, human-reviewed.
   - 5 = schema-validated JSON consumed by another service with no human in the loop.
4. **Task-label availability (L).** How many task-labeled examples do we have?
   - 1 = none (and none are cheap to get).
   - 5 = tens of thousands with gold labels.
5. **Latency budget (T).** p95 end-to-end latency the product can absorb.
   - 1 = minutes acceptable (async).
   - 5 = <500 ms synchronous.
6. **Per-call cost ceiling (C).** Finance-set cost per request.
   - 1 = unconstrained.
   - 5 = sub-cent per call.
7. **Failure-mode severity (S).** Blast radius of a wrong-but-confident answer.
   - 1 = annoyance.
   - 5 = financial, safety, or legal harm. This is the NIST AI RMF "harm" lens.

The axes are intentionally not independent — they cross-constrain the regime set.

### 2.4 Collapse rules (how a score vector picks a regime)

Rules fire in order; the first rule whose conditions are met wins. If none fires, fall through to the tie-break ladder in §2.5.

```
Rule R1 (private + fresh knowledge):     K >= 4 and F >= 3            → RAG (prompt-only disqualified).
                                                                        Fine-tune disqualified unless corpus is static.
Rule R2 (strict output contract):        O = 5                        → prompt-only or RAG with a validator layer;
                                                                        schema validation + repair loop is mandatory.
Rule R3 (label-rich + narrow task):      L >= 4 and task is narrow    → PEFT / LoRA viable; evaluate vs. RAG baseline.
Rule R4 (label-starved):                 L <= 2                       → prompt-only or RAG first; fine-tune blocked
                                                                        until labeled data exists.
Rule R5 (tight latency + tight cost):    T >= 4 and C >= 4            → prompt-only with smallest viable model,
                                                                        or a classical model entirely (escalate to
                                                                        the hybrid lesson, exercise-03).
Rule R6 (fresh + structured):            F >= 4 and O >= 4            → RAG with retrieval over a live store;
                                                                        pure fine-tune disqualified (stale weights).
Rule R7 (high harm):                     S >= 4                       → mandatory output validator, mandatory
                                                                        human-review or guardrail gate, and a fallback
                                                                        regime that the feature can degrade to.
                                                                        This layers on top of the regime pick.
Rule R8 (default):                                                     → prompt-only with a golden eval set and a
                                                                        rollback path to "do nothing / previous flow."
```

Senior-engineer notes on the rules:

- Rule R1 is the most common one teams get wrong. "The LLM seems to know things about our product" is not evidence; it is statistical coincidence over a small golden set. If the knowledge lives in private docs, you need RAG (or fine-tuning on those docs, with the lineage and licence work that implies), not a longer prompt.
- Rule R7 **never picks a regime by itself**. It stacks on top of the pick. A high-harm feature can still be prompt-only — it just cannot be prompt-only *without* a validator and a rollback path. This matches the NIST AI RMF MANAGE function: risk treatment is additive to the design, not a substitute for it.
- Rule R2 implies more than "ask the model for JSON." Schema validation, a repair loop that re-prompts on invalid output, and a hard cap on repair attempts are all mandatory; without them, the "structured output" is aspirational.
- Fine-tuning is the regime most frequently reached for and least frequently *needed*. The practitioner default should be "start with prompt, upgrade to RAG when R1 fires, fine-tune only when R3 fires with measurable gap over the RAG baseline on a held-out eval."

### 2.5 Tie-break ordering

When two regimes pass all rules, prefer in this order:

1. The regime the team has **already deployed and monitored** in production. Familiarity is an evaluation signal — the known failure modes beat the hypothetical ones.
2. The regime with the **cheapest rollback path**. Prompt-only rolls back in one deploy. RAG rolls back in a prompt + index flip. A fine-tuned model behind a vector index rolls back in neither.
3. The regime with the **smallest supply-chain surface**. A hosted LLM is a vendor dependency that can be re-versioned under you — MITRE ATLAS calls this out as a supply-chain tactic. A self-hosted open-weight model is a different (and often larger) surface. Prefer the one your governance process has actually ingested.
4. The regime that **produces the most testable artefact**. A golden eval set + prompt string with versioning beats a LoRA adapter whose training data provenance has to be reconstructed on audit.
5. The regime with the **lowest "silent drift" exposure**. Hosted-vendor regimes drift whenever the vendor ships a new model version even if you pin a name alias; self-hosted regimes drift when you re-pull the base weights. Whichever your team monitors better wins.

### 2.6 Illustrative worked application (not a real feature)

> **Illustrative feature** — "support ticket draft reply." An internal support tool wants to pre-fill a draft reply for the human agent to review and send on each inbound ticket. This example is a composite for calibration; do not treat it as a real employer or incident.

Feature frame:

- Task: given a ticket body (English, 50–2000 words) and the account's product tier, produce a draft reply of 50–200 words that cites the two most relevant internal KB articles.
- User and surface: a human support agent in an internal tool. The human always reads and edits before sending.
- Input distribution: English natural language; mostly cooperative users; adversarial content is rare but possible (prompt-injection attempts, leaked tokens).
- Output contract: JSON with `draft_text`, `kb_cites[]` (each a KB article id and a URL), and `needs_human_escalation` (bool). Schema-validated.
- Failure cost: a wrong-but-confident draft that an agent sends without reading is embarrassing but not catastrophic because the human is in the loop by policy. A refusal is cheap — the agent writes the reply manually.
- Latency / cost: p95 ≤ 4 s; ≤ $0.03 per draft.

Axis scores:

| Axis | Score | Reason |
|---|---|---|
| K (knowledge location) | 4 | Reply quality depends on the internal KB and on account-specific context the LLM has never seen. |
| F (freshness) | 3 | KB articles change weekly; account state changes continuously but the drafter can accept stale-by-hours. |
| O (output structure) | 4 | Schema-validated JSON; downstream UI parses it. Human in the loop softens this slightly — a validator can repair, and the agent can edit. |
| L (labels) | 2 | Years of sent-reply pairs exist, but they are noisy (edited and unedited mixed, agent-style varies). Not immediately training-grade. |
| T (latency) | 3 | 4 s is loose for a draft. |
| C (cost) | 3 | $0.03/draft is generous for a single-shot call, tight for an agent loop. |
| S (failure severity) | 2 | Human in the loop; blast radius bounded. |

Rule trace:

1. R1? K = 4 and F = 3 → fires. RAG is indicated; prompt-only is disqualified (the KB knowledge is not in the base model); fine-tuning is **not** disqualified by R1, but F = 3 means a fine-tuned snapshot will drift against the live KB quickly.
2. R2? O = 4 (not 5) → does not fire as a *hard* schema requirement, but a validator with a repair loop is still cheap insurance.
3. R3? L = 2 → does not fire.
4. R4? L = 2 → fires. Prompt-only or RAG first; fine-tune blocked until the historical ticket data is cleaned and labeled.
5. R5? T = 3, C = 3 → does not fire.
6. R6? F = 3 (not >= 4) → does not fire.
7. R7? S = 2 → does not fire. A human-review gate is built in by product policy anyway.
8. R8 (default) → RAG wins; R1 and R4 both point to it.

Tie-break was not needed (R1 and R4 agreed), but the ladder is still worth walking:

- The team already runs an embedding + vector store for a different feature (search). Reuse is cheap. Favours RAG.
- Rollback for RAG is "turn off the drafter; agents write manually." One feature flag.
- Supply-chain surface: hosted LLM API already approved by the governance process for an unrelated feature. No new vendor onboarding.

Final recommendation (worked):

> **Regime:** RAG with hosted LLM for generation, internal KB chunked and re-embedded nightly, top-k = 5 with reranker, schema-validated JSON output with a one-attempt repair loop on invalid output, and a hard refusal path that returns `needs_human_escalation = true` if retrieval returns no document above a similarity floor. Fine-tuning is **not** on the near-term roadmap; revisit after six months of logged edits generate a cleaner labeled dataset.
>
> **Rollback story:** one feature flag disables the drafter; the UI falls back to a blank editor.
>
> **Success criteria (measured on a frozen golden set of 100 tickets at launch, re-measured monthly):**
> - ≥ 70% of drafts accepted by the agent with ≤ 20% edited characters;
> - ≤ 1% of drafts contain a factual claim that is not supported by a cited KB article (manual audit);
> - p95 latency ≤ 4 s and mean cost ≤ $0.03/draft;
> - `needs_human_escalation = true` on 100% of tickets where retrieval returned zero relevant docs.
>
> **What would change the pick:** if the labeled, cleaned ticket-reply dataset grows past ~5,000 clean pairs *and* RAG hits a quality ceiling on a measured subset, revisit with a PEFT adapter trained on the cleaned pairs *on top of* RAG. Never replace RAG with a fine-tune — the KB keeps shifting.

The write-up shows the rule trace **and** the success criteria **and** the "what would change the pick" paragraph. All three are mandatory. A recommendation without a "what would change the pick" paragraph is not a decision; it is a plan that cannot admit error.

### 2.7 What a full artefact contains

The learner's `llm-regime-decision.md` should contain, in order:

1. The **feature frame** (§2.1), six lines for a real candidate feature.
2. The **regime catalogue** (§2.2), filled in for the regimes credible on the team's stack.
3. The **axis definitions** (§2.3), with any axis anchors the team has sharpened for their domain.
4. The **collapse rules** (§2.4), tuned if the team's regulatory posture or policy changes a threshold.
5. The **tie-break ladder** (§2.5).
6. At least **one worked application** for the team's candidate feature with scored axes, full rule trace, tie-break reasoning, and a success-criteria section.
7. A **threat posture appendix** that names, per picked regime, the specific MITRE ATLAS tactics and OWASP ML Top 10 items the regime inherits, and what the design does about each. At minimum: prompt injection, insecure output handling, training-data poisoning (RAG corpus or fine-tune set), model-supply-chain compromise.
8. A **"what would change the pick"** paragraph with explicit, measurable triggers.
9. A **"last reviewed"** dated line.

## 3. Validation steps

Walk through this checklist on the artefact:

1. **Axes are scorable by someone else.** Hand the feature frame and axes to a peer and ask them to score. If their scores differ from yours by more than 1 on any axis, the axis definition is ambiguous and must be sharpened.
2. **Rule trace is complete.** For the worked application, every rule R1..R8 appears with a fired / did-not-fire note. Skipping rules hides reasoning.
3. **Output contract is testable.** If `O >= 4`, the artefact names the schema (JSON Schema, Protobuf, Pydantic) and the repair loop's maximum attempts.
4. **Threat posture appendix exists.** For the picked regime, the appendix names at least one MITRE ATLAS tactic and at least one OWASP ML Top 10 item, each with a specific mitigation in the design (not "we will monitor").
5. **Freshness math checks out.** If `F >= 3`, the design names how often the retrieval corpus / fine-tune dataset is refreshed and how long stale is tolerable.
6. **Rollback is named.** One sentence per regime on what gets flipped and where the fallback lands.
7. **Success criteria are measurable.** Each criterion has a number, a method, and a frequency.
8. **"What would change the pick" exists.** Thresholds, not vibes. "If labeled data crosses N and RAG quality plateaus below X on the frozen eval, revisit with PEFT."
9. **Last-reviewed date within 90 days.**

Static check:

```bash
grep -c "^Rule R" llm-regime-decision.md                    # expect at least 8 rule lines
grep -c "Worked application\|Worked example" llm-regime-decision.md   # expect >= 1
grep -ci "OWASP\|MITRE ATLAS\|NIST AI RMF" llm-regime-decision.md     # expect >= 1
grep -ci "rollback" llm-regime-decision.md                   # expect >= 1
grep -ci "last reviewed" llm-regime-decision.md              # must exist
```

## 4. Rubric / review checklist

A reviewer should verify the following. Unchecked items are reasons to send the artefact back.

- [ ] Feature frame is six lines or fewer, states task in "given X, produce Y under Z" form, names user, input distribution, output contract, and failure cost per direction.
- [ ] Regime catalogue enumerates at least five regimes the team could credibly pick (prompt-only, RAG, PEFT, full fine-tune, hybrid), each with data requirements, serving cost shape, and primary failure mode.
- [ ] Seven axes defined with scoring anchors specific to the team's problem class.
- [ ] Collapse rules ordered (not a bag), each naming regimes it selects or excludes.
- [ ] Tie-break ladder is explicit and ordered, with supply-chain surface and silent-drift exposure included.
- [ ] One worked application with (a) scored axis vector, (b) complete rule trace, (c) tie-break reasoning if applicable, (d) a one-paragraph recommendation, (e) measurable success criteria, (f) a rollback story, (g) a "what would change the pick" paragraph.
- [ ] Threat posture appendix cites MITRE ATLAS tactics and OWASP ML Top 10 items specifically relevant to the picked regime, each with a concrete mitigation.
- [ ] Any score `S >= 4` has a named mitigation (validator, human review, fallback regime) layered on top of the regime pick. NIST AI RMF alignment is cited.
- [ ] Any regime involving a hosted API or external checkpoint has a supply-chain note (licence, pin / alias policy, deprecation plan).
- [ ] "Last reviewed" date within 90 days.
- [ ] The design never picks fine-tuning on a label-starved axis (`L <= 2`) and never picks prompt-only on a private-knowledge axis (`K >= 4`).

Review-weighted points (for grading):

- Feature frame and output contract specificity: 15%
- Rubric construction (axes, anchors, rules, tie-break): 25%
- Worked application (rule trace + success criteria + rollback): 25%
- Threat posture integration (OWASP ML, MITRE ATLAS, NIST AI RMF cited where they land, with mitigations): 20%
- "What would change the pick" paragraph (falsifiability): 10%
- Hygiene (last-reviewed date, static checks pass): 5%

## 5. Common mistakes

- **Picking fine-tuning before trying RAG.** Teams routinely reach for a fine-tune when the knowledge is in private documents. Fine-tuning *the model* on your documents is a worse way to keep answers aligned with a changing document corpus than retrieving from the corpus at request time. Rule R6 encodes this.
- **Treating "the model seems to know this" as evidence.** Vibes on a 10-example desk test are not an eval. A frozen golden set with a documented scoring protocol is. Rule R1 depends on *knowledge location*, which you measure by auditing the golden-set outputs for factual support, not by asking the model nicely.
- **Over-promising structured output.** "The model usually returns JSON" is not a contract. If `O >= 4`, the design must have a schema, a validator, and a repair loop with a hard retry cap.
- **Ignoring the vendor supply chain.** A hosted LLM is a dependency that can be re-versioned under you; a pinned alias is often not actually pinned to weights. MITRE ATLAS surfaces this; the design must name the pin policy and the drift-detection plan.
- **Scoring the failure-mode axis too low.** "There's a human in the loop" is a mitigation, not a reason to score `S = 1`. The score reflects what would happen *if* the mitigation failed; the mitigation shows up separately under R7.
- **Forgetting labels can be grown.** `L` is a snapshot, not a verdict. The "what would change the pick" paragraph is where the team commits to data-collection work that would unlock fine-tuning later.
- **Picking hybrid because it sounds safer.** Hybrid is more moving parts, each with its own threat surface and its own monitoring cost. Hybrid is right when it is the only way to satisfy two rules that conflict (e.g., `F >= 4` from R6 and `L >= 4` from R3). Otherwise it is a confession that the team did not want to commit.
- **No rollback story.** A regime pick without a rollback plan is a lock-in disguised as a design. Every regime, including the "do nothing" baseline, must be named.
- **Writing the "what would change the pick" paragraph in prose.** It must contain thresholds. "If our labeled set grows and the RAG eval stops improving" is not falsifiable; "if our cleaned labeled set crosses 5,000 pairs *and* the RAG accepted-draft rate on the frozen golden set stays below 75% for two consecutive monthly measurements" is.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. Source of the MAP / MEASURE / MANAGE / GOVERN framing used for the harm axis (S), the risk-treatment layering in Rule R7, and the model-lineage work implied by the fine-tuning regimes.
- MITRE ATLAS — <https://atlas.mitre.org/>. Source of the supply-chain and vendor-model-swap threat surface called out in the tie-break ladder and the threat posture appendix, and of the context-manipulation lens used for RAG corpora.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source of the prompt-injection, insecure-output-handling, and data-poisoning lenses referenced per regime in the catalogue and in the threat posture appendix.

Local context (authoritative for the rubric's taxonomy and axes):

- Paired lesson: `lessons/mod-304-production-llm-integration/exercises/exercise-01-prompt-vs-rag-vs-finetune-decision.md`.
- Module README: `modules/mod-304-production-llm-integration/README.md`.
- Related exercise (hybrid design): `modules/mod-304-production-llm-integration/exercise-03-hybrid-classical-plus-llm-feature/SOLUTION.md`.
- Related exercise (cost/latency guardrails that stack on any pick): `modules/mod-304-production-llm-integration/exercise-04-llm-cost-and-latency-guardrails/SOLUTION.md`.
