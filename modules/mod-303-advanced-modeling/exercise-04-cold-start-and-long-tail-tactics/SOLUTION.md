# exercise-04 — Cold-Start and Long-Tail Tactics (Reference Solution)

Paired exercise: `lessons/mod-303-advanced-modeling/exercises/exercise-04-cold-start-and-long-tail-tactics.md`.

## 1. Solution overview

This exercise produces `cold-start-long-tail-plan.md` — a design that names, for one shipping ML system, (1) which cold-start problem it actually has, (2) which long-tail problem it actually has, and (3) a concrete set of tactics with measurable evaluation slices. These are two problems, not one, and most production regressions come from conflating them. The artefact has five parts:

1. A **problem taxonomy** — classify the system's cold-start as user-cold / item-cold / system-cold, and classify the long-tail as label-long-tail / feature-long-tail / user-behaviour-long-tail.
2. A **tactic catalogue** — the specific tactics considered for each problem class, with inclusion criteria and known-rejection reasons.
3. An **evaluation design** — the slices on which cold and long-tail performance are measured separately from aggregate metrics, so the head of the distribution cannot hide tail regressions.
4. An **exploration / safety plan** — how the system acquires signal on cold items without exposing users to too many bad recommendations or decisions, framed as an exploration-vs-exploitation choice.
5. A **monitoring plan** — how the team notices when a tail regression is happening in production before the aggregate metric moves.

The reference solution below walks through the five parts on one illustrative problem, labels it clearly as illustrative, and gives the grading rubric that applies to every learner's plan.

Cold-start and long-tail are the two places where "we beat the baseline by 1pp on holdout" is most often disconnected from "the system got better." This exercise exists to force that disconnect to be visible before launch, not after.

## 2. Implementation (worked answer)

### 2.1 Problem taxonomy

#### Cold-start — three distinct problems

- **User-cold.** New user, no history. The ranker / classifier has no behavioural features for this user. Most product launches hit this first.
- **Item-cold.** New item, new content, or new entity (new product SKU, new post, new provider) with no interaction history. The system's embedding for this item is either absent or initialised from side information only.
- **System-cold.** New vertical, new country, new market, new model altogether — nothing has history. This is the setting where offline baselines are the most misleading because they have no data to be measured on.

Each has a different mitigation surface. Conflating "we have a cold-start problem" without saying which one is the most common symptom of an incomplete design.

#### Long-tail — three distinct problems

- **Label long-tail.** Classification problems with thousands of labels where a small set dominates the distribution. The tail labels have tens to hundreds of examples each. Under-represented tail classes hurt macro-averaged metrics but hide in micro-averaged ones.
- **Feature long-tail.** Categorical features with high cardinality and a power-law frequency distribution. Rare feature values are either treated as `<UNK>` (information lost) or get their own embeddings (overfit).
- **User-behaviour long-tail.** Interaction count per user or per item follows a power law. The model is trained on examples dominated by heavy users, then deployed on a traffic mix that includes light and new users whose behaviour the model has barely seen.

The three long-tail problems have different fixes, which is why they get separated in the design.

### 2.2 Tactic catalogue

For each problem class, the design document should pick from — and explicitly reject from — this catalogue. "Rejected because …" is as important as "picked because …".

#### User-cold tactics

| Tactic | Mechanism | When it fits | Known issues |
|---|---|---|---|
| **Popularity backoff** | Fall back to a popularity-ranked feed until N interactions. | Cheap, strong baseline. Floor; not ceiling. | Can create a popularity trap for subsequent personalisation. |
| **Context-based personalisation** | Rank on context features only (device, locale, time, entry path) until behavioural signal accrues. | Any system with reasonable context signal. | Weak when context is uninformative. |
| **On-boarding preference capture** | Explicit "pick three interests" step at sign-up. | Products where friction is tolerable. | Users lie / skip; biases the first cohort. |
| **Collaborative + content hybrid** | Combine CF embeddings (strong when present) with content embeddings (always present). | Default for recommendation systems. | Hybrid weight needs tuning per-cohort. |
| **Meta-learning (MAML-style)** | Train the model to adapt quickly from very few examples per new user. | Research-mature; implementation-heavy. | Rarely the cheapest fix; justify vs. hybrid first. |

