# exercise-04 — LLM Cost and Latency Guardrails (Reference Solution)

Paired exercise: `lessons/mod-304-production-llm-integration/exercises/exercise-04-llm-cost-and-latency-guardrails.md`.

## 1. Solution overview

This exercise produces `llm-guardrails.md` — a design document plus a small set of runnable artefacts that bound the cost and latency of one LLM-backed feature in production. The artefact has six parts:

1. A **budget frame** — the cost ceiling (per request, per hour, per day) and the latency SLOs (p50, p95, p99) the feature must live within, and what breaches cost the business.
2. A **latency budget decomposition** — how the end-to-end p95 breaks down across render, retrieval, model call, validation, repair, and fallback, and which component is the long pole.
3. A **cost budget decomposition** — how the end-to-end cost per request breaks down across input tokens, output tokens, retrieval, re-embedding, and repair attempts, and which input most affects it.
4. A **guardrail set** — concrete mechanisms: timeouts, token caps, input-size limits, caching, streaming, request shaping, batching, rate limits, cost circuit breakers, model-tier routing. Each with its default value and when to tune it.
5. A **monitoring and alerting plan** — SLO burn rate, cost anomaly alerts, and the runbook for each.
6. A **degrade ladder** — the ordered series of things the system does when a guardrail is tripped, from "compress the prompt" to "skip the LLM entirely," so breaches do not take the feature down.

The reference solution below includes runnable sketches for the measurable guardrails (token capping, cache key, SLO burn rate calculation) and gives the grading rubric that applies to every learner's guardrail plan.

The exercise exists because LLM features are the first production ML features where **the per-unit economics are visible in real time and the vendor can change them under you**. A feature without guardrails is a feature whose cost is an outage waiting to happen.

## 2. Worked answer / implementation

### 2.1 Budget frame

Before any guardrail: write the budget.

- **Per-request cost ceiling.** The ceiling at which finance stops being happy. For a cost-per-interaction business, this is often cents or sub-cents; for an enterprise-contract business, it is dollars or tens of dollars.
- **Per-hour cost ceiling.** The ceiling at which paging someone is correct. Usually set so one bad hour does not destroy a monthly budget.
- **Per-day cost ceiling.** The ceiling at which the feature is turned off and someone writes a post-mortem.
- **p50 / p95 / p99 latency SLOs.** Separate numbers. p50 drives the median user experience; p95 drives the "noticeably slow for one in twenty" experience; p99 drives the "taking too long, user leaves" experience.
- **Latency breach cost.** What does the business lose when p95 goes from 2 s to 5 s? Users abandoning, downstream SLO cascades, support tickets, revenue. Without this, the budget is arbitrary.

Example (illustrative, labeled as such):

| Metric | Target | Breach consequence |
|---|---|---|
| Cost per draft | ≤ $0.03 | Finance alert at $0.045 over 10-minute window |
| Hourly cost | ≤ $5.00 | Page on-call at $4.00 sustained for 10 min |
| Daily cost | ≤ $80.00 | Feature auto-disables at $120, on-call wakes up |
| p50 latency | ≤ 1.0 s | Dashboard signal |
| p95 latency | ≤ 2.5 s | SLO; 2-week error budget |
| p99 latency | ≤ 5.0 s | Hard client timeout at 6.0 s |

### 2.2 Latency budget decomposition

End-to-end latency is the sum of parts; without a decomposition, tuning is guesswork. For a generic LLM feature:

```
total_p95 = render_p95 + retrieval_p95 + model_call_p95 + validate_p95 + (repair_prob * repair_p95) + overhead
```

Each term has its own instrumentation. An example p95 breakdown for a RAG-based feature with a 2.5 s target:

| Component | p95 budget | Mechanism if over | Typical knob |
|---|---|---|---|
| Request intake + input validation | 20 ms | Bounded; reject on exceeded size | Max input tokens |
| Retrieval (embedding + vector lookup + rerank) | 300 ms | Approximate ANN; cached embeddings | top-k, index type |
| Prompt render + token count | 20 ms | Bounded | — |
| LLM call (first token) | 700 ms | Streaming to user; smaller model | Model tier, prompt length |
| LLM call (remainder) | 1200 ms | Streaming; max_output_tokens cap | max_output_tokens |
| Schema validation | 20 ms | Bounded | — |
| Repair call (if invalid) | 800 ms × 8% = 64 ms amortised | Fewer repair attempts; better prompt | max_attempts |
| Overhead | 76 ms | — | — |
| **Total p95** | **≈ 2.4 s** | — | — |

