# exercise-01 — Modeling Regime Decision Rubric (Reference Solution)

Paired exercise: `lessons/mod-303-advanced-modeling/exercises/exercise-01-modeling-regime-decision-rubric.md`.

## 1. Solution overview

This exercise produces `regime-rubric.md` — a reusable decision rubric a senior ML engineer can hand to a team and defend in an architecture review. The artefact has three parts:

1. A **regime catalogue** that enumerates the modeling regimes under consideration (classical tabular, gradient-boosted trees, deep learning from scratch, fine-tuned foundation model, frozen foundation model + retrieval, prompt-engineered API model, hybrid) and names what each is good at and bad at.
2. A **decision rubric** — the axes you score a problem on, the threshold rules that collapse scores into a regime recommendation, and the explicit tie-break ordering.
3. A **worked application** of the rubric to one illustrative problem, so the reviewer can see the scoring work, not just the conclusion.

The reference solution below shows the intended structure, applies the rubric to one fully-worked illustrative problem, and gives the grading rubric that calibrates every learner's rubric to the same bar.

The exercise is learning-authored because the "right" regime depends on the team's actual constraints (data, latency, cost, skill, regulatory posture). The rubric exists so those constraints are made **explicit** before a regime is picked, not after.

## 2. Implementation (worked answer)

### 2.1 Regime catalogue (fill out per team)

For each regime your team might credibly pick, record:

- **Name** (e.g., "gradient-boosted trees on tabular features").
- **One-sentence description.**
- **Minimum viable team shape** (e.g., "one ML engineer, no GPU budget").
- **Typical training data floor** (order-of-magnitude, labeled vs. unlabeled).
- **Typical serving cost shape** (CPU-only? needs GPU? per-call API cost?).
- **Interpretability posture** (feature importances, SHAP-amenable, attention weights only, black box).
- **Failure modes it is known for** (brittle to feature drift, prone to hallucination, hard to debug, expensive to retrain, etc.).

A workable starter catalogue (adjust to your stack):

| Regime | One-line description | Data floor | Serving | Interp. | Typical failure |
|---|---|---|---|---|---|
| Linear / logistic + handcrafted features | Baseline. Scales to millions of rows cheaply. | ~10^4 labeled | CPU | High (coefficients) | Under-fits non-linear structure |
| Gradient-boosted trees (XGBoost / LightGBM / CatBoost) | Workhorse on tabular. | ~10^4 labeled | CPU | Medium (SHAP) | Feature drift silently degrades |
| Deep learning from scratch (CNN / RNN / Transformer) | Owns unstructured modalities when you have data. | ~10^6 labeled per task | GPU for training | Low | Expensive to iterate; needs infra |
| Fine-tuned foundation model | Transfer learning from a pretrained checkpoint. | ~10^3–10^5 task-labeled | GPU | Low | License / lineage obligations |
| Frozen foundation model + retrieval (RAG) | Embedding + nearest-neighbor + prompt. | 10^3 documents | API or GPU | Partial (citations) | Retrieval quality is the product |
| Prompt-only API model | Zero/few-shot via hosted LLM. | 0 labeled | API | Low | Vendor lock + cost at scale |
| Hybrid (retrieval → LLM → structured post-processor) | Compose multiple regimes behind one interface. | Varies | Mixed | Partial | Operational complexity |

### 2.2 Decision rubric — axes and scoring

Score the **problem**, not the model, on seven axes. Each axis is scored 1–5 (1 = cheap/easy, 5 = expensive/hard):

1. **Latency budget (L).** What is the p95 end-to-end latency the product can absorb? 1 = minutes, 5 = <50 ms synchronous.
2. **Interpretability requirement (I).** Does the business or a regulator need a per-prediction explanation? 1 = "a leaderboard is fine", 5 = per-decision reason codes required by policy or law.
3. **Label volume (V).** How many task-specific labeled examples exist today? 1 = tens of millions, 5 = under a hundred.
4. **Distribution stability (D).** How often does the input distribution change? 1 = effectively stationary, 5 = shifts weekly (e.g., fashion, news, fraud tactics).
5. **Cost ceiling (C).** Serving + training cost ceiling imposed by finance. 1 = unconstrained, 5 = must be CPU-only or sub-cent-per-call.
6. **Failure-mode severity (F).** Blast radius of a wrong prediction. 1 = annoyance, 5 = safety / financial / legal harm. This is the NIST AI RMF "harm" lens.
7. **Iteration speed need (S).** How fast must the team cycle? 1 = quarterly, 5 = multiple shipped changes per week.