#### Item-cold tactics

| Tactic | Mechanism | When it fits | Known issues |
|---|---|---|---|
| **Content-feature embedding** | Represent new items via their content features (text, image, metadata) through a pre-trained encoder. | Always the floor; new items with zero interactions still get a usable embedding. | Content embedding ≠ behaviour embedding; the distribution of content-embeddings differs from CF-trained embeddings. |
| **Side-information warm-up** | Initialise CF embedding from a learned mapping of side features. Learn the mapping on items with both. | Good when you have many "warm" items with both side info and CF embeddings. | The mapping is only as good as the warm sample it learned from. |
| **Exploration slot** | Reserve a small fraction of impressions for new items regardless of model score. | Marketplace / content platforms with ongoing item churn. | Must be budget-capped; see §2.4. |
| **Two-tower retrieval with shared embedding space** | Item tower embeds from content features; new items get embedded for free at serve time. | The default for large retrieval systems. | Does not fix the ranker on top. |

#### System-cold tactics

| Tactic | Mechanism | When it fits | Known issues |
|---|---|---|---|
| **Heuristic baseline + instrumentation** | Ship a rule-based or popularity baseline and instrument thoroughly. | Launch day, no data. | Looks un-ML; is the right answer most of the time. |
| **Transfer from a sibling model** | Fine-tune a model trained on a related market or vertical. | Cross-market launches where distributions overlap. | Transfer failures are catastrophic when the overlap is assumed, not measured. |
| **LLM-as-cold-start-ranker** | Use a hosted LLM for ranking until behavioural data accrues. | Low-volume launches with tolerant latency. | Cost, latency, and supply-chain surface (see MITRE ATLAS). |

#### Label long-tail tactics

| Tactic | Mechanism | When it fits | Known issues |
|---|---|---|---|
| **Class-balanced loss (Cui et al., 2019)** | Re-weight loss by the effective number of samples per class. | Classification with hundreds+ classes. | Over-weights noisy tail labels. |
| **Focal loss (Lin et al., 2017)** | Down-weight well-classified examples; focus on hard tail examples. | Detection and classification with heavy class imbalance. | Sensitive to γ; tune on a tail slice. |
| **Decoupled representation + classifier training** | Train the backbone on instance-balanced data, re-train the classifier on class-balanced data (Kang et al., 2020). | Long-tailed classification where feature learning and classification require different sampling. | Two-stage training infra. |
| **Hierarchical labels / grouping** | Predict parent class; fall back to parent when child confidence is low. | Taxonomies with meaningful hierarchy. | Needs clean taxonomy. |
| **Oversampling / SMOTE-family** | Synthetic or repeated sampling of tail classes. | Small-class classification. | SMOTE in embedding space is useful; SMOTE in raw space often meaningless for high-dim data. |

#### Feature long-tail tactics

| Tactic | Mechanism | When it fits | Known issues |
|---|---|---|---|
| **Hashing + collision tolerance** | Map high-cardinality IDs into a fixed-size table via hashing. | Billions of IDs, bounded memory. | Collision-induced interference; watch rare-category regressions. |
| **Frequency-based bucketing** | Treat all values below a count threshold as `<RARE>`. | Categorical features with long tails. | Loses signal on rare values that matter (e.g., fraud-indicating rare merchants). |
| **Compositional / sub-token embeddings** | Build embeddings from pieces (character n-grams, product-attribute tuples). | When the feature has compositional structure. | Infrastructure to produce the sub-tokens. |

#### User-behaviour long-tail tactics

| Tactic | Mechanism | When it fits | Known issues |
|---|---|---|---|
| **Per-cohort evaluation + per-cohort models** | Separate models or separate heads per user activity tier. | Large user base with clearly-separable tiers. | Operational cost of multiple models. |
| **Inverse-propensity / exposure re-weighting** | Correct training-data bias toward heavy users. | When logged data over-represents a cohort. | Propensity model is itself a dependency. |
| **Session-based models** | Model short sessions explicitly, not user histories. | Light-user-dominated traffic. | Different modeling family; added infra. |

