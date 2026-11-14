# exercise-03 — Hybrid Classical + LLM Feature (Reference Solution)

Paired exercise: `lessons/mod-304-production-llm-integration/exercises/exercise-03-hybrid-classical-plus-llm-feature.md`.

## 1. Solution overview

This exercise produces `hybrid-design.md` — a design document for one production feature that composes a **classical model** (logistic / GBDT / rules) with an **LLM** behind a single interface. The artefact has five parts:

1. A **role split** — what the classical component decides, what the LLM composes, and what the orchestration layer between them is responsible for. A hybrid where both components can veto each other is one design; a hybrid where only one calls the LLM is a different, usually better one.
2. A **gating policy** — the rule that decides, per request, whether the LLM is called at all. Classical models in a hybrid usually do two jobs: they produce the signal *and* they decide when calling the LLM is worth it.
3. A **fallback ladder** — the explicit degrade path when the LLM is slow, failing, flagged by a safety filter, or returning schema-invalid output. Hybrid without a fallback is just "LLM with a nicer label."
4. A **monitoring and drift plan** — classical and LLM components drift differently and must be monitored separately before anyone is woken up.
5. A **threat posture** — the hybrid inherits threats from both sides. Prompt injection enters via the LLM leg; data poisoning and distribution drift enter via the classical leg; adversarial inputs can traverse both.

The reference solution below works a fully-labeled illustrative example through the design and gives the grading rubric that applies across every learner's hybrid.

The exercise exists because hybrids are the regime that the previous two exercises most frequently recommend (exercise-01 Rule R5, exercise-02 serving-time fallback), and because a hybrid designed badly is strictly worse than either of its parts.

## 2. Worked answer / implementation

### 2.1 Role split

Four canonical role splits, and when each is right:

| Shape | Who decides | Who composes | When it fits |
|---|---|---|---|
| **Classifier-gated LLM** | Classical model gates whether the LLM is called at all. | LLM composes output when the gate opens. | Most common and lowest-risk shape. Classical handles the easy / confident majority of traffic; the LLM is reserved for the long-tail minority. |
| **LLM-gated classical** | LLM reads the request, picks which classical model / skill to invoke. | Classical model produces the final answer. | When the hard problem is *routing*, not *answering*. The LLM is used for its natural-language understanding, not its generation. |
| **LLM output, classical validator** | LLM composes. | Classical model (or rule set) validates / scores / rejects / repairs the LLM output. | Strict output contracts, safety-sensitive outputs, or any case where the LLM's confidence is unreliable. |
| **Ensemble / blended score** | Both produce a signal; a fixed or learned combiner fuses. | Combiner. | Scoring/ranking tasks where both components see the same input. Rare in LLM hybrids because LLMs are expensive relative to their contribution to a stable score. |

Senior-engineer notes:

- **The default should be "classifier-gated LLM."** It is the shape that most directly bounds cost, latency, and blast radius. Reach for other shapes only when there is a specific reason the gating shape does not fit.
- **"Both can veto" is a smell.** Two components each with a kill switch produce four decisions and two monitoring surfaces. Prefer one component with authority and one component with a bounded role.
- **Validators are not just schema checks.** A content-level validator (e.g., "every claimed KB citation resolves to a KB article that actually contains the claim") is a classical component. Design it like one.

### 2.2 Gating policy

The gating policy is itself an artefact. It names:

- **The signal.** What classical model or rule produces the gate score. Probability from a calibrated GBDT, a confidence threshold on a classifier, a slice-level rule set, or a composite.
- **The threshold(s).** A gate score ≥ T opens the LLM path; otherwise the request takes the classical-only path. If `T` is tunable, the knob is in config, not source.
- **Secondary conditions.** Request size (token budget), user tier, feature flag, cost budget consumed so far this hour, upstream latency pressure.
- **The explicit default.** When any gate signal is unavailable (classical model is down, embedding service is slow), what is the default? Opening the gate (expensive) or closing it (conservative)? Pick one and write it down.

A gating policy template:

```yaml
# feature: support-draft
gate:
  signal:
    producer: "ticket-classifier-v7"              # existing classical model
    output: "p_llm_worthwhile"                    # probability the LLM will meaningfully improve this ticket
    calibration: "platt-scaled against 2026-10 held-out set"
  primary_threshold: 0.35                          # set so ~30% of traffic goes to LLM
  secondary_gates:
    - name: "cost_budget_guard"
      condition: "hourly_llm_spend < $5.00"
      on_closed: "deny_llm, log 'cost_ceiling'"
    - name: "feature_flag"
      condition: "flags.support_draft_enabled == true"
      on_closed: "deny_llm, log 'flag_off'"
    - name: "token_budget"
      condition: "estimated_input_tokens <= 6000"
      on_closed: "deny_llm, log 'oversized_input'"
  default_on_signal_unavailable: "closed"           # conservative: fall back to classical-only path
  observability:
    emit: ["gate_decision", "gate_score", "signal_source", "closed_reason"]
```

