# exercise-01 — ML SLIs And SLOs (Reference Solution)

Paired exercise: `lessons/mod-307-ml-reliability-slos/exercises/exercise-01-ml-slis-and-slos.md`.

## 1. Solution overview

This exercise produces `slo.md` plus a small runnable artefact (`burn_rate.py`) — a written Service Level Objective document for one production ML surface, backed by the burn-rate math that converts the SLO into pageable alerts. The artefact has seven parts:

1. A **service frame** that pins down what the SLO covers, who owns it, which users feel it when it misses, and what decision the error budget feeds.
2. A **menu of SLIs** — the measurable signals, defined as "good-events / valid-events," covering availability, latency, request-level quality, freshness, and drift. Each SLI has a precise numerator and denominator, a measurement window, and a data source.
3. A **set of SLOs** — the targets on those SLIs, each with a measurement window and an error-budget size derived from the target, plus the business reason the target is where it is.
4. A **burn-rate alert policy** — the multi-window multi-burn-rate (MWMB) alerts that page on-call when the error budget is being consumed fast enough to matter, with page/ticket/email tiers.
5. A **monitoring contract** — the metric names, labels, cardinality limits, and dashboards that make the SLIs computable from telemetry a real on-call can trust.
6. An **error-budget policy** — what happens when the budget depletes: ship-freeze, launch-gate escalation, retraining pause, named owner.
7. A **review cadence** — when the SLOs are recalibrated, by whom, and what inputs (incident post-mortems, user escalations, cost ceilings) feed the next round.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric that calibrates every learner's SLO document to the same bar.

The exercise exists because an ML surface without SLOs is not operated; it is tended. "Latency should be good" is not an SLO — it is a hope. A well-written SLO lets an on-call engineer, a product owner, and a model owner all answer the same question with the same number: *is this system healthy enough to ship into, or do we stop and fix?* NIST AI RMF MEASURE calls for continuous measurement of AI system trustworthiness characteristics (accuracy, reliability, safety); SLOs are that measurement expressed as a user-facing contract. MANAGE grounds error-budget policy as the risk-control surface: when the budget is exhausted, risk mitigation becomes a constraint on launches, not a discussion in a meeting.

## 2. Worked answer / implementation

### 2.1 Service frame

Six lines, in this order:

- **Surface covered.** The user-facing endpoint, model version, and traffic shape the SLO protects. Pinned to a name. "The recommender" is not a surface; "`reco-serve` producing per-user item lists for the homepage feed of logged-in US web users" is.
- **Users who feel a miss.** Which cohort pays when the SLO breaches — all users, a segment, an internal consumer. An SLO owned by nobody affects nobody; the "who feels it" line prevents an SLO whose budget burn is politically invisible.
- **Decision the error budget feeds.** Ship-freeze above a budget-burn threshold, retraining pause, canary gate, or a lower-severity backlog. The decision is what makes the SLO load-bearing.
- **Named owners.** A primary owner (the ML/platform team that operates the surface) and a secondary stakeholder (the product team the surface serves). Roles, not people, so the SLO survives rotation.
- **Dependencies.** Upstream surfaces whose SLOs this one inherits — feature store, embedding service, authentication, authorization, LLM provider. A downstream SLO cannot be tighter than the composition of its upstreams.
- **Last reviewed.** A date, within the SLO's declared review cadence (default: quarterly; see §2.7).

### 2.2 SLI menu

An SLI is a **good-events / valid-events** ratio over a window. "Good" and "valid" are strings with definitions, not adjectives. The SLI menu for an ML surface pulls from five families; pick one or two per family, not all.

**F1. Availability SLIs.** The surface responds without a terminal error.

- *HTTP availability.* good = responses with status in `{200, 204, 404}` (404 is a user-caused miss, not a service error); valid = all terminated requests excluding client-cancelled.
- *Semantic availability.* good = responses whose body schema validates (e.g., non-empty item list for a ranker; non-null response for a generative endpoint); valid = all 200 responses. Semantic availability catches "200 with empty payload," which HTTP availability misses.