The axes are deliberately not independent — they cross-constrain the regime set.

### 2.3 Collapse rules (how a score vector picks a regime)

Rules fire in order; the first rule whose conditions are met wins. If none fires, fall through to the tie-break ladder in §2.4.

```
Rule R1 (interpretability mandate):  I >= 4  →  linear model or GBDT with SHAP.
                                                 Deep / foundation regimes excluded.
Rule R2 (label-starved + stable):    V >= 4 and D <= 2  →  frozen foundation + retrieval,
                                                            or prompt-only API.
Rule R3 (label-starved + volatile):  V >= 4 and D >= 4  →  frozen foundation + retrieval
                                                            with monitored eval set; avoid fine-tuning
                                                            until you have a labeled drift sample.
Rule R4 (tight latency + tight cost):L >= 4 and C >= 4  →  GBDT or linear; API regimes excluded.
Rule R5 (high harm):                 F >= 4             →  prefer most-interpretable regime that
                                                            still meets accuracy floor; add human
                                                            review gate regardless of regime.
Rule R6 (ample data, unstructured):  V <= 2 and modality in {image, audio, text at scale}
                                                            →  fine-tuned foundation model or
                                                               deep-from-scratch if licensing blocks FMs.
Rule R7 (ample data, tabular):       V <= 2 and modality == tabular
                                                            →  GBDT; deep tabular only if ablation wins.
Rule R8 (default):                                       →  GBDT baseline first, iterate from there.
```

Two senior-engineer notes on the rules:

- Rule R1 is non-negotiable in regulated domains. "We can approximate the LLM output with a surrogate GBDT for explanations" is not an interpretability answer — the surrogate is a different model, and the regulator will ask about the deployed one.
- Rule R5 does not pick a regime by itself; it layers a **review gate** on top. This is consistent with the NIST AI RMF MANAGE function: a high-severity failure mode requires risk treatment, not just a different architecture.

### 2.4 Tie-break ordering

When two regimes pass all rules, prefer in this order:

1. The regime the team has **already deployed and monitored** in production. Familiarity is an evaluation signal — you know its failure modes.
2. The regime with the **cheapest rollback path**. GBDT behind a feature flag rolls back instantly; a fine-tuned LLM behind a vector index does not.
3. The regime that produces **the most testable artefact**. A GBDT with a held-out slice report beats a prompt string every time.
4. The regime with the **smallest supply-chain surface**. Rule-of-thumb: a model you train from a checkpoint you host is lower-risk than a vendor API that can be re-versioned under you. This is the MITRE ATLAS supply-chain lens.

### 2.5 Illustrative worked application (not a real problem)

> **Illustrative problem** — the "support-ticket router." A B2B SaaS support team wants to auto-route inbound tickets to one of twelve internal queues. This example is a composite for calibration; do not treat it as a real employer or incident.

Score the seven axes:

| Axis | Score | Reason |
|---|---|---|
| L (latency) | 2 | Tickets are asynchronous; tens-of-seconds is fine. |
| I (interpretability) | 3 | Managers want to know "why did this go to Queue 7?", but no regulator is in scope. |
| V (labels) | 4 | ~400 historical routed tickets with clean labels; everything else is noisy. |
| D (drift) | 3 | New product features monthly change vocabulary; not fast fashion, but not stationary. |
| C (cost) | 3 | Finance wants per-prediction cost <$0.01. |
| F (failure severity) | 2 | A mis-routed ticket is embarrassing, not harmful. |
| S (iteration) | 3 | Expect weekly improvements driven by new queue taxonomy. |

Rule trace:

1. R1? I = 3, not >= 4 → does not fire.
2. R2? V = 4 and D = 3 → V >= 4 is true, but D <= 2 is false → does not fire.
3. R3? V = 4 and D >= 4 → false → does not fire.
4. R4? L = 2 and C = 3 → false → does not fire.
5. R5? F = 2 → false → does not fire.
6. R6? V = 4 (not <= 2) → does not fire.
7. R7? V = 4 (not <= 2) → does not fire.
8. R8 (default) → GBDT baseline.