### 2.3 Evaluation design

Aggregate metrics are not enough. The design must specify:

1. **Named cold slices.** For each cold-start class the system has, name the slice ("users with fewer than N sessions", "items created in the last K days", "merchants new to this market"). Metrics are computed on each.
2. **Named tail slices.** For each long-tail class the system has, name the slice ("labels with < N training examples", "feature values appearing < K times in the last T days", "user cohort with N < 5 lifetime sessions").
3. **Non-aggregate metrics.** Macro-averaged accuracy / F1 (so tail classes get equal weight). Micro-only reporting is a long-tail regression generator.
4. **Head / tail decomposition.** Report the metric split at head vs. tail. A model that improves head by 2pp and regresses tail by 1pp has not improved the system — it has compressed the tail further.
5. **Baselines per slice.** The baseline for a cold slice is usually a non-model heuristic (popularity, random, context-only). If the ML model does not beat that baseline on the cold slice, the ML model is worse on the cold slice than doing nothing.
6. **Exposure-controlled offline metrics.** For recommendation systems, inverse-propensity-weighted offline evaluation on the tail is often required because logged exposure is already biased toward the head.

### 2.4 Exploration / safety plan

Cold items only accrue data if the system shows them. This is an exploration-vs-exploitation choice; the design has to make it:

- **Explicit exploration budget.** "Reserve X% of impressions for items in the item-cold slice." Measured and capped.
- **Contextual bandit as the exploration policy.** Thompson sampling or UCB-style allocation of the exploration budget. Equal-random exploration is wasteful and user-hostile.
- **Safety guardrails on exploration.** Hard rules that override the exploration policy: do not show cold items that fail content safety checks regardless of exploration budget; do not show cold items to cold users at the same time (cold²); respect any user-level content filters.
- **Measurable exploration outcome.** Report "how many new items crossed N interactions in week T" as a direct measure of whether exploration is working, not just "we spent X% of impressions on exploration."

The safety plan is where OWASP ML Top 10 and MITRE ATLAS context matters most: an exploration slot that shows any content indiscriminately is a surface for content-based attacks (poisoning the recommender with adversarial content). The plan must name the content-safety filter that runs *before* the exploration policy sees a candidate.

### 2.5 Monitoring plan

The system must surface tail regressions before they become incidents:

- **Per-slice metric dashboards** on the slices named in §2.3 — not just global metrics. Cold / tail regressions are invisible on global dashboards until they become large.
- **Exposure monitoring** per slice. Did exposure to new items drop? Did exposure to tail labels drop? Shift in exposure predicts shift in metric.
- **Alerting on the tail, not just the head.** A tail-slice metric that regresses by > X% relative to its last-week value is page-worthy, even when the global metric is flat.
- **Drift on cold triggers.** Whenever the mix of cold users or cold items changes by more than Y% (e.g., a new market opens), the cold-slice evaluation is re-run on a fresh slice before any model promotion.
- **NIST AI RMF MEASURE alignment.** The per-slice monitoring is the measurable-characteristic implementation; cite it.

### 2.6 Illustrative worked plan (not a real product)

> **Illustrative problem** — a two-tower recommendation system for a marketplace with heterogeneous sellers. New sellers join weekly, new users sign up continuously, and the catalogue has a long-tail distribution across thousands of categories. This example is a composite for calibration; it is not a real marketplace and the numbers are not measured.

Problem taxonomy:

- Cold-start: **user-cold** (continuous sign-ups), **item-cold** (new sellers / new listings weekly). System-cold does not apply — the marketplace has history.
- Long-tail: **label long-tail** (category taxonomy), **feature long-tail** (rare brand / attribute values), **user-behaviour long-tail** (power-law session counts).

Tactic picks:

- **User-cold:** popularity backoff for the first K interactions, context features (device, locale, entry path) in a dedicated cold-user tower, hybrid weight tuned per-cohort. Meta-learning rejected on cost / complexity.
- **Item-cold:** content-feature embeddings via a pre-trained encoder (text + image), side-information warm-up mapping trained on warm items, 5% exploration slot allocated via Thompson-sampling bandit for new-seller items.
- **Label long-tail:** macro-F1 as the primary offline metric; class-balanced loss for the categorisation head; hierarchical fallback to parent category when child confidence is below 0.4.
- **Feature long-tail:** compositional embeddings for brand / attribute (concatenate learned embeddings of seller + attribute token pieces); `<RARE>` bucket for values appearing < 5 times in the last 30 days; monitor fraud-sensitive rare values separately.
- **User-behaviour long-tail:** two heads at serving (session-based for light users, history-based for heavy users), with the router using lifetime-session-count thresholds.

Evaluation design:

- Cold slices: new-user (< 3 lifetime sessions), new-seller (< 7 days on platform), new-market (if any).
- Tail slices: categories with < 500 training examples, brands with < 10 impressions per day, users with < 5 lifetime sessions.
- Macro-F1 for category prediction, head-vs-tail decomposed. For retrieval: recall@K with head/tail split, IPS-weighted on the tail.
- Baselines per slice: popularity for cold users; content-only ranker for cold items; parent-category prediction for tail labels.
- Gate: no promotion of a joint model that regresses any cold or tail slice vs. the baseline or the prior model.

Exploration / safety plan:

- Hard budget: 5% of top-10 impressions reserved for items in the item-cold slice.
- Policy: Thompson-sampling bandit over a cold-item candidate pool.
- Safety guardrails: content-safety filter runs before the bandit; cold items are never shown to cold users (cold²); respect user-level filters.
- Outcome metric: number of new items crossing 100 interactions per week.

Monitoring plan:

- Per-slice dashboards (new-user recall, new-seller exposure, tail-category macro-F1, light-user session CTR) updated daily.
- Alerting: any slice metric regresses > 3% relative to a 7-day rolling baseline → page the on-call ML engineer.
- Exposure drift alert: cold-user traffic share changes by > 20% WoW → re-run cold-slice evaluation.
- NIST AI RMF MEASURE cited in the dashboard's design doc.

## 3. Validation steps

Walk through this checklist on the artefact:

1. **Cold-start class is named specifically** (user-cold / item-cold / system-cold). "We have a cold-start problem" is not enough.
2. **Long-tail class is named specifically** (label / feature / user-behaviour). Same.
3. **Every chosen tactic has a rejection note on the alternatives.** "We picked hybrid CF+content; we did not pick meta-learning because …" This is a design review, not a tactic bingo card.
4. **Per-slice evaluation exists and is primary, not supplementary.** Any plan that reports only aggregate metrics has not been reviewed yet.
5. **Baselines per slice are named.** If the ML model does not beat the baseline on the cold slice, the ML model does not ship for that slice.
6. **Exploration has a budget and a safety filter.** Un-budgeted exploration is a user-hostile change; un-filtered exploration is a safety incident waiting to happen.
7. **Monitoring is per-slice and alerts on the tail.** Aggregate-only alerting will miss every tail regression.
8. **Rollback per slice.** A tactic that regresses one slice can be rolled back without rolling back everything.
9. **Cross-refs to NIST AI RMF MEASURE** on the monitoring, and to MITRE ATLAS + OWASP ML Top 10 on the exploration safety plan.

Static check:

```bash
grep -c "cold-start\|cold slice"      cold-start-long-tail-plan.md   # expect >= 2
grep -c "tail slice\|long-tail"       cold-start-long-tail-plan.md   # expect >= 2
grep -c "baseline"                    cold-start-long-tail-plan.md   # expect >= 2
grep -c "Exploration budget\|exploration budget" cold-start-long-tail-plan.md  # expect >= 1
grep "last reviewed"                  cold-start-long-tail-plan.md   # must exist within 90 days
```

## 4. Rubric / review checklist

