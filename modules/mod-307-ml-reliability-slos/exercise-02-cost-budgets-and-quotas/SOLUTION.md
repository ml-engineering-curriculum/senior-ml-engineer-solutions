# exercise-02 — Cost Budgets And Quotas (Reference Solution)

Paired exercise: `lessons/mod-307-ml-reliability-slos/exercises/exercise-02-cost-budgets-and-quotas.md`.

## 1. Solution overview

This exercise produces `cost-policy.md` plus a small runnable artefact (`quota.py`) — a written cost-budget policy and the enforcement code that turns the policy into *admission control*, not a dashboard. The artefact has six parts:

1. A **cost frame** that pins down what the policy covers, what the unit of cost is for this surface, who pays, and what decision the budget feeds.
2. A **cost-SLI menu** — the measurable signals that convert spend into `good / valid` ratios: cost per request, cost per user per day, cost per tenant per month, token-usage share by model and route. Cost gets the same SLI/SLO discipline as latency and availability (see `exercise-01`).
3. A **budget hierarchy and allocation table** — organization → surface → tenant → caller, with hard caps and soft caps, and the share of each parent budget each child is entitled to.
4. A **quota-enforcement contract** — the admission-control points (edge, app, model gateway, batch scheduler), the algorithm (token bucket, leaky bucket, concurrency semaphore, cost-aware rate limiter), the response contract (`429 Too Many Requests` with `Retry-After`, structured error body, cost-attribution header), and the fail-closed vs. fail-open stance per path.
6. A **cost-driven degradation playbook** — the ordered list of what the surface sacrifices as a budget nears exhaustion: cache harder, downgrade model tier, truncate context, deny non-essential traffic, return cached fallback. Degradation is pre-registered, not improvised.
7. An **attribution and reporting contract** — the metrics, labels, logs, and traces that let an owner answer "who spent what, when, and why," on a cadence that keeps the budget binding.

The reference solution below shows the structure, works a labeled illustrative example through it, and gives the grading rubric.

The exercise exists because a cost that is only *reported* is a cost that only shocks. A dashboard that turns red at month-end is not a budget; it is a memorial. A cost budget becomes a safety property when the system enforces it in-line: the next token over the limit is rejected with a clear contract, the next tenant over its share is throttled with an auditable trace, the next retrain over its ceiling is paused at the scheduler. NIST AI RMF MANAGE grounds budget enforcement as an active risk control. OWASP ML Top 10 scenarios around model-denial-of-service (expensive-input abuse, context-stuffing) and model-theft via unmetered access rely on absent quota enforcement; the policy below is what defeats them. MITRE ATLAS covers adjacent tactics (ML model inference API abuse, resource exhaustion).

## 2. Worked answer / implementation

### 2.1 Cost frame

Six lines, in this order:

- **Surface covered.** The service, endpoint, model, or batch job this policy binds. Pinned to a name. "The LLM" is not a surface; "`agent-assist/draft` served by `assist-llm-v4` via `llm-gateway`" is.
- **Unit of cost.** The measurable thing a dollar sign eventually attaches to: tokens, GPU-seconds, embedding calls, storage-GB-days, egress-GB, provider-side billed units. Cost math is done in the unit the provider meters, not in dollars; dollar conversion is applied at the reporting edge after rates change.
- **Who pays.** The cost center, product line, or tenant whose budget this surface draws from. A surface without a payer is a surface no one will defend when the bill arrives.
- **Decision the budget feeds.** A merge gate, a scheduler pause, a per-tenant feature toggle, a launch block. The decision is what makes the budget load-bearing.
- **Named owners.** A primary owner (the platform team that operates the surface) and a secondary stakeholder (the finance / cost partner). Roles, not people.
- **Last reviewed.** A date, within the policy's declared review cadence (default: monthly review of consumption; quarterly review of allocations).

### 2.2 Cost-SLI menu