**F2. Latency SLIs.** The surface responds inside the user-perceivable budget.

- *End-to-end p95 latency.* good = requests whose p95 user-observed latency is below a threshold over the window; valid = all requests served. The common form is "99% of requests complete under T ms over 30 days," which is a latency SLI at a tail quantile.
- *Component latency breakdown.* good = each named stage (featurization, inference, post-processing, policy check) stays within its per-stage budget; valid = requests reaching that stage. Component SLIs let you debug which stage burned budget; the user-facing SLO is still the end-to-end number.

**F3. Quality SLIs.** The output has the property the user is paying for.

For a ranker/recommender: non-empty list, policy-clean items, diversity minimum, freshness of inventory. For a classifier: calibration within tolerance on a canary slice. For a generative endpoint: structured-output schema validity, refusal rate on a safety probe set, prompt-injection-success rate on a MITRE ATLAS-aligned adversarial-input slice, groundedness on a RAG canary set.

Quality SLIs are **not** offline accuracy. Offline accuracy is a release-time artefact (see `mod-305/exercise-01`). Quality SLIs are online, computed on live traffic (or a shadow of it), and they correspond to properties a user can feel.

- *Example (ranker):* good = served responses whose top-k passes the policy filter and whose non-empty rate is ≥ 95%; valid = all 200 responses.
- *Example (LLM endpoint):* good = responses matching the declared output schema AND not flagged by the safety classifier; valid = all 200 responses.

**F4. Freshness SLIs.** The data the model used is recent enough.

- *Feature freshness.* good = requests served with upstream feature snapshots whose age is below threshold (e.g., "last ingested row < 15 min for session features, < 24 h for daily features"); valid = requests requiring that feature.
- *Model freshness.* good = requests served by a model version whose training cutoff and evaluation window are within policy; valid = all served requests. Model freshness ties to `exercise-04` (retraining as a deploy): if the retrain pipeline stalls, model freshness burns the error budget before anyone notices the pipeline is red.

**F5. Drift and data-health SLIs.** The traffic the model sees resembles the traffic it was built for.

- *Input drift.* good = inference requests whose input-feature distribution (per-feature PSI or KS statistic vs. training window) is below threshold; valid = requests sampled for drift monitoring. The SLI measures the fraction of traffic in-distribution, not the raw drift value — the thresholded form gives you an error budget.
- *Label drift / performance proxy.* good = per-slice online proxy metric (click-through, acceptance, task-completion) stays within tolerance of the pre-launch baseline; valid = requests with an observable outcome. For LLM surfaces with no clean label, substitute a judge-score SLI (see `mod-305/exercise-04`) or an acceptance-rate SLI.

The SLI menu for one surface is typically 4–6 chosen SLIs across these families, not all possible SLIs. Too few SLIs misses failure modes users feel; too many dilutes on-call attention.

### 2.3 SLOs

An SLO attaches a target and a window to an SLI. The SLO is the statement: "over the window, the SLI will exceed the target." The error budget is `1 - target` over the window, expressed in the SLI's unit.

**Picking the window.**

- *30 days (rolling)* is the default for most user-facing surfaces. Long enough to average over weekly seasonality and short enough that recent incidents affect the budget.
- *7 days (rolling)* is tighter; useful for high-change surfaces (retraining weekly) and for alerting windows (see §2.4).
- *Quarterly* is used for very low-volume surfaces where a 30-day denominator is too small for meaningful statistics.

A window shorter than one weekly cycle is almost never right for an availability SLI; a weekday-heavy metric's "good day" vs. "bad day" moves the SLI more than real incidents do.

**Picking the target.**

Targets are a product/risk judgement, not a statistical artefact. The question: *what target makes the error budget large enough to deliver value at and small enough that a miss is a real problem?*