- [ ] Problem taxonomy names the cold-start class (user / item / system) and the long-tail class (label / feature / user-behaviour) explicitly.
- [ ] For each class, a tactic is chosen with a one-sentence rationale and at least one rejection note.
- [ ] Evaluation design names cold slices and tail slices with concrete thresholds (not "tail items").
- [ ] Each cold / tail slice has a non-ML baseline named. The ML model must beat that baseline on the slice.
- [ ] Macro-averaged metric (or an equivalent tail-weighted metric) is primary for long-tail classification; aggregate metric is reported but not primary.
- [ ] Exploration plan has an explicit budget, a policy (ideally contextual bandit), and a named safety filter that runs before the policy.
- [ ] Safety filter is justified against OWASP ML Top 10 / MITRE ATLAS context (content-based attack surface on exploration slots).
- [ ] Monitoring plan has per-slice dashboards, exposure drift alerts, and tail-slice regression alerts.
- [ ] Rollback is per-slice or per-tactic, not all-or-nothing.
- [ ] NIST AI RMF MEASURE alignment cited on the monitoring plan.
- [ ] Artefact has a "last reviewed" date and an owner for the per-slice dashboards.

Review-weighted points (for grading):

- Problem taxonomy correctness and specificity: 15%
- Tactic catalogue coverage (picks + rejections): 20%
- Evaluation design (slices, baselines, head/tail decomposition): 25%
- Exploration / safety plan (budget, policy, safety filter): 20%
- Monitoring plan (per-slice, exposure drift, tail alerts): 20%

## 5. Common mistakes

- **Treating cold-start as one problem.** User-cold, item-cold, and system-cold have different fixes; the design must pick which one it is addressing.
- **Treating long-tail as one problem.** Same: label / feature / user-behaviour long-tails are distinct and each has its own tactic catalogue.
- **Aggregate-only metrics.** The most common cause of invisible tail regressions. Macro-averaging and head/tail decomposition are mandatory for long-tail systems.
- **No slice baseline.** Reporting that the ML model has 0.85 macro-F1 on the tail is meaningless without the baseline. Popularity / random / parent-class prediction are the baselines.
- **Un-budgeted exploration.** "Reserve some impressions for new items" without a cap is a reputation incident. Budget it.
- **Exploration without safety filter.** Every content-based attack surface on recommenders starts with "the exploration slot showed something harmful." The content-safety filter must run before the exploration policy, not after.
- **Cold-user × cold-item simultaneously.** A new user getting a new item with no history is not a personalisation decision; it is a coin flip. Avoid stacking cold conditions.
- **Over-reliance on meta-learning.** Meta-learning is a strong research result and expensive engineering. Justify it against the cheaper hybrid baseline first.
- **SMOTE in raw high-dimensional space.** SMOTE's assumptions do not hold in high-dim representation spaces; apply in embedding space or not at all.
- **Hashing collisions ignored.** Feature hashing is a long-tail tactic, but collisions interfere. Monitor rare-value accuracy after a hash-table resize.
- **No per-slice rollback.** If a tactic regresses the item-cold slice but improves everything else, the design should let you roll that tactic back without reverting the whole model.
- **Single-dashboard summary.** One number does not tell you which slice regressed. Per-slice dashboards are not optional.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. Source of the MEASURE function (per-slice measurement of trustworthy-AI characteristics including validity and reliability under distribution shift) referenced throughout the evaluation and monitoring plans.
- MITRE ATLAS — <https://atlas.mitre.org/>. Source of the adversarial-ML context for recommender-system attacks that exploit exploration slots and cold-start candidate pools.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. Source of the input-validation and data-poisoning context relevant to cold-item admission and exploration safety filters.

Local context:

- Paired lesson: `lessons/mod-303-advanced-modeling/exercises/exercise-04-cold-start-and-long-tail-tactics.md`.
- Module README: `modules/mod-303-advanced-modeling/README.md`.

Prior art named by name:

- Cui, Jia, Lin, Song, Belongie (2019) — class-balanced loss based on effective number of samples.
- Lin, Goyal, Girshick, He, Dollár (2017) — focal loss for dense object detection.
- Kang, Xie, Rohrbach, Yan, Gordo, Feng, Kalantidis (2020) — decoupling representation and classifier for long-tailed recognition.
- Finn, Abbeel, Levine (2017) — model-agnostic meta-learning (MAML).
- Agarwal, Basu, Schnabel, Joachims (2019) and related — off-policy evaluation and IPS-weighted recommender evaluation.
- Chapelle & Li (2011) — empirical evaluation of Thompson sampling in contextual bandits.