Senior-engineer notes:

- **Instrument every term.** If the breakdown exists only in a design doc and not in metrics, you cannot tell which component is the long pole when the SLO burns.
- **Repair is amortised.** Repair attempts rarely blow p50 but are the typical p99 killer. Monitor repair rate separately from latency.
- **Streaming changes what "latency" means.** Time-to-first-token (TTFT) matters to the user; time-to-full-response matters to downstream consumers that need the whole message. Pick the right one per user; report both.

### 2.3 Cost budget decomposition

For a hosted-LLM feature, cost per request is approximately:

```
cost_per_request = input_tokens * price_input + output_tokens * price_output
                 + retrieval_cost + (repair_prob * cost_per_repair)
                 + (reembedding_cost_per_update / updates_per_period * cache_miss_rate)
```

Three observations senior engineers need to internalise:

- **Input tokens dominate for RAG.** When you stuff top-k documents into the prompt, input tokens can be 10×–50× output tokens. "The output is only 200 tokens" is not the relevant number.
- **Output tokens are capped by you, not the model.** The `max_output_tokens` knob is the single most effective cost lever. Set it to the largest value the use case actually needs, not to the vendor default.
- **Repair compounds.** A 10% schema-invalid rate on first attempt means repair cost is 10% of the first-attempt cost — not a rounding error at scale.

Example per-request cost table (illustrative):

| Term | Default | Cost | Lever |
|---|---|---|---|
| Input tokens | ~5000 (ticket + 5 KB snippets of ~800 tokens each) | ~$0.015 | Snippet count, snippet length, summarisation |
| Output tokens | ~250 (capped at 500) | ~$0.004 | `max_output_tokens`, prompt that encourages brevity |
| Retrieval + rerank | — | ~$0.001 amortised | top-k, rerank model tier |
| Repair (~8% probability × ~$0.010) | — | ~$0.001 amortised | Better prompt, better schema, `max_attempts = 1` |
| **Total per request** | — | **~$0.021** | — |

### 2.4 Guardrails

A guardrail is a specific mechanism with a default and a documented tune. The set below is the minimum; the design must name defaults for each.

**Input / output capping.**

- `max_input_tokens` — hard cap at the application layer. Oversized requests are rejected at intake, not at the vendor. Default based on budget decomposition, not the vendor maximum.
- `max_output_tokens` — hard cap passed to the vendor. Default = largest value the use case needs, not vendor default.
- `max_snippets_in_rag_prompt` — hard cap on retrieved context included. Larger `top-k` for retrieval, smaller `k` for prompt stuffing.

**Timeouts and retries.**

- `model_call_timeout_ms` — hard per-call timeout. Default = p99 target + small buffer, not "use the vendor default" (which is often minutes).
- `retrieval_timeout_ms` — separate from model timeout.
- `retries` — zero for idempotent LLM calls unless the error is explicitly retryable (429 with Retry-After). Default 0 and document why anything higher.

**Caching.**

- **Request-level cache.** Deterministic requests (same input, same prompt version, same pinned model) return the cached response. Key = `hash(model_pinned_id || prompt_version || normalized_input)`. Honest ask: how often does the exact same request recur? For some features (idempotent completion of a form field) the answer is "often"; for support drafts it is "almost never."
- **Prompt prefix cache.** Many vendors offer first-class prompt caching for a stable prefix (system prompt + stable context) with reduced input pricing. Design the prompt so the stable prefix is actually stable; do not interpolate request-specific values into it.
- **Embedding cache.** Re-embed on corpus updates, not per query. Monitor hit rate.

**Model-tier routing.**

- **Tiered models.** Route easy / short requests to a smaller model; reserve the larger model for the long tail. The router is itself a classical model or a rule set; see `exercise-03` for the hybrid design pattern.
- **Draft-then-review.** For some tasks, a small model produces a draft and the large model corrects only if a validator flags it.

**Request shaping.**