- A target "too tight" (e.g., 99.999% on a surface users would accept at 99%) burns engineering capacity on noise and starves launches.
- A target "too loose" (e.g., 95% on a surface users leave at 99%) hides real degradation inside the budget.

Common starting points, to be argued against the surface's user tolerance, not accepted by reflex:

- Availability: 99% to 99.9% for user-facing non-critical surfaces; 99.9% to 99.95% for paid tier surfaces. "99.9%" sounds like a cliché; it is also a sensible starting point because `1 - 0.999 = 43.2 min/month` of budget is a number teams can reason about.
- Latency: p95 latency "under T ms" where T is calibrated against user-experienced tolerance for that surface. For a search response, T is often 300–500 ms; for a long-form LLM completion, T might be 5–10 s. The key is not the number but that the number was chosen from user behaviour (abandonment rate vs. latency), not from a vendor dashboard default.
- Quality: 95–99% of responses pass the quality SLI depending on how visible a failure is. A non-empty list is nearly free to achieve and so sits at 99%+; a policy-clean generation is harder and sits lower.
- Freshness: thresholds tied to the business unit the data represents ("session features < 15 min" vs. "daily aggregates < 24 h").

**Expressing the SLO in words and in math.**

```
SLO:  99.9% of /predict requests return within 500 ms, measured over a trailing 30-day window.
Math: SLI_latency = count(latency_ms <= 500) / count(request) over 30d
      Target_latency = 0.999
      ErrorBudget_latency = 1 - 0.999 = 0.001 ( = 0.1% of requests may exceed 500 ms over 30d )
```

An SLO without numeric target, window, and budget is a wish.

### 2.4 Burn-rate alert policy

An alert fires when the error budget is being consumed *too fast*. "Too fast" is a burn rate: the ratio of "error budget spent in window W" to "error budget that would be spent at the target rate over the same window." A burn rate of 1.0 is nominal; a burn rate of 10 means the budget will be exhausted in 1/10 of the SLO window if the rate holds.

**Why not alert on the raw SLI.** The raw SLI over 30 days changes slowly; by the time it drops below target, the incident has been running for days. Alerting on the raw SLI either pages on noise or pages after the fact.

**Multi-window multi-burn-rate (MWMB).** Two alert tiers, each with a short and a long window. The combination catches both "fast large outage" (page immediately) and "slow slow burn" (ticket within hours).

```
Page alert (fast burn):
  Alert if (burn_rate(short_window=5m) > 14.4) AND (burn_rate(long_window=1h) > 14.4)
  Rationale: burn rate of 14.4 over 1h consumes 2% of a 30d error budget. That is
             "a page-worthy chunk of budget in an hour."

Ticket alert (slow burn):
  Alert if (burn_rate(short_window=30m) > 6)   AND (burn_rate(long_window=6h) > 6)
  Rationale: burn rate of 6 over 6h consumes 5% of a 30d error budget. That is
             "a slow burn worth paging during business hours."

Low-urgency alert (very slow burn):
  Alert if (burn_rate(short_window=2h) > 3)    AND (burn_rate(long_window=24h) > 3)
  Rationale: 10% of the 30d budget in a day; worth a ticket, not a page.
```

The two-window conjunction is the key move: the short window arms the alert ("something is actually happening now") and the long window suppresses it ("it has been happening long enough that this is not a blip"). An SLO backed only by a single-window alert either pages on 30-second blips or waits hours to notice a hard outage.

Formally, the burn-rate alert thresholds satisfy:

```
burn_rate × (alert_window / SLO_window) = fraction_of_budget_consumed_before_alert
```

Pick two targets ("page in ≤ 5 minutes if 2% of the budget is being consumed per hour" and "ticket in ≤ 1 hour if 5% is being consumed per 6 hours") and solve for the burn-rate thresholds. The policy cites those numbers, not just the thresholds; a reviewer must be able to reproduce them.