But the tie-break ladder matters because the default rule is weak. Reconsidering:

- The team already has GBDT infra deployed for another classifier (tie-break #1 favours GBDT or "stay in-regime").
- Rollback for GBDT is a model-version flip; rollback for a hosted-LLM router is a prompt rollback plus a vector-index re-embed (tie-break #2 favours GBDT).
- Testable artefact for GBDT = held-out confusion matrix per queue; testable artefact for an LLM prompt is a golden set + prompt string (tie-break #3 slightly favours GBDT).
- V = 4 (labels are scarce). This flags a real risk for GBDT: 400 examples across 12 queues is ~33 per class, which is borderline. A **hybrid** is worth considering — a frozen embedding model (foundation-model encoder) feeding a logistic classifier on top. That keeps the regime close to GBDT's operational profile but borrows representation from the FM.

Final recommendation (worked):

> **Regime:** frozen foundation-model text encoder → logistic classifier on top, with a confidence threshold routing to a human when the top-class probability is below 0.6. Rationale: V = 4 argues against training a deep classifier from scratch and against GBDT-with-hand-features for text; D = 3 argues for not fine-tuning the encoder yet (fine-tune after we have drift-labeled data); I = 3 is satisfied by per-class probabilities plus nearest-neighbor citations from the encoder's embedding space; F = 2 lets us skip a hard human-review gate but keep a soft confidence-threshold gate.

The write-up shows the rule trace **and** the tie-break reasoning — both are mandatory. A recommendation with no rule trace is not a rubric application; it is a hunch wearing a rubric costume.

### 2.6 What a full artefact contains

The learner's `regime-rubric.md` should contain, in order:

1. The **regime catalogue** (§2.1), filled in for the team's actual stack.
2. The **axis definitions** (§2.2), with the team's calibration of what "score 1" vs. "score 5" means on each axis for their specific problem class.
3. The **collapse rules** (§2.3), tuned if the team's regulatory posture changes a threshold.
4. The **tie-break ladder** (§2.4).
5. At least **two worked applications** — one from the team's existing production system (back-casting: does the rubric reproduce the regime you already chose?) and one from a problem the team is currently scoping.
6. A **failure-mode appendix** that names, per regime, the specific NIST AI RMF MEASURE harms and MITRE ATLAS threats the regime is prone to (e.g., a prompt-only API model inherits prompt-injection risk; a fine-tuned FM carries the lineage obligations of its base checkpoint).

## 3. Validation steps

Walk through this checklist on the artefact:

1. **Rubric is testable.** Hand the axes to a peer, give them the illustrative problem, and ask them to score. If their scores differ from yours by more than 1 on any axis, the axis definition is ambiguous and must be sharpened.
2. **Back-cast.** Apply the rubric to a regime your team has already shipped. If the rubric picks a different regime than what is in production, write the one-paragraph explanation — either the production pick was wrong (useful finding) or the rubric is wrong (also useful).
3. **Rule-trace is complete.** For every worked application, every rule R1..R8 appears with a fired/did-not-fire note. Skipping rules hides the reasoning.
4. **Tie-break ladder was actually used.** If two regimes passed and you picked one, the write-up cites which ladder step broke the tie. "I picked GBDT because it felt right" is not a rubric application.
5. **Harm axis (F) and NIST AI RMF alignment.** For any problem scored F >= 4, the write-up names the specific harm category and the mitigation. "F = 4 so we use GBDT" is incomplete — it must also name the human-review gate or the monitored eval.
6. **Supply-chain surface named.** For any regime involving a hosted API or an external pretrained checkpoint, the write-up names the licence, the version pinning strategy, and what happens when the vendor deprecates the endpoint.
7. **Rubric revisits itself.** The artefact has a dated "last reviewed" line. Rubrics rot — a six-month-old rubric predates whatever new regime landed since.

Static check:

```bash
grep -c "^Rule R" regime-rubric.md         # expect at least 8 rule lines
grep -c "Worked application"  regime-rubric.md   # expect >= 2
grep -c "NIST AI RMF\|MITRE ATLAS\|OWASP" regime-rubric.md   # expect >= 1
grep "last reviewed" regime-rubric.md      # must exist and be within 90 days
```

## 4. Rubric / review checklist

A reviewer should verify the following. Unchecked items are reasons to send the artefact back.

- [ ] Regime catalogue enumerates at least five regimes the team could credibly pick, each with data floor, serving shape, interpretability posture, and typical failure mode.
- [ ] Seven axes defined, each with scoring anchors ("what does 1 look like, what does 5 look like") specific to the team's problem class.
- [ ] Collapse rules are ordered (not a bag) and each rule names the regime(s) it selects or excludes.
- [ ] Tie-break ladder is explicit and ordered.
- [ ] At least two worked applications, each with (a) a scored axis vector, (b) a complete rule trace, (c) a cited tie-break step if applicable, and (d) a one-sentence recommendation.
- [ ] One of the worked applications back-casts an already-shipped system.
- [ ] Any regime involving a hosted API or external checkpoint is accompanied by a supply-chain note (licence, pin, deprecation plan). MITRE ATLAS context cited.
- [ ] Any problem scored F >= 4 has a named human-review or monitored-eval mitigation. NIST AI RMF alignment cited.
- [ ] "Last reviewed" date within the last 90 days.
- [ ] The rubric never selects a regime on a single axis alone. If a single axis would be sufficient, the rubric over-specifies the axes.

Review-weighted points (for grading):

- Rubric construction (axes, anchors, rules, tie-break): 35%
- Back-cast worked application (does it reproduce a real decision?): 20%
- Forward worked application (rule trace + tie-break reasoning): 20%
- Failure-mode / supply-chain integration (NIST AI RMF + MITRE ATLAS references land in the right places): 15%
- Hygiene (last-reviewed date, cross-references, static checks pass): 10%

## 5. Common mistakes

- **Picking a regime before scoring the axes.** The axis scores are supposed to constrain the regime; if the regime is pre-committed, the axes get retrofitted and the rubric is theatre.
- **Treating "accuracy" as an axis.** Accuracy is the dependent variable the regime is trying to deliver; it is not a problem constraint. The axes are *about the problem*, not about candidate models.
- **One-axis selection.** "F = 4 so we use a linear model" skips the rule trace and loses information. A high-harm problem still needs the latency / cost / label axes scored to pick among the interpretable regimes.
- **Hiding the tie-break.** Teams often pick the regime they already know and justify it afterward. The ladder exists to make that preference *explicit* (and often correct), not to pretend it isn't happening.
- **Forgetting the supply chain for hosted APIs.** "We'll use the vendor's LLM" is not a complete regime pick without a licence check and a pin strategy. This is the MITRE ATLAS supply-chain surface in practice.
- **Letting the rubric expire.** A rubric authored before a new regime landed (e.g., before cheap retrieval-augmented generation was available on your stack) will systematically mis-route problems to the regimes that existed at authoring time.
- **Scoring interpretability (I) by vibe.** I is a hard constraint derived from policy, regulation, or an explicit business commitment — not from "it would be nice to know why." If the team scores I = 4 everywhere, the rubric collapses to "always GBDT," which is a different failure mode.
- **Scoring drift (D) too low on text problems.** Text distributions shift whenever product vocabulary shifts. A score of D = 1 on a text problem is almost always wrong.
- **Writing rules that cannot be falsified.** "Prefer the right regime for the problem" is not a rule. Every rule in R1..R8 must have a condition on an axis score and an action naming or excluding specific regimes.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. Source of the MAP / MEASURE / MANAGE / GOVERN functions referenced by the F axis and Rule R5 (harm-driven review gates).
- MITRE ATLAS — <https://atlas.mitre.org/>. Source of the supply-chain and adversarial-threat surface referenced in the tie-break ladder and in the failure-mode appendix.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source of the regime-level threat posture noted per regime in the catalogue (e.g., API model → inherits input-validation threats).

Local context (authoritative for the rubric's taxonomy and axes):

- Paired lesson: `lessons/mod-303-advanced-modeling/exercises/exercise-01-modeling-regime-decision-rubric.md`.
- Module README: `modules/mod-303-advanced-modeling/README.md`.