- **Summarise long inputs.** A ticket thread of 100 messages is compressed before being retrieved against. Summarisation is itself an LLM call, so measure the trade-off; it is only a win if the saved input-token cost exceeds the summariser call cost.
- **Batch where latency permits.** Async / batch endpoints are substantially cheaper than real-time for the same model. Use them for anything that can wait more than a few seconds.
- **Streaming.** For user-visible outputs, stream — it does not reduce total latency but reduces perceived latency and lets the user cancel.

**Cost circuit breakers.**

- **Per-hour cost cap.** When exceeded, the gate defaults closed and the feature falls back to the classical path. (If the feature has no fallback, breach triggers a hard refusal, not silent over-spend.)
- **Per-user and per-tenant caps.** Multi-tenant LLM features where one tenant can pay for everyone else's cost excursion are a known operational trap. Caps per tenant with telemetry per tenant.
- **Anomaly breaker.** 24-hour rolling cost baseline; alert when a 10-minute window exceeds 3× baseline. This catches silent vendor model swaps.

**Rate limits.**

- **Per-user / per-tenant rate limits.** Independent of cost caps. A single user cannot drive the entire feature.
- **Global rate limit.** Caps outbound requests to the vendor, protecting against the feature going viral.

Each guardrail in the design document has a one-liner: default, when to tune, metric that would warn you to tune it.

### 2.5 Monitoring and SLO alerting

The SLO maths is standard but worth writing down:

- **SLO:** p95 end-to-end latency ≤ 2.5 s, measured over a 28-day window.
- **Error budget:** 5% of requests may breach (or whatever the SLO target error rate is).
- **Burn rate:** actual error rate / allowed error rate. Burn rate > 1 means you will exhaust the budget before the window ends.
- **Multi-window alerts:** page on-call when (fast window = 5 min, burn ≥ 14.4) AND (slow window = 1 h, burn ≥ 6). Standard multi-window multi-burn-rate alerting.

Example burn-rate calculation sketch:

```python
def burn_rate(breach_count: int, total_count: int, slo_error_rate: float) -> float:
    """Burn rate = actual error rate / SLO's allowed error rate."""
    if total_count == 0 or slo_error_rate == 0:
        return 0.0
    return (breach_count / total_count) / slo_error_rate


def should_page(fast_burn: float, slow_burn: float,
                fast_threshold: float = 14.4, slow_threshold: float = 6.0) -> bool:
    """Two-window two-burn-rate paging rule (Google SRE workbook)."""
    return fast_burn >= fast_threshold and slow_burn >= slow_threshold
```

Cost alerting is simpler but easier to get wrong:

- Two levels of alert: **warn** (page nobody, dashboard colour change) and **page** (wake someone up).
- Thresholds are **relative to budget**, not absolute. "Spent 60% of hourly budget in 20 minutes" is more useful than "spent $3.00 this hour."
- Alerts fire on **sustained** anomalies (≥ 2 of the last 3 windows), not on a single spike.

Dashboards the on-call must have:

- Request volume, cost, and latency per minute.
- Cost per request distribution.
- Gate open rate (if a hybrid is in play).
- Schema-invalid rate and repair-attempt rate.
- Safety-filter hit rate.
- Model-pinned_id sanity (the current config, visible on the dashboard, next to recent latency / cost).

### 2.6 Degrade ladder

When a guardrail trips, the system must degrade in order rather than fail. The ladder:

```
Level 0 — Normal operation. All guardrails within budget.

Level 1 — Latency pressure (p95 trending high but SLO not yet burning fast):
   - Reduce max_output_tokens by 20%.
   - Drop one snippet from retrieval (top-k - 1).
   - Disable the repair loop; schema-invalid responses fall through to the classical fallback.

Level 2 — Cost pressure (hourly spend above warn threshold):
   - Close the gate on low-confidence traffic (if hybrid is in play).
   - Route all traffic to the smaller model tier.
   - Disable summarisation if it is net-positive only at normal cost.

Level 3 — SLO burning or hard cost ceiling reached:
   - Disable the LLM leg entirely; classical-only.
   - Return the "feature temporarily unavailable" refusal for use cases without a classical fallback.
   - Alert the owning team.

Level 4 — Vendor incident or safety-filter mass-reject:
   - Pin to a known-good vendor model version (if multi-vendor) or disable the feature.
   - Open an incident; the feature's kill switch is thrown in config, not via a code deploy.
```