**Per-SLI alerts.** Each SLO gets its own burn-rate alert. A latency burn and an availability burn are different incidents and produce different runbooks (see `exercise-03`).

### 2.5 Monitoring contract

The SLO document is only as real as the telemetry that feeds it. The monitoring contract names:

- **Metric names.** `ml_request_total{model_version, surface, status, response_class}` as the valid denominator; `ml_request_latency_seconds_bucket{...}` as a histogram feeding the latency SLI; `ml_response_quality_total{quality_outcome}` for the quality SLI, and so on. Names are prefixed per team, not per model, so a model rotation does not break the SLO.
- **Label cardinality.** Each label is budgeted. A `user_id` label is forbidden (explodes cardinality); a `model_version` label is allowed (bounded); a `region` label is allowed (small set); a `request_id` label is forbidden (unbounded). The SLO document lists allowed labels and their domain so a new dashboard does not silently blow the budget of the metrics backend.
- **Sampling policy.** If an SLI is computed on a sample (drift and quality SLIs often are), the sample rate and the reservoir policy are named. "We drift-check 1% of requests, stratified by region" is a statement; "we drift-check" is not.
- **Data source of truth.** The metric backend (Prometheus, Datadog, Cloud Monitoring) and the retention window. If the SLO measures over 30 days but the backend retains 15 days of high-resolution data, the SLO is unmeasurable as written. This mismatch is discovered at review time, not when the alert fires.
- **Dashboards.** A named dashboard per SLO with: current SLI, target line, 30-day budget consumption, burn-rate panel for each alert window. The dashboard is the artefact the on-call opens first; it is not a decoration.
- **Freshness of the telemetry itself.** If the metric pipeline is 5 minutes behind, the fast-burn page is also 5 minutes behind. The monitoring-pipeline latency is a meta-SLI on the meta-system; it should be measured and bounded.

### 2.6 Error-budget policy

The error budget is **policy**, not a spreadsheet. The policy says what changes when the budget state changes.

```
Budget > 25% remaining:   Normal operations. Launches proceed with standard review.
                          Feature changes allowed. Retraining on schedule.

Budget 10%–25% remaining: Caution. New model deploys require a named-role sign-off.
                          Retraining permitted on schedule; its canary gate is tightened
                          (see exercise-04). On-call reviews recent incident patterns in weekly sync.

Budget 0%–10% remaining:  Ship-freeze on the surface. Only reliability work and
                          mandated security fixes merge. No new model version deploys
                          except rollback or hotfix. Retraining paused unless its sole
                          purpose is addressing the burning SLO (e.g., a drift-driven
                          quality-SLI burn is addressed by a retrain).

Budget < 0 (budget exhausted): Incident declared (see exercise-03). On-call, product
                          owner, and platform owner meet within 24h to agree a recovery
                          plan with named dates. The recovery plan includes: a cause, a
                          mitigation, and a date-bound SLO re-review. If the SLO is
                          wrong for the surface (users are not feeling the "miss"),
                          the SLO is loosened in the re-review with a dated note;
                          moving the goalposts mid-burn is explicitly flagged.
```

The policy is **binding**. A product owner who overrides it on a specific launch files a dated, named-role waiver that lists what was shipped, why, and the expected recovery path. Without the waiver, the launch does not merge.

### 2.7 Review cadence

An SLO that is not reviewed decays. The document names:

- **Quarterly review** by the surface owner and a product stakeholder. Inputs: budget consumption pattern, incident post-mortems (from `exercise-03`), user escalations, and explicit asks to tighten or loosen.
- **Trigger-based review** on any of: a user-visible incident attributed to a surface that nominally passed its SLO (the SLO was wrong), three consecutive quarters of < 10% budget consumption (the SLO is loose), or a major traffic-shape change (new segment, new geography).
- **Deprecation.** When a surface is retired, its SLO is archived with the retirement date; dead SLOs on live dashboards cost on-call attention and credibility.