Three invariants:

- **The classical gate is calibrated.** An uncalibrated probability as a gate threshold is a hidden lever; the operator who turns the knob next week has no basis to pick a new value. See the paired module's calibration exercise.
- **"Closed" is the safe default.** Any failure of a gate input defaults closed. Hybrid regressions most commonly occur when a gate degrades to "always open" silently.
- **Gate decisions are first-class telemetry.** `gate_decision`, `gate_score`, `closed_reason` are emitted on every request. Without them, you cannot tell whether a change in LLM cost is from traffic, gate drift, or a vendor price change.

### 2.3 Fallback ladder

Every LLM call has a non-zero chance of being slow, failing, returning invalid output, or being flagged by a safety filter. The ladder names what happens in each case:

```
Step 1  Call LLM with pinned model, hard per-call timeout T_llm (e.g., 2.5 s).
Step 2  If response arrives and schema validates → done.
Step 3  If response arrives but schema invalid and repair_enabled:
           retry once with a repair prompt, hard timeout T_repair (e.g., 1.5 s).
Step 4  If repair fails or disabled: log, emit metric, fall through to Step 5.
Step 5  If a safety filter flagged the request (vendor-side or in-house):
           emit metric, do not retry, go to Step 7.
Step 6  If LLM timed out or errored at Step 1:
           emit metric. If circuit breaker open, go to Step 7 directly for the next N seconds.
Step 7  Classical-only fallback: return the classical model's answer (or the "no draft; human writes it"
           path), labeled so the UI / downstream service can distinguish it.
Step 8  If classical fallback itself unavailable: return the explicit safe refusal
           ("feature temporarily unavailable"). Never block the upstream request.
```

Senior-engineer notes:

- **Every step has a metric.** `hybrid.llm.ok`, `hybrid.llm.timeout`, `hybrid.llm.schema_invalid`, `hybrid.llm.safety_filtered`, `hybrid.repair.ok`, `hybrid.fallback.classical`, `hybrid.fallback.safe_refusal`. Monitoring the ladder is how you know whether the LLM is actually earning its slot.
- **The circuit breaker is mandatory.** Without one, a slow vendor API consumes the request-handling pool and takes the whole feature down. A simple "open if `>= 5` consecutive timeouts in the last 30 s; stay open for 60 s" is a workable floor.
- **Fallbacks must be labeled.** The downstream UI needs to know "this draft came from the classical fallback" vs. "this came from the LLM." Without the label, agents learn to distrust everything.

### 2.4 Monitoring and drift

The two components drift differently and must be monitored separately. Minimum signals:

Classical leg:

- **Input distribution** — feature-by-feature drift vs. training reference. PSI / KS per key feature.
- **Score distribution** — the gate score's distribution over time. A sudden shift means the gate is firing more or less often even if nothing else changed.
- **Calibration** — reliability diagram re-fit monthly on a held-out slice. The gate's probability-to-threshold mapping is only as good as its calibration.

LLM leg:

- **Invocation volume and cost** — calls / hour and $ / hour, with alerts on upward anomalies.
- **Latency distribution** — p50, p95, p99 end-to-end and per-segment (render + call + validate + repair + fallback).
- **Schema validity rate** — % of first-try responses that pass the output schema.
- **Repair attempt rate** — % of requests that needed the repair loop; this is a proxy for prompt + model fit.
- **Safety filter hit rate** — % of requests flagged, by category.
- **Golden-set eval on production inputs** — sampled, re-run weekly against the frozen golden set. If the vendor swaps weights, this is the canary.

Joint signals:

- **Gate-vs-outcome** — among requests the gate opened, what fraction had a successful LLM response and a validated schema? A declining joint-success rate is the hybrid's earliest drift signal.
- **Fallback share** — ratio of classical-fallback answers to LLM answers. Trending up means something upstream is degrading.

Drift triage runbook (mandatory in the artefact):