Senior-engineer notes:

- **The ladder is written as config, not as code.** The levels are selectable from a dashboard or a runbook step; they do not require a redeploy.
- **Dropping to a cheaper model mid-incident is an anti-pattern** unless the smaller model has been evaluated for the feature on a frozen golden set. Degrading to a model whose output you have not validated is swapping a cost incident for a quality incident.
- **Every level transition is logged.** The feature can rightfully be at Level 2 for hours during a traffic spike; the on-call needs to know when and why, and roll back automatically when load normalises.

### 2.7 Illustrative worked application (not a real feature)

> **Illustrative feature** — "support draft reply" from the paired exercises. This example is a composite for calibration; do not treat it as a real employer or incident.

**Budget frame.**

| Metric | Target |
|---|---|
| Cost per draft | ≤ $0.03 |
| Hourly cost | ≤ $5.00 |
| Daily cost | ≤ $80.00 |
| p50 latency | ≤ 1.2 s |
| p95 latency | ≤ 2.5 s |
| p99 latency | ≤ 5.0 s |

**Latency decomposition.** Retrieval 300 ms, prompt render 20 ms, model call 1.9 s (TTFT 600 ms + remainder 1.3 s), schema validate 20 ms, repair (8% × 800 ms) 64 ms. Overhead budget 196 ms. Total p95 ≈ 2.5 s.

**Cost decomposition.** Input tokens (5 KB snippets × 800 tokens + ticket body ~1000 + system prompt ~500) ≈ 5500 tokens @ input price ≈ $0.014; output tokens (max 500, p50 ~250) @ output price ≈ $0.004; retrieval + rerank amortised ≈ $0.001; repair (~8% × $0.010) ≈ $0.001. Total ≈ $0.020. Headroom: $0.010 per request.

**Guardrails chosen (defaults):**

- `max_input_tokens` = 8000. `max_output_tokens` = 500.
- `max_snippets_in_rag_prompt` = 5. Retrieval top-k = 20 with reranker; prompt uses top 5 reranked.
- `model_call_timeout_ms` = 4500. `retrieval_timeout_ms` = 500. `retries` = 0.
- Request-level cache disabled (recurrence is near zero). Prompt-prefix cache enabled on system prompt (vendor-supported).
- Model-tier routing: all traffic through the production model tier for now; measure before adding a smaller tier.
- Cost circuit breaker at hourly $4.00 warn, $5.00 page, $7.50 auto-disable.
- Per-tenant rate limit: 30 drafts / minute. Global rate limit: 300 / minute.

**Monitoring.** All dashboard tiles per §2.5. Multi-window burn-rate alerting on p95 breaches. Cost alerting on sustained 60% / 80% / 100% of hourly budget. Daily cost summary auto-emailed to the owning team.