### 2.8 Illustrative worked example (not a real system)

> **Illustrative feature** — an LLM-backed support-agent assistant. Internal product used by human support agents to draft responses to customer tickets. US English. Streaming response surface. This example is a composite for calibration; do not treat it as a real system or incident.

Service frame:

- Surface: `agent-assist/draft` endpoint, backend model `assist-llm-v4`, streaming completions; traffic ~ 2k rps peak, bimodal weekday pattern.
- Users who feel a miss: human support agents; downstream, the customers waiting for the agent to reply.
- Decision the error budget feeds: whether a new prompt/model/version deploys that week.
- Named owners: `team-agent-assist` (primary), `team-support-ops` (stakeholder).
- Dependencies: embedding service (`embed-serve`), retrieval index (`support-index`), LLM provider (`llm-gateway`), identity service.
- Last reviewed: 2026-11-10.

SLI menu + SLOs:

- **Availability (semantic).** SLI: `good = count(200 and non-empty completion) / valid = count(200)`. SLO: 99.9% over 30d. Error budget: 0.1% (~43 min of "empty completion" per month at nominal load).
- **Latency (time-to-first-token).** SLI: `good = count(ttft_ms <= 1500) / valid = count(request)`. SLO: 99% over 30d. Rationale: agent typing latency tolerance is 1.5 s before the agent switches tabs.
- **Latency (total completion).** SLI: `good = count(completion_ms <= 8000) / valid = count(request)`. SLO: 95% over 30d.
- **Quality (structured output).** SLI: `good = count(response matches draft schema and passes safety classifier) / valid = count(200)`. SLO: 99% over 30d.
- **Freshness (retrieval).** SLI: `good = count(retrieval index staleness <= 15min) / valid = count(request requiring retrieval)`. SLO: 99.5% over 30d.
- **Drift (prompt mix).** SLI: `good = count(request whose prompt-feature PSI vs training window <= 0.2) / valid = count(request sampled for drift)`. SLO: 98% over 30d.

Burn-rate alerts (per SLO, example for the TTFT SLO at 99% / 30d):

- Page: `burn_rate(5m) > 14.4 AND burn_rate(1h) > 14.4`
- Ticket: `burn_rate(30m) > 6 AND burn_rate(6h) > 6`
- Low-urgency: `burn_rate(2h) > 3 AND burn_rate(24h) > 3`

Monitoring contract excerpt:

- Allowed labels on `ml_request_latency_seconds`: `surface`, `model_version`, `region`, `response_class`, `is_streaming`.
- Forbidden labels: `user_id`, `ticket_id`, `request_id`.
- Sampling: drift SLI runs on a 2% stratified sample by region.
- Dashboard: `grafana://agent-assist/slo-overview` (illustrative URL — real deployments pin the real one).

Error-budget policy:

- A 0.1%-remaining state ship-freezes the surface for the week and routes a post-mortem review through `exercise-03`.
- A chronic drift-SLI burn triggers a retrain per `exercise-04`.

### 2.9 Runnable sketch: `burn_rate.py`