A cost SLI is **cost consumed** over a window, compared against an **allowed cost**. Like reliability SLIs, cost SLIs are `good / valid` fractions; "good" is "request served within its per-request cost budget" or "window ended within the aggregate cost budget."

**C1. Per-request cost SLI.** `good = count(cost_units_per_request <= per_request_cap) / valid = count(request)`. Catches expensive-input abuse (very long prompts, pathological inputs) at the request level.

**C2. Per-tenant daily cost SLI.** `good = count(tenant-days whose consumption <= daily_tenant_cap) / valid = count(tenant-days)`. Catches runaway tenants and informs pricing / packaging.

**C3. Surface monthly cost SLI.** `good = 1 if month-to-date consumption <= monthly_surface_cap else 0; valid = 1 per month`. The aggregate SLO: the surface stayed inside its envelope this month.

**C4. Attribution completeness SLI.** `good = count(request with full cost-attribution labels) / valid = count(request)`. A cost you cannot attribute is a cost you cannot bound; this meta-SLI catches logging-pipeline gaps before they become mystery spend.

**C5. Provider-side rate-limit SLI.** `good = count(request not provider-throttled) / valid = count(upstream call)`. If the upstream LLM provider throttles the surface, the internal quota is irrelevant — the user is already losing.

Pick 3–5 cost SLIs for one surface. The common minimum for a generative surface is C1 + C3 + C4; add C2 if the surface is multi-tenant; add C5 if the surface depends on an external metered provider.

**Cost SLOs** attach a target and window to each SLI, the same way reliability SLOs do (`exercise-01`). A typical set:

- C1 at 99% over 30d: at most 1% of requests exceed the per-request cost cap.
- C3 at 100% per month: the surface never exceeds its monthly cap. A 100% target implies the enforcement is hard, not statistical; the "error budget" is paid in degraded service, not in dollars.
- C4 at 99.9% over 30d: attribution is near-complete; the budget on missing attribution is tight because an unattributed cost is a cost without an owner.

### 2.3 Budget hierarchy and allocation

Budgets nest. A surface's budget is a slice of a product line's budget; a tenant's budget is a slice of the surface's. The allocation table records who-gets-what and leaves headroom.

```
L0  Organization monthly ML spend cap:                   100% (anchor)
L1  Product line "Support Agent Assist":                   X% of L0
      L2  Surface "agent-assist/draft":                    Y% of L1
            L3a  Tier "pro" tenants:                       Z1% of L2
            L3b  Tier "basic" tenants:                     Z2% of L2
            L3c  Internal staff / eval traffic:            Z3% of L2
            L3d  Headroom (unallocated, for spikes):       Z4% of L2  (≥ 10%)
      L2  Surface "agent-assist/summarize":                W% of L1
```

Rules on the table:

- **Headroom ≥ 10%.** A budget allocated to 100% has no room for a legitimate traffic spike; the first spike then looks like an abuse pattern because enforcement cannot distinguish them.
- **Soft cap < hard cap.** The soft cap triggers degradation (§2.5); the hard cap triggers refusal. Soft < hard is often 80% of hard, so degradation has time to bend the curve before refusal fires.
- **Per-tenant caps are per tier**, not per tenant. A thousand tenant-specific numbers are unmaintainable; three tier numbers plus a one-off exceptions list is.
- **Exceptions list is dated and named.** A tenant granted a temporary over-cap allowance has a dated entry with the role that approved, the reason, and the expiry. Expired entries fall back to tier default without a new review.
- **Allocation reviewed quarterly.** Business changes (new tier, new product) shift the table; a table from 18 months ago binding enforcement today is a landmine.

### 2.4 Quota-enforcement contract

A quota becomes enforcement when a request is actually rejected. The policy names the enforcement points, the algorithms, and the response contract.

**Enforcement points.** In order of first-touch:

1. **Edge / API gateway** — cheapest refusal. Reject requests violating per-IP / per-API-key rate limits before any model work happens. Required for defense against inference-API abuse (MITRE ATLAS tactics include unmetered inference access and model-theft via replay).
2. **Application layer** — per-tenant, per-feature quota. Resolves the authenticated tenant, looks up their tier, checks the current consumption against the per-tenant cap. Returns `429` with `Retry-After` when exceeded.
3. **Model gateway (or LLM router)** — per-model token-bucket quota. Even a well-behaved application can swamp a single model instance; a model-gateway-level quota prevents one surface's burst from starving another's.
4. **Batch scheduler** — for offline jobs (retraining, re-embedding, batch inference), the scheduler holds job admission against a job-class budget. A retraining run that would blow the month's budget does not start, regardless of the schedule.
5. **Provider-side budget** — the upstream LLM or compute provider's own budget controls. The policy records the provider's per-key spend caps so an internal quota failure does not produce a surprise bill.

Enforcement is layered because any single layer fails. An edge rate limiter cannot see which tenant is calling; an application-layer tenant check cannot see pre-authentication abuse; a model-gateway quota cannot see batch workloads.

**Algorithms.**

- **Token bucket** for per-key, per-tenant, per-model rate limiting. Parameters: refill rate (requests or cost units per second), burst capacity. Named rationale per parameter: "refill = steady-state budget / seconds in window," "burst = tolerated spike size per tenant."
- **Leaky bucket** for smoothing: upstream must not see traffic bursts beyond the bucket drain rate. Same math, different framing.
- **Concurrency semaphore** for GPU-bound inference: limit in-flight requests to the number of tolerated concurrent model invocations. Prevents queue-driven latency collapse when traffic exceeds serving capacity.
- **Cost-aware rate limiter.** Each request charges its *predicted* cost against the bucket before admission; actual cost reconciles after the response. Needed when per-request costs vary by orders of magnitude (short prompts vs. long prompts, cheap models vs. expensive models). Prevents the pathological "many cheap requests pass; one huge request blows the month."
- **Admission policy** on batch jobs: schedule only if `job.estimated_cost + current_month_spend <= monthly_cap * safety_factor`. Safety factor < 1 (commonly 0.9) leaves room for miss-estimates.

**Response contract.**

- **On exceed (hard):** `HTTP 429 Too Many Requests`, `Retry-After: <seconds>` based on the bucket refill time, response body `{"error":"quota_exceeded","scope":"tenant|surface|org","reset_at":"<ISO-8601>","retry_after_seconds":N}`. The structured body is required so client SDKs can back off correctly.
- **On exceed (soft):** `HTTP 200` with a degraded payload (see §2.5) *and* a response header or metadata field `X-Served-Degraded: 1; reason=soft-cap-80pct`. A silent degradation is a lie to the client; a degradation with a stamp is a contract.
- **Attribution header.** Every response carries `X-Cost-Units: <n>; X-Cost-Center: <id>` (or an equivalent metadata field for non-HTTP protocols). Cost without attribution is cost without accountability.
- **Logging.** Every rejected request writes a structured log with tenant, scope, bucket state, and cause. Rejected-request volume is itself monitored; a surface rejecting 10% of traffic is in an incident even if everything that did get served looks healthy.

**Fail-closed vs. fail-open.**

Per path, the policy names what happens when the quota-enforcement subsystem is itself down:

- Authenticated, billed API: **fail closed**. If the quota store is unreachable, reject. Open quotas on a down enforcement path invite abuse that only shows up at the bill.
- Internal non-billed, best-effort features (e.g., async notification generation): **fail open** with a logged "quota-check-bypassed" marker. The marker shows up in the attribution gap SLI (C4) so the outage is visible.

Fail-stance is explicitly named per surface; "we'll figure it out" is the pattern that lets a quota outage double a month's bill.

### 2.5 Cost-driven degradation playbook

When a budget nears exhaustion, the surface degrades gracefully rather than refusing. Degradation is pre-registered, ordered, and reversible.