- "Cost up, volume flat, schema-valid up" → likely vendor swap with larger model behind alias; verify pinned_id, check calibration on sampled production inputs.
- "Volume flat, schema-valid down" → vendor silent model swap; roll back pinned_id if available; open eval against current pinned_id.
- "Volume up, gate score distribution stable" → feature traffic actually grew; nothing to do in the model, but the cost guard may fire.
- "Volume up, gate score distribution shifted right" → input distribution changed; classical model needs re-evaluation.

### 2.5 Threat posture

The hybrid inherits threats from both legs. The design must name, per leg, the specific threat and the specific mitigation.

LLM leg (OWASP ML Top 10 + MITRE ATLAS lenses):

- **Prompt injection** — mitigated in the system prompt (treat user content as data), reinforced by an output validator that rejects obviously-injected outputs, exercised in the golden set. See also `modules/mod-304-production-llm-integration/exercise-02-prompt-as-code/SOLUTION.md`.
- **Insecure output handling** — downstream consumers treat LLM output as untrusted data, never as code or SQL. Schema validation is necessary, not sufficient.
- **Model supply chain** — pinned_id, not alias; monitored for silent swaps via the production golden-set canary.

Classical leg (OWASP ML Top 10 + MITRE ATLAS lenses):

- **Training-data poisoning** — pipeline provenance, held-out audit set with known-good labels re-scored on each retrain.
- **Input-feature drift** — PSI monitoring; gate behaviour is a direct function of the classical model's input distribution.
- **Adversarial inputs** — inputs crafted to flip the gate score. If the gate opening is costly (or safety-sensitive), the gate should include rate limits and anomaly scoring per user.

Joint / orchestration threats:

- **Gate bypass via label confusion** — a request that looks easy to the gate but is adversarial to the LLM. Mitigation: downstream validator never trusts the gate; it re-checks the output regardless of which leg produced it.
- **Fallback forcing** — an attacker who can force LLM timeouts (e.g., very large inputs) to always land on the classical fallback. Mitigation: input-size caps enforced *before* the gate.

### 2.6 Illustrative worked application (not a real feature)

> **Illustrative feature** — "support draft triage." An internal support tool classifies inbound tickets into twelve queues. For 70% of tickets the classifier is confident; for the remaining 30% the team wants an LLM to produce a short rationale and queue recommendation before the human reviews. This example is a composite for calibration; do not treat it as a real employer or incident.

**Role split.** Classifier-gated LLM. Classical GBDT classifier is authoritative for queue assignment when confident; the LLM is called only to produce the rationale and a second-opinion queue suggestion on the uncertain tail. The LLM never overrides the classifier's queue assignment; its output is attached to the ticket for the human reviewer. (This is the "LLM output, classical validator" shape layered on top of a "classifier-gated LLM" shape — the validator is the classifier itself.)

**Gating policy.**

```yaml
gate:
  signal:
    producer: "ticket-classifier-gbdt-v7"
    output: "top_class_probability"
    calibration: "Platt against 2026-10 held-out, re-fit on each retrain"
  primary_threshold: 0.75              # top-class p >= 0.75 → classical-only; otherwise LLM invoked
  secondary_gates:
    - name: "cost_budget_guard"
      condition: "hourly_llm_spend < $3.50"
      on_closed: "classical_only"
    - name: "oversize_guard"
      condition: "ticket_body_tokens <= 6000"
      on_closed: "classical_only"
    - name: "feature_flag"
      condition: "flags.support_rationale_enabled == true"
      on_closed: "classical_only"
  default_on_signal_unavailable: "classical_only"
  observability:
    emit: ["gate_decision", "classifier_top_p", "closed_reason"]
```

**Fallback ladder.** Per §2.3. The ladder's "classical fallback" step is the natural one here: the classifier's top class is already the primary decision; the LLM rationale is purely additive. If the LLM leg fails or times out, the UI simply does not render a rationale panel, and the ticket proceeds with the classifier's assignment.

**Monitoring and drift.** Per §2.4. Gate-vs-outcome is the hybrid-specific signal: among tickets where the gate opened (classifier was unconfident), how often did the LLM's suggested queue match the classifier's eventual (human-confirmed) queue? That number tracks the hybrid's value; if it drops, the LLM is no longer adding rationale that corresponds to reality.

**Threat posture.** Per §2.5, with one feature-specific note: because the LLM output is rendered to human reviewers who may trust it, insecure output handling is the dominant risk. Rationale text is escaped, never rendered as HTML. Any URLs the LLM emits are validated against the KB domain allowlist before being shown.

**Success criteria (measured on a frozen golden set of 100 uncertain-tail tickets at launch, re-measured monthly):**