```python
# burn_rate.py
"""Reference burn-rate calculator and MWMB threshold derivation.

Design goal: make burn-rate thresholds reproducible from SLO parameters,
not inherited from a tutorial. A reviewer must be able to answer
'why 14.4?' from the inputs.
"""

from dataclasses import dataclass
from math import ceil


@dataclass(frozen=True)
class SLO:
    target: float               # e.g., 0.999 for 99.9%
    window_seconds: int         # e.g., 30 * 24 * 3600 for 30 days

    @property
    def error_budget(self) -> float:
        return 1.0 - self.target


@dataclass(frozen=True)
class BurnAlert:
    """One burn-rate alert tier.

    `fraction_of_budget` is the budget share consumed within `long_window_seconds`
    that justifies firing the alert. The MWMB threshold is derived from it.
    """
    short_window_seconds: int
    long_window_seconds: int
    fraction_of_budget: float
    severity: str  # 'page', 'ticket', 'email'

    def burn_rate_threshold(self, slo: SLO) -> float:
        return self.fraction_of_budget * (slo.window_seconds / self.long_window_seconds)


def default_mwmb_policy(slo: SLO) -> list[BurnAlert]:
    return [
        BurnAlert(short_window_seconds=5 * 60,
                  long_window_seconds=60 * 60,
                  fraction_of_budget=0.02,
                  severity='page'),
        BurnAlert(short_window_seconds=30 * 60,
                  long_window_seconds=6 * 60 * 60,
                  fraction_of_budget=0.05,
                  severity='ticket'),
        BurnAlert(short_window_seconds=2 * 60 * 60,
                  long_window_seconds=24 * 60 * 60,
                  fraction_of_budget=0.10,
                  severity='email'),
    ]


def observed_burn_rate(good: int, valid: int, slo: SLO) -> float:
    """Burn rate over the observation window, relative to the SLO target.

    A burn_rate of 1.0 means the surface is burning the error budget at exactly
    the rate the target allows. A burn_rate of 10 means 10x the allowed rate.
    """
    if valid == 0:
        return 0.0
    observed_bad_fraction = 1.0 - (good / valid)
    allowed_bad_fraction = slo.error_budget
    if allowed_bad_fraction <= 0:
        raise ValueError("SLO target of 100% has no non-zero error budget")
    return observed_bad_fraction / allowed_bad_fraction


def minutes_until_budget_exhausted(burn_rate: float, slo: SLO) -> float:
    if burn_rate <= 0:
        return float('inf')
    return (slo.window_seconds / 60.0) / burn_rate


if __name__ == "__main__":
    slo = SLO(target=0.999, window_seconds=30 * 24 * 3600)
    print(f"error budget: {slo.error_budget:.4%}")
    for alert in default_mwmb_policy(slo):
        br = alert.burn_rate_threshold(slo)
        print(f"{alert.severity:>6}: burn_rate > {br:.2f} "
              f"over short={alert.short_window_seconds // 60}m "
              f"long={alert.long_window_seconds // 60}m")
    # observed example
    br = observed_burn_rate(good=9_800, valid=10_000, slo=slo)
    print(f"observed burn rate: {br:.2f}")
    print(f"budget exhausted in: {minutes_until_budget_exhausted(br, slo):.1f} min")
```

The script is deliberately small. The discipline the SLO document enforces is that every alert threshold is reproducible from the SLO's own parameters — target, window, and the fraction-of-budget the team decided justifies a page.

## 3. Validation steps

Walk the SLO document through this checklist:

1. **Each SLI has a numerator and a denominator written as code-ready sets**, with the measurement window named.
2. **"Good" and "valid" are defined in words**, not inferred — a reviewer can tell a 404 from a user-caused miss, a streaming TTFT from a completion latency, a non-empty-list quality from a 200 response.
3. **Each SLO names a target, a window, and an error budget in absolute terms** (minutes of unavailability per month, or fraction of requests allowed to breach latency).
4. **Target was argued from user tolerance**, not accepted from a template. The document records the "why 99.9%?" line.
5. **At least one SLI is model-specific** (freshness, drift, or quality) rather than borrowed from generic service monitoring.
6. **Burn-rate alerts are MWMB** (two windows per tier), with thresholds derivable from the SLO parameters.
7. **Monitoring contract names the metric prefix, allowed labels, forbidden labels, sampling policy, and dashboard URL** (or deliberately marks the URL as illustrative if the surface does not exist yet).
8. **Error-budget policy is binding**, not advisory. There is a ship-freeze threshold with a named override procedure.
9. **Dependencies on upstream surfaces are listed**, and the SLO is not tighter than the composition of their targets.
10. **Review cadence is named** and a last-reviewed date is within it.
11. **An LLM/safety surface has at least one safety SLI** (schema validity, prompt-injection-resistance, refusal-rate on a probe set) aligned with OWASP ML Top 10 and MITRE ATLAS tactics.
12. **NIST AI RMF MEASURE and MANAGE functions are cited** in the rationale for SLOs and the error-budget policy.