**Degrade ladder.** Per §2.6, with feature-specific actions in brackets. Level 2 action: close gate on classifier `top_p >= 0.75` (today it's ≥ 0.5). Level 3 action: disable LLM leg, fall back to no-draft (agent writes manually).

**Rollback.** Kill switch (`flags.support_draft_enabled = false`) disables the LLM leg entirely in one config flip.

**What would change the pick:**

- If `max_output_tokens = 500` is hit on > 10% of requests, revisit prompting toward brevity before raising the cap.
- If per-tenant rate limit (30/min) is hit by a single tenant more than once a week, revisit tenant onboarding / contract terms, not the cap.
- If a new vendor tier offers meaningfully lower input-token pricing, re-run the cost decomposition before adopting — input tokens dominate here, so a 40% cheaper input-token tier justifies the migration cost.

### 2.8 What a full artefact contains

The learner's `llm-guardrails.md` should contain, in order:

1. The **budget frame** (§2.1) with per-request, per-hour, per-day cost ceilings and p50/p95/p99 latency SLOs, each with a breach consequence.
2. The **latency decomposition** (§2.2) term-by-term, summing to the p95 target.
3. The **cost decomposition** (§2.3) term-by-term, summing to the per-request target, with the dominant term explicitly named.
4. The **guardrail set** (§2.4) with default values per guardrail and the metric that would prompt a tune.
5. The **monitoring and alerting plan** (§2.5), including multi-window burn-rate paging and cost alerting structured as warn/page/auto-disable.
6. The **degrade ladder** (§2.6) with levels named, actions per level, and the invariant that every level transition is logged.
7. At least **one worked application** with the budget frame, the decomposition, the chosen defaults, and a "what would change the pick" paragraph.
8. A **"last reviewed"** dated line.

## 3. Validation steps

Walk through this checklist on the artefact:

1. **Budget frame is complete.** All six rows present (per-request cost, per-hour cost, per-day cost, p50, p95, p99), each with a breach consequence.
2. **Latency decomposition sums to the p95 target.** Add up the components; they must sum within overhead.
3. **Cost decomposition names the dominant term.** For RAG features, this should be input tokens; if the design says otherwise, the decomposition is wrong or the feature is not RAG.
4. **Every guardrail has a default value.** Not `null`, not "TBD."
5. **Timeouts are shorter than the vendor default.** If the design relies on the vendor default timeout (minutes), it has no latency guardrail.
6. **Caching strategy is named explicitly.** Request-level, prompt-prefix, and embedding cache are each listed as enabled/disabled with the reasoning.
7. **Multi-window burn-rate alerting is specified.** Two windows, two thresholds; not a single threshold on a single window.
8. **Cost alerts are relative to budget.** Dollar amounts alone are not enough; the design cites percentage of hourly / daily budget.
9. **Degrade ladder has at least four levels.** Normal, latency pressure, cost pressure, hard-off. Each level has concrete actions and a log-the-transition invariant.
10. **Mid-incident model downgrade is forbidden unless pre-evaluated.** The degrade ladder's smaller-model action is gated on a frozen golden-set evaluation of the smaller model; otherwise the ladder does not change model tier.
11. **Rollback is one config flip, not a redeploy.**

Static check (sketch):

```bash
grep -Ei "per-request|per-hour|per-day" llm-guardrails.md | wc -l    # expect >= 3
grep -Ei "p50|p95|p99" llm-guardrails.md | wc -l                     # expect >= 3
grep -Ei "burn[- ]rate" llm-guardrails.md                            # expect >= 1
grep -Ei "degrade ladder|level [0-9]" llm-guardrails.md              # expect >= 4
grep -Ei "max_output_tokens|max_input_tokens" llm-guardrails.md      # expect >= 2
grep -ci "last reviewed" llm-guardrails.md                            # expect >= 1
```

## 4. Rubric / review checklist

A reviewer should verify the following. Unchecked items are reasons to send the artefact back.

- [ ] Budget frame complete: per-request / per-hour / per-day cost ceilings and p50/p95/p99 latency SLOs, each with a named breach consequence.
- [ ] Latency decomposition sums to the p95 target with per-component budgets.
- [ ] Cost decomposition names the dominant term and sums to the per-request target.
- [ ] `max_output_tokens`, `max_input_tokens`, `max_snippets_in_rag_prompt` all set to explicit numbers that come from the decomposition, not from the vendor default.
- [ ] `model_call_timeout_ms` set below the vendor default and justified against the p99 SLO.
- [ ] Caching strategy named per layer: request-level, prompt-prefix, embedding. Hit-rate target or explicit "disabled, because X."
- [ ] Multi-window multi-burn-rate paging on p95 SLO.
- [ ] Cost alerts phrased as percentages of budget, not raw dollar thresholds.
- [ ] Per-user and per-tenant rate limits specified for multi-tenant features.
- [ ] Degrade ladder has at least four levels, each with concrete actions and a log-the-transition invariant.
- [ ] Mid-incident model downgrade is gated on a pre-existing evaluation of the smaller model on the feature's golden set.
- [ ] Kill switch is a single config flip, not a code deploy.
- [ ] Worked application includes decomposition numbers, chosen defaults, and a "what would change the pick" paragraph.
- [ ] "Last reviewed" date within 90 days.

Review-weighted points (for grading):

- Budget frame + latency decomposition + cost decomposition: 25%
- Guardrail set (defaults, caching, timeouts, rate limits): 25%
- Monitoring + SLO alerting (multi-window burn rate, cost alerts): 20%
- Degrade ladder (levels, concrete actions, kill switch): 20%
- Worked application + "what would change the pick": 5%
- Hygiene (last-reviewed, no vendor-default timeouts, static checks pass): 5%

## 5. Common mistakes

- **Treating the vendor's default timeout as a guardrail.** The default for most hosted LLM APIs is minutes. If your feature inherits that default, you do not have a latency guardrail; you have a vendor-dependent one.
- **Capping `max_output_tokens` to the vendor default.** The default is almost always too generous. Measuring the p95 output length and setting the cap slightly above it is a one-line, measurable cost reduction.
- **Optimising output tokens before input tokens.** For RAG features the input side is where the cost lives; shaving output tokens is a rounding error compared to removing a snippet from the prompt.
- **One-window SLO alerting.** Fires on noise, misses the slow burn. Multi-window multi-burn-rate is the standard; use it.
- **Cost alerts as fixed dollar thresholds.** The threshold rots as the business grows. Alerts should be relative to the current budget.
- **Conflating "spent 100% of today's budget" with "stop the feature."** Those are different decisions; budgets can be exceeded on legitimate traffic. Pair cost alerts with volume and error-rate alerts, and page a human unless the overspend is extreme.
- **No per-tenant limits in multi-tenant features.** One tenant that runs away with the LLM feature can blow the whole budget. Per-tenant rate limits and per-tenant cost telemetry are mandatory, not optional.
- **Caching keys that include a mutable field.** A cache key that includes a wall-clock timestamp, a request id, or an unsalted user id cannot hit. Normalise inputs and omit per-request identifiers from the key.
- **Caching across prompt versions.** The cache key must include the prompt version id. Otherwise a prompt change silently serves old-prompt outputs.
- **Caching across model-pinned_ids.** Same reason. The key must include the pinned_id.
- **Repair rate ignored.** An 8% repair rate on first-attempt invalid output is a measurable cost and latency drag that can usually be driven below 1% with a better system prompt or a stricter schema. Measure it; tune it.
- **Downgrading to a smaller model mid-incident without evaluation.** Swapping models under load can trade a cost incident for a quality incident the next team notices a week later. Smaller-model paths must be pre-evaluated and kept warm.
- **Kill switch that requires a redeploy.** By the time the deploy ships, the incident is over or much worse. The switch must be a config flip visible to on-call.
- **Treating streaming as a latency fix.** Streaming reduces *perceived* latency for human-visible outputs. It does not reduce full-response latency for downstream consumers; if the downstream service needs the whole response, streaming does not help.
- **Failing to version the guardrails.** Guardrail defaults live in config alongside the prompt and model pin. "We lowered `max_output_tokens` last month, didn't we?" is not auditable if the value is baked into application code.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. Source of the MEASURE and MANAGE framing for the budget frame, the degrade ladder, and the monitoring invariants (log every level transition, alert on relative anomalies).
- MITRE ATLAS — <https://atlas.mitre.org/>. Source of the supply-chain / silent-vendor-swap threat that the cost-anomaly alert and the pinned_id dashboard tile are defending against.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source of the input-manipulation and output-handling lenses used in the input-size cap and the mid-incident model-downgrade guardrail.

Local context (authoritative for the guardrail defaults and the illustrative decomposition):

- Paired lesson: `lessons/mod-304-production-llm-integration/exercises/exercise-04-llm-cost-and-latency-guardrails.md`.
- Module README: `modules/mod-304-production-llm-integration/README.md`.
- Related exercise (regime decision that this guardrail set is layered on): `modules/mod-304-production-llm-integration/exercise-01-prompt-vs-rag-vs-finetune-decision/SOLUTION.md`.
- Related exercise (prompt-as-code project where guardrail defaults live in config): `modules/mod-304-production-llm-integration/exercise-02-prompt-as-code/SOLUTION.md`.
- Related exercise (hybrid design where the gate interacts with the cost circuit breaker): `modules/mod-304-production-llm-integration/exercise-03-hybrid-classical-plus-llm-feature/SOLUTION.md`.
- SLO burn-rate maths follows the multi-window multi-burn-rate pattern from the Google SRE Workbook (chapter on alerting on SLOs). Cite that chapter if the team wants a deeper reference.