- ≥ 60% of LLM-produced rationales are rated "useful" by the reviewing agent.
- ≤ 5% of rationales contain a factual claim unsupported by the ticket body or the KB snippets (manual audit).
- p95 added latency ≤ 2.0 s and mean added cost ≤ $0.015 per uncertain-tail ticket.
- `fallback.classical` share < 5% on a steady-state day.

**Rollback story.** Kill switch (`flags.support_rationale_enabled = false`) disables the LLM leg entirely in one deploy; the classical classifier continues to run unchanged.

**What would change the pick:**

- If the classifier's top-class p-distribution shifts such that the "uncertain tail" grows past 40% of traffic, revisit the design: the LLM leg is no longer the exception path.
- If reviewer "useful" rating stays below 50% for two consecutive monthly measurements after a prompt tuning cycle, revisit the role split — perhaps the LLM should route (LLM-gated classical shape) rather than compose rationale.
- If cost-per-rationale creeps past $0.025 despite the cost guard, revisit model choice before revisiting the hybrid.

### 2.7 What a full artefact contains

The learner's `hybrid-design.md` should contain, in order:

1. The **role split** (§2.1) with an explicit named shape.
2. The **gating policy** (§2.2) as a readable config, with calibration cited and secondary gates enumerated.
3. The **fallback ladder** (§2.3) with a metric name per step.
4. The **monitoring plan** (§2.4) separating classical signals, LLM signals, and joint signals, with a drift triage runbook.
5. The **threat posture** (§2.5) naming OWASP ML Top 10 items and MITRE ATLAS tactics inherited per leg, each with a specific mitigation.
6. At least **one worked application** for the team's candidate feature with success criteria, rollback story, and a "what would change the pick" paragraph.
7. A **"last reviewed"** dated line.

## 3. Validation steps

Walk through this checklist on the artefact:

1. **Role split is named.** The artefact names one of the four canonical shapes (or justifies a fifth) and does not describe a design in which both components can veto each other without saying how conflicts resolve.
2. **Gate default is "closed."** Reading the gating config, when any required signal is unavailable, the LLM is not called. Flipping this to "open on failure" must be flagged as a design change.
3. **Fallback ladder is exhaustive.** Every branch of "LLM call might fail" has a step: timeout, error, safety filter, schema invalid, repair failed, circuit breaker open.
4. **Every ladder step has a metric name.** Grep for `hybrid.*` in the artefact; expect at least one metric per step.
5. **Calibration of the gate is named.** The gating config references the calibration method and the dataset it was fit on, and a re-fit cadence.
6. **Circuit breaker exists.** The artefact names the threshold, the open duration, and the behaviour when open.
7. **Monitoring separates the three signal groups.** Classical, LLM, joint.
8. **Drift triage runbook exists.** At least three "symptom → likely cause → action" rows.
9. **Threat posture cites both OWASP ML and MITRE ATLAS.** With specific mitigations, not "we will monitor."
10. **Worked application has success criteria, rollback, and "what would change the pick."**

Static check:

```bash
grep -Eci "role split|gating policy|fallback ladder" hybrid-design.md       # >= 3
grep -Eci "circuit breaker" hybrid-design.md                                 # >= 1
grep -ci "default_on_signal_unavailable" hybrid-design.md                    # >= 1
grep -ci "OWASP\|MITRE ATLAS\|NIST AI RMF" hybrid-design.md                  # >= 1
grep -ci "rollback" hybrid-design.md                                         # >= 1
grep -ci "last reviewed" hybrid-design.md                                    # >= 1
```

## 4. Rubric / review checklist

A reviewer should verify the following. Unchecked items are reasons to send the artefact back.

- [ ] Role split is one of the four canonical shapes (classifier-gated LLM, LLM-gated classical, LLM output + classical validator, blended) and justified.
- [ ] Gating policy names the signal, the threshold, secondary gates, and the "closed on signal unavailable" default.
- [ ] Gate signal is calibrated, and the calibration dataset and cadence are named.
- [ ] Fallback ladder names every failure mode (timeout, error, safety filter, schema invalid, repair failed, circuit breaker open) and every step has a metric name.
- [ ] Circuit breaker exists; threshold, open duration, and behaviour-when-open are specified.
- [ ] Monitoring separates classical, LLM, and joint signals.
- [ ] A drift triage runbook enumerates at least three "symptom → cause → action" rows.
- [ ] Threat posture cites specific OWASP ML Top 10 items and MITRE ATLAS tactics per leg with concrete mitigations.
- [ ] Worked application contains scored assessment of role split, gate thresholds (not symbolic — actual numbers), success criteria with thresholds, a rollback story, and a "what would change the pick" paragraph.
- [ ] Every fallback output is labeled distinctly from an LLM output in downstream telemetry.
- [ ] No design uses an LLM model alias (`latest`) as the pinned identifier.
- [ ] "Last reviewed" date within 90 days.