Static checks to drop into CI on `slo.md`:

```bash
grep -ciE "sli|slo|error budget|burn rate"              slo.md   # the vocabulary is present
grep -ciE "target|window|good/|valid/"                   slo.md   # structure is present
grep -ciE "ship.freeze|policy"                           slo.md   # policy is binding
grep -ciE "page|ticket|alert"                            slo.md   # alerting is specified
grep -ciE "freshness|drift|quality"                      slo.md   # ML-specific SLIs present
grep -ciE "nist ai rmf|owasp|mitre atlas"                slo.md   # standards cited
grep -ciE "last reviewed"                                slo.md   # freshness
```

For `burn_rate.py`, a smoke test instantiates the SLO in §2.9, prints the three MWMB thresholds, and compares them to stored reference values. A design where the thresholds change between runs is a bug in the calculator.

## 4. Rubric / review checklist

- [ ] Service frame names surface, users who feel a miss, decision fed, named owners (roles), dependencies, last-reviewed date.
- [ ] SLI menu covers at least two of the ML-specific families (quality, freshness, drift) in addition to availability and latency.
- [ ] Every SLI is expressed as `good / valid` with the two sets defined in words.
- [ ] Every SLO has a numeric target, a window, and an error budget stated in absolute user-facing terms.
- [ ] Target is argued from user behaviour or business constraint, not inherited from a default.
- [ ] Burn-rate alerts are MWMB with at least two tiers (page and ticket); thresholds are derivable from the SLO parameters.
- [ ] The alert policy records "fraction of budget consumed per window" rationale per tier, not just the numeric threshold.
- [ ] Monitoring contract lists metric names, allowed labels, forbidden labels, sampling policy, and dashboard URL.
- [ ] Error-budget policy has a ship-freeze threshold and a named-role override with logging requirement.
- [ ] Dependencies on upstream surfaces are listed with their SLOs or the acknowledgement that the composition caps this SLO.
- [ ] Safety SLIs reflect OWASP ML Top 10 and MITRE ATLAS tactics where the surface has a safety/abuse surface.
- [ ] NIST AI RMF MEASURE cited for measurement-validity framing; MANAGE cited for the error-budget-as-risk-control framing.
- [ ] Review cadence named; last-reviewed date within cadence.
- [ ] Cross-references to `exercise-02` (cost SLOs and quota enforcement), `exercise-03` (incident response on budget exhaustion), and `exercise-04` (retraining as a deploy under the error-budget policy).

Review-weighted points for grading:

- Service frame and owner clarity: 10%
- SLI menu coverage (availability, latency, quality, freshness, drift) with precise definitions: 25%
- SLO targets with user-grounded rationale and error-budget sizing: 20%
- Burn-rate alert policy (MWMB, derivable thresholds, severity tiers): 20%
- Monitoring contract and dashboarding: 10%
- Error-budget policy (binding ship-freeze, named override, cross-reference to incident response): 10%
- Standards alignment (NIST AI RMF MEASURE / MANAGE, OWASP ML Top 10 / MITRE ATLAS for safety SLIs) and hygiene: 5%

## 5. Common mistakes