```
Budget at 60% consumption, time-pacing on track:
    No degradation. Baseline service.

Budget at 80% consumption, pacing on track (soft cap):
    - Aggressive response caching: raise cache TTL for cacheable responses.
    - Increase batching: group small requests into larger model calls where latency allows.
    - Prefer cheaper model tier for low-value routes (per route: "which class of user does this route serve").

Budget at 90% consumption:
    - Downgrade the default model tier for all non-essential routes.
    - Truncate context windows (longer-context calls are the usual spend drivers).
    - Return cached-fallback responses for best-effort traffic.

Budget at 95% consumption:
    - Essential-only mode: non-paying / internal traffic gets 429s;
      paying tenants served at the downgraded tier.
    - Batch jobs paused at the scheduler.

Budget at 100% (hard cap):
    - All non-essential traffic rejected with 429.
    - Essential traffic served at the lowest-cost tier that still meets
      the surface's reliability SLOs from exercise-01.
    - Incident declared (exercise-03) if the hard-cap hit was unplanned.
```

Each step names: *what changes, for whom, reversible by what event*. "Revert when consumption drops below 85%" is a reversal rule; "revert when the engineer feels better" is not.

**Per-tenant degradation.** Within a surface-level cap, individual tenants hit their own tier caps independently. A tier-1 tenant who exhausts their daily budget is degraded even when the surface is at 20% overall. Cross-tenant fairness is the point of per-tenant caps; without them, one tenant burns the shared budget and every other tenant pays with degraded service.

### 2.6 Attribution and reporting contract

Attribution is the discipline that makes cost actionable. The policy names:

- **Request-level attribution.** Every request is tagged with: `tenant_id` (or anonymous-session for pre-auth), `cost_center`, `surface`, `model_version`, `route`, `is_streaming`, `is_batch`, `request_class`. These labels are allowed on the cost metric (`ml_cost_units_total`); other labels are forbidden to keep cardinality bounded.
- **Metric emission.** `ml_cost_units_total{...}` counter at request end, with the request's measured cost in the surface's unit. A `ml_cost_units_estimated{...}` counter at request start supports pre-admission accounting.
- **Trace propagation.** The cost center and tenant travel through the trace context so downstream calls inherit attribution. A call the surface makes to an upstream LLM gateway charges the same cost center the inbound request was attributed to.
- **Dashboards.** One per surface, with: month-to-date consumption vs. budget, trailing 7-day burn rate, top N tenants by consumption, per-route per-model breakdown, attribution-completeness SLI panel.
- **Report cadence.** Daily: top-N callers and anomaly alerts. Weekly: trend review by the surface owner. Monthly: formal variance-to-budget review by owner + finance stakeholder; outputs feed the quarterly allocation review (§2.3).
- **Alerts.** Burn-rate alerts on the monthly cost SLI mirror the MWMB pattern from `exercise-01`:
    - Page: trending to exceed the monthly cap within 48h at current burn.
    - Ticket: trending to exceed within the remaining calendar days of the month.
    - Low-urgency: an individual tenant trending above their tier cap by ≥ 50%.

### 2.7 Illustrative worked example (not a real system)

> **Illustrative feature** — the same `agent-assist/draft` LLM support-draft surface from `exercise-01`'s example. Composite for calibration; do not treat as a real system or incident.

Cost frame:

- Surface: `agent-assist/draft`, backend `assist-llm-v4` via `llm-gateway`, streaming.
- Unit of cost: provider-metered tokens (input + output, weighted per the provider's price sheet; the policy keeps weights in a versioned file, not inline in code).
- Who pays: `cost-center:product-support-ops`.
- Decision fed: scheduler pauses retraining above 90% consumption; ship-freeze above 100%.
- Named owners: `team-agent-assist` (primary), `team-finance-ops` (stakeholder).
- Last reviewed: 2026-11-10.

Cost SLIs and SLOs:

- **C1 per-request:** `good = count(cost_tokens_per_request <= 4000) / valid = count(request)`. SLO: 99% over 30d.
- **C2 per-tenant daily (pro tier):** daily cap 500k tokens/tenant. SLO: 99% of tenant-days in-bounds over 30d.
- **C3 surface monthly:** hard cap `H` tokens/month (value redacted; a real policy pins the number). SLO: 100% per month — the surface does not blow the cap.
- **C4 attribution completeness:** 99.9% over 30d.
- **C5 provider throttling:** 99% over 30d.

Allocation (illustrative shares; real policy pins the numbers):

- L2 `agent-assist/draft`: 70% of L1 product-line budget.
    - L3a pro tenants: 60% of L2.
    - L3b basic tenants: 15% of L2.
    - L3c internal/eval: 10% of L2.
    - L3d headroom: 15% of L2.

Enforcement:

- Edge: per-API-key rate limit (requests per second). Bucket refill `R` req/s, burst `B`.
- App: per-tenant cost-aware bucket; estimated cost charged on admission, reconciled on response.
- Model gateway: per-model concurrency semaphore `C`.
- Batch scheduler: retraining admission against monthly cap.

Degradation at 80% monthly consumption: downgrade long-context requests from a premium to a mid-tier model for internal traffic; truncate context from 32k → 16k tokens on non-paying traffic.

### 2.8 Runnable sketch: `quota.py`

```python
# quota.py
"""Reference cost-aware token-bucket admission controller.

Design goal: pre-admission cost charging. Each incoming request declares an
*estimated* cost; the bucket admits only if the estimate fits. Actual cost
reconciles after the response, charging the delta or refunding the overshoot.

This is deliberately standalone and in-memory. A real deployment hooks the
bucket state into a shared store (Redis, DynamoDB, a dedicated quota service)
and attaches it to the identity layer.
"""

from dataclasses import dataclass, field
from threading import Lock
from typing import Optional


@dataclass
class TokenBucket:
    """Cost-unit token bucket with refill rate."""
    capacity: float               # bucket size in cost units
    refill_rate_per_second: float # cost units added per second
    tokens: float = field(init=False)
    last_refill_ts: float = field(init=False)
    _lock: Lock = field(default_factory=Lock, init=False, repr=False)

    def __post_init__(self) -> None:
        self.tokens = self.capacity
        # Caller passes clock explicitly in `refill`; constructor does not read wall-clock.
        self.last_refill_ts = 0.0

    def refill(self, now_ts: float) -> None:
        with self._lock:
            elapsed = max(0.0, now_ts - self.last_refill_ts)
            added = elapsed * self.refill_rate_per_second
            self.tokens = min(self.capacity, self.tokens + added)
            self.last_refill_ts = now_ts

    def try_charge_estimate(self, estimated_cost: float, now_ts: float) -> bool:
        self.refill(now_ts)
        with self._lock:
            if self.tokens >= estimated_cost:
                self.tokens -= estimated_cost
                return True
            return False

    def reconcile(self, estimated_cost: float, actual_cost: float) -> None:
        delta = actual_cost - estimated_cost
        with self._lock:
            # Charge extra (delta > 0 drains bucket further, possibly negative)
            # Refund overshoot (delta < 0 restores tokens, capped at capacity).
            self.tokens = min(self.capacity, self.tokens - delta)


@dataclass(frozen=True)
class AdmissionResult:
    admitted: bool
    reason: Optional[str]
    retry_after_seconds: float


class CostAwareLimiter:
    """Per-key cost-aware limiter.

    One bucket per (scope, key) pair. Scope is 'tenant' | 'surface' | 'route'.
    A request admits only if every applicable bucket has room; the first
    bucket that rejects names the scope the caller must back off against.
    """

    def __init__(self) -> None:
        self._buckets: dict[tuple[str, str], TokenBucket] = {}
        self._lock = Lock()

    def install_bucket(self, scope: str, key: str, capacity: float, refill_rate_per_second: float) -> None:
        with self._lock:
            self._buckets[(scope, key)] = TokenBucket(capacity=capacity, refill_rate_per_second=refill_rate_per_second)

    def admit(self, scopes: dict[str, str], estimated_cost: float, now_ts: float) -> AdmissionResult:
        """Try to charge `estimated_cost` against every applicable bucket.

        If any bucket rejects, roll back prior charges (we are single-threaded in
        this sketch) and return the rejection reason.
        """
        charged: list[tuple[str, str]] = []
        for scope, key in scopes.items():
            bucket = self._buckets.get((scope, key))
            if bucket is None:
                continue
            if not bucket.try_charge_estimate(estimated_cost, now_ts):
                # Rollback.
                for (s_rb, k_rb) in charged:
                    self._buckets[(s_rb, k_rb)].reconcile(estimated_cost, 0.0)
                retry = estimated_cost / max(bucket.refill_rate_per_second, 1e-9)
                return AdmissionResult(admitted=False, reason=f"{scope}_quota_exceeded", retry_after_seconds=retry)
            charged.append((scope, key))
        return AdmissionResult(admitted=True, reason=None, retry_after_seconds=0.0)

    def reconcile(self, scopes: dict[str, str], estimated_cost: float, actual_cost: float) -> None:
        for scope, key in scopes.items():
            bucket = self._buckets.get((scope, key))
            if bucket is not None:
                bucket.reconcile(estimated_cost, actual_cost)


if __name__ == "__main__":
    # Illustrative wiring, deterministic clock.
    limiter = CostAwareLimiter()
    limiter.install_bucket("tenant", "t-pro-123", capacity=500_000, refill_rate_per_second=500_000 / 86_400)
    limiter.install_bucket("surface", "agent-assist/draft", capacity=5_000_000, refill_rate_per_second=5_000_000 / 86_400)

    t = 0.0  # test clock
    scopes = {"tenant": "t-pro-123", "surface": "agent-assist/draft"}
    # Cheap request.
    r1 = limiter.admit(scopes, estimated_cost=1_500, now_ts=t)
    print(f"r1 admit={r1.admitted} reason={r1.reason}")
    limiter.reconcile(scopes, estimated_cost=1_500, actual_cost=1_480)
    # Expensive request near tenant cap.
    r2 = limiter.admit(scopes, estimated_cost=600_000, now_ts=t + 1)
    print(f"r2 admit={r2.admitted} reason={r2.reason} retry_after={r2.retry_after_seconds:.0f}s")
```

The sketch is deliberately small. The discipline the policy enforces is that enforcement is a function of named buckets and named scopes, not a sprinkle of `if` statements across controllers.

## 3. Validation steps

Walk the cost policy through this checklist:

1. **Cost frame names the surface, unit of cost, payer, decision fed, owners, and last-reviewed date.**
2. **Cost SLIs are `good / valid` fractions**, not raw spend numbers.
3. **Monthly cost SLO is at 100% per month** (hard cap is not statistical) with degradation absorbing the "error budget."
4. **Budget hierarchy has at least three levels** (org → surface → tenant/route) with explicit headroom ≥ 10%.
5. **Soft cap < hard cap** named per budget level; the soft cap triggers degradation, the hard cap triggers refusal.
6. **Enforcement points are named** (edge, app, model gateway, batch scheduler) with the algorithm per point.
7. **At least one enforcement algorithm is cost-aware** (not just request-count rate limiting), because per-request costs vary by orders of magnitude.
8. **Response contract names the HTTP status, the Retry-After derivation, the structured body, and the attribution header.**
9. **Fail-closed vs. fail-open stance** is named per path.
10. **Degradation playbook is ordered**, pre-registered, and reversible.
11. **Per-tenant caps exist** on multi-tenant surfaces, with tier-level numbers and a dated exceptions list.
12. **Attribution is enforced at the metric level** with named allowed/forbidden labels; cardinality is bounded.
13. **Burn-rate alerting on the monthly cost SLI** mirrors the MWMB structure from `exercise-01`.
14. **The policy cites OWASP ML Top 10 model-denial-of-service** and MITRE ATLAS inference-abuse tactics in the enforcement rationale.

Static checks to drop into CI on `cost-policy.md`:

```bash
grep -ciE "hard cap|soft cap|headroom"            cost-policy.md
grep -ciE "token bucket|leaky bucket|semaphore"   cost-policy.md
grep -ciE "429|retry.after|attribution"           cost-policy.md
grep -ciE "fail.closed|fail.open"                 cost-policy.md
grep -ciE "degradation|downgrade|fallback"        cost-policy.md
grep -ciE "nist ai rmf|owasp|mitre atlas"         cost-policy.md
grep -ciE "last reviewed"                         cost-policy.md
```

For `quota.py`, a smoke test drives the limiter through: a cheap request (admit), many cheap requests up to the tenant cap (first N admit; N+1 rejects with `tenant_quota_exceeded`), an expensive request near the cap (rejects before it starves other callers), a `reconcile` that returns tokens when actual < estimate.

## 4. Rubric / review checklist

- [ ] Cost frame names the surface, unit of cost, payer, decision fed, owners (roles), and last-reviewed date.
- [ ] Cost SLIs cover per-request, aggregate (monthly), and attribution-completeness.
- [ ] Monthly cost SLO is at 100% per month with the "error budget" paid in degraded service, not dollar overage.
- [ ] Budget hierarchy has ≥ 3 levels with explicit per-tier shares and ≥ 10% headroom.
- [ ] Soft cap and hard cap distinguished per budget level with a named reversal rule.
- [ ] Enforcement points are named: edge, application, model gateway, batch scheduler, provider-side.
- [ ] Algorithms chosen per point are correct for the workload (token bucket for rate, semaphore for concurrency, cost-aware limiter for variable-cost requests).
- [ ] Response contract: HTTP status, Retry-After derivation, structured error body, X-Cost-Units / X-Cost-Center attribution header.
- [ ] Fail-closed vs. fail-open stance named per path with rationale.
- [ ] Degradation playbook is ordered (cache harder → downgrade tier → truncate context → essential-only → refuse), each step reversible with a named condition.
- [ ] Per-tenant caps exist on multi-tenant surfaces; tier numbers are named; dated exceptions list is referenced.
- [ ] Attribution: allowed/forbidden labels on `ml_cost_units_total`; trace propagation of cost-center; dashboard URL.
- [ ] Burn-rate alerts mirror MWMB structure; pages, tickets, low-urgency levels are named.
- [ ] OWASP ML Top 10 model-denial-of-service and MITRE ATLAS inference-abuse tactics cited in the enforcement rationale.
- [ ] NIST AI RMF MANAGE cited for budget-enforcement-as-risk-control framing.
- [ ] Review cadence (monthly consumption, quarterly allocations) named; last-reviewed date within cadence.
- [ ] Cross-references to `exercise-01` (reliability SLOs that bound degradation), `exercise-03` (incident response on hard-cap breach), `exercise-04` (retraining admission against the budget).

Review-weighted points for grading:

- Cost frame and payer clarity: 10%
- Cost-SLI menu with per-request, aggregate, and attribution coverage: 15%
- Budget hierarchy with headroom and soft/hard distinction: 15%
- Enforcement contract — points, algorithms, response shape, fail-stance: 25%
- Degradation playbook: ordered, pre-registered, reversible: 15%
- Attribution and reporting (labels, traces, dashboards, alert tiers): 10%
- Standards alignment (OWASP ML Top 10, MITRE ATLAS, NIST AI RMF MANAGE) and hygiene: 10%

## 5. Common mistakes

- **Dashboard-only budget.** A red number on a dashboard at month-end is not a budget; it is a memorial. Enforcement must reject or degrade in-line.
- **Count-based rate limiting on variable-cost calls.** For a surface where one request can cost 10× another, "100 requests per second" is a target that either blocks legitimate low-cost traffic or lets one pathological request blow the day. Cost-aware limiters charge estimated cost.
- **No pre-admission charging.** Charging only *after* the response means a single pathological request can commit the surface to a cost the budget cannot absorb. Admission uses the estimated cost; reconciliation corrects it.
- **No headroom.** Allocating 100% of a budget to callers leaves nothing for legitimate spikes; the first spike then trips enforcement as if it were abuse. Headroom ≥ 10% is a feature, not a loss.
- **Soft cap = hard cap.** Without a soft cap, the surface goes from "normal" to "refused" with no graceful degradation; users experience a cliff.
- **Silent degradation.** Returning a cheaper-model response without a header stamp lies to the client about what they got. `X-Served-Degraded` makes the trade visible.
- **No per-tenant caps.** A single tenant can burn a shared budget, and every other tenant pays with degraded service. Multi-tenant surfaces require per-tenant caps or the "noisy neighbour" is a monthly event.
- **Fail-open on billed APIs.** An authenticated billed API that keeps serving when its quota store is unreachable turns a quota-service outage into a surprise bill. Fail-closed is the default on billed paths.
- **No attribution labels.** A cost without `tenant_id` and `cost_center` is a cost that cannot be challenged, defended, or reduced. Attribution is enforcement for finance teams.
- **Cardinality-explosive cost labels.** Putting `request_id` on `ml_cost_units_total` explodes the metrics backend. The policy names allowed labels; a new label that is not on the list requires review.
- **Retry-after as a magic number.** `Retry-After: 60` for every rejection is a hint, not a contract. The value is the bucket refill time at current rate; clients that back off by the right amount behave; clients that back off by less re-collide into the limiter.
- **Degradation without rollback rules.** A surface that degrades at 80% and never un-degrades is in a permanent soft-outage. The rollback condition is named per step.
- **Retraining pipelines outside the cost model.** Offline retraining can easily exceed monthly online inference cost. A budget that covers only serving is a budget that misses half the spend; the scheduler enforces job-class admission.
- **Provider-side budget ignored.** A tight internal quota does not help when the upstream provider itself throttles; cost-SLI C5 catches that, and the policy records the provider's own spend caps.
- **No incident on hard-cap breach.** Hitting the hard cap unplanned is an incident, not a dashboard anomaly. The error-budget policy in `exercise-01` and the incident drill in `exercise-03` are the follow-through.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. MANAGE function grounds the active enforcement of cost budgets as a risk control: the budget binds admission and launches rather than describing them after the fact.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Model-denial-of-service (expensive-input abuse, context-stuffing) and unmetered-inference abuse are explicit scenarios; quota enforcement is the countermeasure.
- MITRE ATLAS — <https://atlas.mitre.org/>. Named adversarial tactics include inference-API abuse and resource-exhaustion patterns that targeted quota enforcement defeats.

Local context (authoritative for the policy's shape and the cross-references):

- Paired lesson: `lessons/mod-307-ml-reliability-slos/exercises/exercise-02-cost-budgets-and-quotas.md`.
- Module README: `modules/mod-307-ml-reliability-slos/README.md`.
- Related exercise (reliability SLOs whose budget interacts with cost degradation steps): `modules/mod-307-ml-reliability-slos/exercise-01-ml-slis-and-slos/SOLUTION.md`.
- Related exercise (incident response drill — the runbook for an unplanned hard-cap breach): `modules/mod-307-ml-reliability-slos/exercise-03-ml-incident-response-drill/SOLUTION.md`.
- Related exercise (retraining as a deploy — scheduler admission against the monthly budget): `modules/mod-307-ml-reliability-slos/exercise-04-retraining-as-a-deploy/SOLUTION.md`.
- Related exercise (production LLM integration — the per-request latency and cost guardrails that cost policy is built against): `modules/mod-304-production-llm-integration/exercise-04-llm-cost-and-latency-guardrails/SOLUTION.md`.