Review-weighted points (for grading):

- Role split justification: 15%
- Gating policy (signal, calibration, defaults, secondary gates): 20%
- Fallback ladder completeness and metric coverage: 20%
- Monitoring plan (three-way split + triage runbook): 15%
- Threat posture (OWASP ML, MITRE ATLAS references and specific mitigations): 15%
- Worked application (success criteria + rollback + "what would change the pick"): 10%
- Hygiene (last-reviewed, pinned_id, static checks pass): 5%

## 5. Common mistakes

- **Letting the LLM leg authority leak upward.** A "classifier-gated LLM" that quietly ships with the LLM rewriting the classifier's decision when it disagrees is actually a "blended / ensemble" shape, and must be designed, monitored, and reviewed as one. If disagreement handling is not in the design, assume the hybrid is leaking authority.
- **Uncalibrated gate thresholds.** Setting `gate_threshold = 0.5` on an uncalibrated classifier is a coin flip wrapped in a number. Calibrate first; thresholds without calibration are not comparable across retrains.
- **"Open on failure" as a default.** When the gate signal is unavailable, calling the LLM for everything is the shortest path to a bill shock. Default to the classical path.
- **No circuit breaker.** A slow LLM vendor without a breaker takes the whole feature down. This is the most common availability mistake in hybrids.
- **Fallback output indistinguishable from LLM output.** Reviewers and downstream services need to know which path produced each answer, or they lose trust in both paths.
- **Monitoring the LLM but not the classical leg.** The hybrid can regress from either side; most incidents in classifier-gated LLMs that *look* like "LLM got worse" are actually "gate signal drifted and the LLM is now seeing a different tail."
- **No drift triage runbook.** On-call has to figure out "cost up, volume flat, schema-valid up" at 3 AM. Pre-writing the three obvious diagnoses is a one-hour task that saves the on-call one bad hour.
- **Prompt-injection posture inherited and forgotten.** The LLM leg carries prompt-injection risk whether the design acknowledges it or not. The hybrid's validator step is a natural place to re-check output for injection symptoms; this is a one-line mitigation that is not in the typical design.
- **"Hybrid" as marketing.** A design labeled hybrid in which the LLM is called on 100% of traffic with a classical rubber-stamp is not a hybrid; it is the LLM path with a decorative companion. The gating ratio must be non-trivial and must be reported.
- **Neglecting input-size caps before the gate.** An attacker who can push oversized inputs into the system forces LLM calls (if the gate opens) or forces timeouts (if it does not). Cap input size *before* the gate decision.
- **Picking "LLM output + classical validator" and treating schema validation as the validator.** Schema validation is necessary and far from sufficient. The classical validator in that shape must evaluate *content* (e.g., citations resolve, numeric claims are plausible), not just shape.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. Source of the MAP / MEASURE / MANAGE / GOVERN framing used for the gate's "closed by default" invariant, the drift-triage runbook (MEASURE), and the two-leg risk accounting in the threat posture (GOVERN).
- MITRE ATLAS — <https://atlas.mitre.org/>. Source of the supply-chain, context-manipulation, and model-swap tactics attributed to the LLM leg, and the adversarial-input lens applied to the classical leg's gate.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source of the prompt-injection and insecure-output-handling posture on the LLM leg and the training-data poisoning / input-manipulation lenses on the classical leg.

Local context (authoritative for the hybrid's structure):

- Paired lesson: `lessons/mod-304-production-llm-integration/exercises/exercise-03-hybrid-classical-plus-llm-feature.md`.
- Module README: `modules/mod-304-production-llm-integration/README.md`.
- Related exercise (regime decision that frequently lands on hybrid): `modules/mod-304-production-llm-integration/exercise-01-prompt-vs-rag-vs-finetune-decision/SOLUTION.md`.
- Related exercise (operational practice for the LLM leg): `modules/mod-304-production-llm-integration/exercise-02-prompt-as-code/SOLUTION.md`.
- Related exercise (cost + latency guardrails that stack on the gating policy): `modules/mod-304-production-llm-integration/exercise-04-llm-cost-and-latency-guardrails/SOLUTION.md`.
- Calibration prerequisite for the gate signal: `modules/mod-303-advanced-modeling/exercise-03-calibration-and-uncertainty/SOLUTION.md`.