- **One SLO for the whole system.** Availability is not latency is not quality. A single SLO averages away the signal; each has its own user-perceived failure mode and its own runbook.
- **Target picked by vibes.** "99.9% because the last team used 99.9%" is not an argument. The target is picked from user behaviour, upstream dependencies, and the engineering capacity it implies to meet.
- **No error budget in absolute terms.** "99.95% availability" is a target; "21.56 minutes of downtime per month at current traffic" is an error budget. The document must have both.
- **Alerting on the raw SLI.** The raw SLI over 30 days changes slowly; by the time it crosses the target, the incident has already run for days. Burn-rate alerts are the fix.
- **Single-window burn-rate alert.** A single short window pages on 30-second blips; a single long window waits hours to notice a hard outage. MWMB is the resolution.
- **No ML-specific SLI.** Borrowing a generic HTTP availability SLO and calling it done ignores the ways ML surfaces fail: stale features, drifted inputs, degraded output quality at nominal availability and latency.
- **Freshness SLO without a freshness pipeline.** An SLO that says "features are < 15 min old" is unmeasurable unless the feature pipeline timestamps emission and the metric is wired up. The SLO document must point at the mechanism.
- **Quality SLI defined as "offline accuracy."** Offline accuracy is a release artefact; it is not an online SLI. The quality SLI must be computable from live traffic (or a shadow / canary thereof).
- **Cardinality-explosive labels.** `user_id` on an SLI metric blows the metrics backend before the first alert fires. The monitoring contract lists allowed labels for a reason.
- **Ignoring upstream SLOs.** If the LLM provider's own SLO is 99.9% and your surface claims 99.99%, your document is either wrong, over-architected (you have fallbacks that absorb provider outages), or implicitly promising something you cannot keep.
- **Error-budget "policy" that is advisory.** A policy that reads "we will take the budget seriously" ships code into exhausted budgets every quarter. The policy must bind merges and launches.
- **SLOs that never change.** An SLO that passes every quarter at 15% consumption is loose; one that breaches every quarter is tight or the surface is unhealthy. The review cadence is what makes the SLO responsive.
- **Dashboards without the target line.** An SLI panel without the SLO line and the error-budget-consumption panel invites the viewer to guess. Dashboards are the artefact that makes the SLO legible on a Monday morning.
- **Safety SLIs absent on generative surfaces.** OWASP ML Top 10 and MITRE ATLAS name the tactics; a generative surface that only measures availability and latency hides the failure mode users actually care about.
- **No ship-freeze teeth.** A budget-exhaustion policy without a merge-blocking gate is a suggestion. The error budget's job is to make the cost of unreliability show up in the shipping velocity, where it can be weighed.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MEASURE function grounds the ongoing measurement of AI system trustworthiness characteristics — accuracy, reliability, safety — that an SLO expresses as a user-facing contract. MANAGE function grounds the error-budget-as-risk-control framing: when the budget is exhausted, risk mitigation becomes a constraint on launches, not a conversation.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source for the adversarial-input and output-integrity scenarios that inform safety SLIs on generative and classification surfaces.
- MITRE ATLAS — <https://atlas.mitre.org/>. Source for named adversarial tactics (evasion, model extraction, prompt injection) that a safety SLI probe set can be built against.

Local context (authoritative for the SLO document's shape and for the cross-references):

- Paired lesson: `lessons/mod-307-ml-reliability-slos/exercises/exercise-01-ml-slis-and-slos.md`.
- Module README: `modules/mod-307-ml-reliability-slos/README.md`.
- Related exercise (cost budgets as an SLI family and quota enforcement when cost burns the budget): `modules/mod-307-ml-reliability-slos/exercise-02-cost-budgets-and-quotas/SOLUTION.md`.
- Related exercise (incident response drill — what happens when the SLO breaches and the error-budget policy fires): `modules/mod-307-ml-reliability-slos/exercise-03-ml-incident-response-drill/SOLUTION.md`.
- Related exercise (retraining as a deploy, which consumes error budget when its canary gate trips): `modules/mod-307-ml-reliability-slos/exercise-04-retraining-as-a-deploy/SOLUTION.md`.
- Related exercise (offline harness whose slice-level thresholds feed the online quality SLI): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
- Related exercise (shadow and canary deploy design, source of the "canary gate" referenced in the error-budget policy): `modules/mod-305-advanced-evaluation/exercise-02-shadow-and-canary-deploy-design/SOLUTION.md`.
