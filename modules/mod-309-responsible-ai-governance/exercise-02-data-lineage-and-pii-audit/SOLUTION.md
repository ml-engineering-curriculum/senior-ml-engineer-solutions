# exercise-02 — Data Lineage Graph, PII Register, And Lineage Audit (Reference Solution)

Paired exercise: `lessons/mod-309-responsible-ai-governance/exercises/exercise-02-data-lineage-and-pii-audit.md`.

## 1. Solution overview

This exercise produces three linked artefacts for one production ML surface and its training/eval pipeline:

1. `lineage.yaml` — a machine-readable data-lineage graph covering raw source → training set → trained model → served predictions and feedback-loop paths. Statically validatable; the exercise ships the schema and a validator.
2. `pii-register.md` — a dated, signed register of every personal-data category present at every node of the lineage, with legal basis, retention schedule, minimization treatment, and deletion / rectification propagation path.
3. `lineage-audit.md` — the written audit that reads the lineage graph against the PII register and names the gaps (untraced sources, PII categories without a legal basis, orphan datasets, broken deletion paths), with a prioritized action list.

The reference solution below shows the structure of each artefact, works a labeled illustrative example through them, and gives the grading rubric that calibrates every learner's set of artefacts to the same bar.

The exercise exists because a senior ML engineer's third-order output on any production surface — after the SLO and the Model Card — is the provenance story that makes the model's data trail auditable. Lineage without PII annotation is a graph; a PII register without lineage is a spreadsheet; the audit against both is the artefact that answers the questions legal, security, and governance ask when an incident happens: "where did this training example come from, who consented, which downstream models were trained on it, and what happens on a deletion request?" NIST AI RMF GOVERN function grounds the lineage as the governance substrate that makes dataset dependencies legible; MAP function grounds the PII register as the characterization of risk in the data; MANAGE function grounds the audit's action list as risk control on deletion / rectification / purpose-limitation obligations. The OWASP ML Top 10 category ML02 (Data Poisoning) is where the lineage artefact pays its operational rent: an unauthenticated upstream source in the graph *is* a poisoning surface.

## 2. Worked answer / implementation

### 2.1 Lineage graph: `lineage.yaml`

Lineage is a directed graph. Nodes are the datasets, pipelines, and models that touch training or serving data; edges are the transformations that produce one from another. The YAML encoding is deliberately minimal — rich enough to audit, plain enough to lint.

**Schema (author this once per surface, re-use across releases):**

```yaml
# lineage.yaml — data lineage for one ML surface
schema_version: "1.0"
surface:
  name: "agent-assist/draft"                   # user-facing name
  model_version: "assist-llm-v4.2.1"           # the model this lineage describes
  owner: "team-agent-assist"                   # L30 owner, role-named
  last_reviewed: "2026-11-15"                  # YYYY-MM-DD
  next_review_trigger:                         # list; any triggers a re-audit
    - "model retrain"
    - "new upstream dataset"
    - "deletion-policy change"
    - "incident with data attribution"

nodes:
  # Each node is one of: source | pipeline | dataset | model | sink
  - id: "src-support-ticket-db"
    type: "source"
    kind: "database"                           # database | api | s3 | message-bus | user-event-stream | third-party
    system: "support-platform"
    owner: "team-support-platform"
    description: "Primary support-ticket store. Rows inserted on ticket creation."
    schema_ref: "support-platform://schema/tickets_v3"
    pii_categories: ["name", "email", "free_text_body"]   # see pii-register.md
    legal_basis: "ToS §7.4 (2023-05 revision)"
    retention: "36 months raw"
    access_control: "RBAC: roles support-agent, data-engineer, model-owner"
    monitoring:
      freshness_sla: "p99 < 5 min lag"
      source_integrity: "write-path authenticated via mTLS; row-level audit log"
    governance_review: "gov-req-2024-01-support-source"

  - id: "pipe-ticket-export"
    type: "pipeline"
    tool: "data-platform-airflow"
    code_ref: "git://data-platform/dags/support-ticket-export@v3.4.0"
    owner: "team-data-platform"
    inputs: ["src-support-ticket-db"]
    outputs: ["ds-support-tickets-raw"]
    transformations:
      - "schema conformance: enforce support-platform://schema/tickets_v3"
      - "row-level filter: exclude flagged='restricted' or flagged='safety-sensitive'"
      - "deletion-propagation: exclude user_ids present in user-deletion-service exclusion list"
    monitoring:
      row_count_check: "relative delta < 10% day-over-day"
      schema_drift_check: "fail on unexpected column"
      freshness_check: "p99 < 1 h lag"

  - id: "ds-support-tickets-raw"
    type: "dataset"
    storage: "s3://ml-data/support-tickets-raw/dt=YYYY-MM-DD/"
    owner: "team-data-platform"
    format: "parquet"
    partitioning: "by day"
    retention: "12 months"
    pii_categories: ["name", "email", "free_text_body"]
    access_control: "roles data-engineer, model-owner; deny-by-default for support-agent"
    data_statement_ref: "data-statements/support-tickets-raw-v3.md"

  - id: "pipe-pii-redact"
    type: "pipeline"
    tool: "pii-redactor-v3"
    code_ref: "git://safety/pii-redactor@v3.2.1"
    owner: "team-data-platform"
    inputs: ["ds-support-tickets-raw"]
    outputs: ["ds-support-tickets-redacted"]
    transformations:
      - "regex + NER redaction for: name, email, phone, address, credit-card, SSN, IP-address"
      - "residual-rate sample audit: 0.3% (target < 0.5%); audited 2026-10"
    monitoring:
      residual_pii_sample_rate: "0.3%"
      redactor_version_pin: "v3.2.1"

  - id: "ds-support-tickets-redacted"
    type: "dataset"
    storage: "s3://ml-data/support-tickets-redacted/dt=YYYY-MM-DD/"
    owner: "team-data-platform"
    format: "parquet"
    retention: "12 months"
    pii_categories: ["free_text_body_residual"]   # residual only
    data_statement_ref: "data-statements/support-tickets-redacted-v3.md"

  - id: "pipe-training-build"
    type: "pipeline"
    tool: "training-platform"
    code_ref: "git://assist-llm/training/build_v4@v4.2.0"
    owner: "team-agent-assist"
    inputs: ["ds-support-tickets-redacted"]
    outputs: ["ds-training-v4"]
    transformations:
      - "dedup by template-reply hash"
      - "language filter: en-US, pt-BR, es-MX"
      - "queue filter: billing, shipping, account-mgmt"
      - "temporal window: 2024-01-01 .. 2026-06-30"
      - "EU-resident exclusion per legal guidance 2026-07"
    monitoring:
      row_count_check: "snapshot size within pre-registered bounds"
      language_distribution_check: "declared in data-statements/training-ds-v4.md §4"

  - id: "ds-training-v4"
    type: "dataset"
    storage: "s3://ml-data/training/assist-llm-v4/"
    owner: "team-agent-assist"
    retention: "12 months (recompiled on each retrain)"
    pii_categories: ["free_text_body_residual"]
    data_statement_ref: "data-statements/training-ds-v4.md"

  - id: "pipe-train"
    type: "pipeline"
    tool: "training-platform"
    code_ref: "git://assist-llm/training/train_v4@v4.2.0"
    owner: "team-agent-assist"
    inputs: ["ds-training-v4"]
    outputs: ["model-assist-llm-v4.2.1"]

  - id: "model-assist-llm-v4.2.1"
    type: "model"
    registry_ref: "registry://assist-llm-v4.2.1"
    owner: "team-agent-assist"
    base_model: "base-llm-v3"
    base_model_license: "internal commercial use, no redistribution"

  - id: "sink-serving-logs"
    type: "sink"
    storage: "s3://ml-logs/assist-llm/dt=YYYY-MM-DD/"
    owner: "team-agent-assist"
    retention: "30 days hot; 90 days cold"
    pii_categories: ["request_payload_residual", "user_id"]
    purpose: "debugging, monitoring; NOT approved for feedback-loop training without re-review"

  - id: "pipe-feedback-loop"
    type: "pipeline"
    tool: "training-platform"
    code_ref: "git://assist-llm/training/feedback_v4@v4.2.0"
    owner: "team-agent-assist"
    inputs: ["sink-serving-logs"]
    outputs: ["ds-feedback-v4"]
    transformations:
      - "agent-edit-signal extraction"
      - "re-apply PII redaction"
      - "deletion-propagation re-applied"
    governance_review: "gov-req-2026-09-feedback-loop"
    status: "approved with time-boxed review 2027-Q1"

edges:
  - { from: "src-support-ticket-db",   to: "pipe-ticket-export" }
  - { from: "pipe-ticket-export",      to: "ds-support-tickets-raw" }
  - { from: "ds-support-tickets-raw",  to: "pipe-pii-redact" }
  - { from: "pipe-pii-redact",         to: "ds-support-tickets-redacted" }
  - { from: "ds-support-tickets-redacted", to: "pipe-training-build" }
  - { from: "pipe-training-build",     to: "ds-training-v4" }
  - { from: "ds-training-v4",          to: "pipe-train" }
  - { from: "pipe-train",              to: "model-assist-llm-v4.2.1" }
  - { from: "model-assist-llm-v4.2.1", to: "sink-serving-logs" }
  - { from: "sink-serving-logs",       to: "pipe-feedback-loop" }
  - { from: "pipe-feedback-loop",      to: "ds-feedback-v4" }
```

**Node discipline.** Every source names its `system`, `owner`, `pii_categories`, `legal_basis`, and `retention`. Every pipeline names its `code_ref` (git URL + tag), `inputs`, `outputs`, and `transformations` (one bullet per material step). Every dataset names its `storage`, `owner`, `retention`, `pii_categories`, and a pointer to its Data Statement. Every model names its `registry_ref` and `base_model_license`. Every sink names its `retention` and whether it is approved for feedback-loop training (if yes, there must be a governance-review reference).

**Edge discipline.** Every edge is a directed `(from, to)` pair where `from` is a node id and `to` is a node id. The graph is validated as a DAG (loops indicate an unlabelled feedback path; feedback paths are modeled as a sink → pipeline → new dataset, not as a cycle).

### 2.2 Lineage validator: `lineage_validate.py`

The YAML is useful only if it is enforced. A small validator runs in CI on every change to `lineage.yaml` and fails on structural breaks.

```python
# lineage_validate.py
"""Validate a lineage.yaml artefact against the schema and the exercise-02 rules.

Fails on:
- unknown node types, missing required fields
- edges referencing unknown node ids
- cycles (feedback must be modeled as a sink -> pipeline -> dataset chain)
- PII-bearing nodes without a legal_basis or retention declaration
- datasets without a data_statement_ref
- pipelines without a code_ref
- sinks reused as training inputs without a governance_review reference
"""

from __future__ import annotations
import sys
import yaml


REQUIRED_NODE_FIELDS = {
    "source":   {"id", "type", "kind", "system", "owner"},
    "pipeline": {"id", "type", "tool", "code_ref", "owner", "inputs", "outputs"},
    "dataset":  {"id", "type", "storage", "owner", "retention"},
    "model":    {"id", "type", "registry_ref", "owner"},
    "sink":     {"id", "type", "storage", "owner", "retention"},
}

PII_BEARING_NODE_TYPES = {"source", "dataset", "sink"}


def _has_cycle(nodes: dict[str, dict], edges: list[dict]) -> bool:
    graph: dict[str, list[str]] = {nid: [] for nid in nodes}
    for e in edges:
        graph[e["from"]].append(e["to"])
    WHITE, GRAY, BLACK = 0, 1, 2
    color = {nid: WHITE for nid in nodes}

    def dfs(n: str) -> bool:
        color[n] = GRAY
        for m in graph[n]:
            if color[m] == GRAY:
                return True
            if color[m] == WHITE and dfs(m):
                return True
        color[n] = BLACK
        return False

    return any(color[n] == WHITE and dfs(n) for n in nodes)


def validate(doc: dict) -> list[str]:
    errors: list[str] = []
    if doc.get("schema_version") != "1.0":
        errors.append("schema_version must be '1.0'")

    surface = doc.get("surface", {})
    for required in ("name", "model_version", "owner", "last_reviewed"):
        if not surface.get(required):
            errors.append(f"surface.{required} is required")

    nodes_list = doc.get("nodes", []) or []
    nodes: dict[str, dict] = {}
    for n in nodes_list:
        nid = n.get("id")
        ntype = n.get("type")
        if not nid:
            errors.append(f"node without id: {n}")
            continue
        if nid in nodes:
            errors.append(f"duplicate node id: {nid}")
        nodes[nid] = n
        if ntype not in REQUIRED_NODE_FIELDS:
            errors.append(f"{nid}: unknown node type {ntype!r}")
            continue
        for f in REQUIRED_NODE_FIELDS[ntype]:
            if f not in n:
                errors.append(f"{nid}: missing required field {f}")

        if ntype in PII_BEARING_NODE_TYPES and n.get("pii_categories"):
            if ntype == "source" and not n.get("legal_basis"):
                errors.append(f"{nid}: PII-bearing source missing legal_basis")
            if not n.get("retention"):
                errors.append(f"{nid}: PII-bearing node missing retention")

        if ntype == "dataset" and not n.get("data_statement_ref"):
            errors.append(f"{nid}: dataset missing data_statement_ref")

    edges = doc.get("edges", []) or []
    for e in edges:
        if "from" not in e or "to" not in e:
            errors.append(f"edge missing endpoints: {e}")
            continue
        if e["from"] not in nodes:
            errors.append(f"edge {e}: unknown 'from' id")
        if e["to"] not in nodes:
            errors.append(f"edge {e}: unknown 'to' id")

    if _has_cycle(nodes, edges):
        errors.append("lineage graph has a cycle; feedback paths must be modeled as sink -> pipeline -> dataset")

    for nid, n in nodes.items():
        if n.get("type") != "sink":
            continue
        downstream_pipes = [e["to"] for e in edges if e["from"] == nid]
        for p in downstream_pipes:
            if nodes[p].get("type") == "pipeline" and not nodes[p].get("governance_review"):
                errors.append(
                    f"pipeline {p} consumes serving sink {nid} without governance_review reference"
                )

    return errors


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("usage: lineage_validate.py <lineage.yaml>")
        sys.exit(2)
    with open(sys.argv[1]) as fh:
        doc = yaml.safe_load(fh)
    errs = validate(doc)
    for e in errs:
        print(f"ERROR: {e}")
    sys.exit(1 if errs else 0)
```

`pyyaml` is a reasonable dependency; a dependency-free version using `json.loads` on a JSON-encoded lineage file is equivalent. The validator is small on purpose; a human still reviews the audit.

### 2.3 PII register: `pii-register.md`

The PII register is the semantic layer on top of the lineage graph. It enumerates every category of personal data that touches the pipeline, where it enters, where it flows, how long it is kept, under what legal basis, and by what mechanism it is minimized, redacted, or removed.

**Register structure:**

```markdown
# PII register — agent-assist/draft

Owner: `team-agent-assist` (consumer) + `team-data-platform` (data owner)
Legal reviewer sign-off: `legal-register/entry-0412` (2026-10-28)
Last reviewed: 2026-11-15
Next-review triggers: new data source, new PII category introduced,
retention-policy change, deletion-mechanism change, legal-register update.

## Category table

| Category | Sensitivity | Lineage nodes | Legal basis | Retention | Minimization | Deletion propagation |
|---|---|---|---|---|---|---|
| Customer name | moderate | `src-support-ticket-db`, `ds-support-tickets-raw` | ToS §7.4 (2023-05) + legitimate-interest internal product improvement, documented `legal-register/entry-0412` | Source: 36 mo. Raw dataset: 12 mo. | Redacted by `pii-redactor-v3` before entering redacted dataset; residual sample-audit rate 0.3%. | User deletion → `user-deletion-service` → exclusion list → next training recompile within 30 d; raw-dataset row-level soft-delete within 7 d. |
| Customer email | moderate | `src-support-ticket-db`, `ds-support-tickets-raw` | same as name | same | same | same |
| Phone number | moderate | source + raw | same | same | redacted | same |
| Street address | moderate | source + raw | same | same | redacted | same |
| Credit-card number | restricted | source + raw (should never reach raw; filtered at redactor) | PCI-DSS exclusion: disallowed in support-ticket body | not retained | hard-reject at `pipe-pii-redact`: any row with regex match is quarantined and logged to `security-incident-queue` | — (data is not stored in dataset) |
| SSN / national ID | restricted | source + raw | disallowed in support-ticket body | not retained | hard-reject at `pipe-pii-redact` | — |
| IP address | moderate | source (metadata), serving sink (request log) | legitimate-interest for abuse detection and debugging | source: 36 mo; sink: 30 d hot / 90 d cold | partial redaction in serving logs (last octet zeroed); full value kept in raw ticket DB for abuse-handling path | source deletion within 30 d of user deletion; sink deletion on retention expiry |
| Free-text body | high (free-form may contain any of the above) | `src-support-ticket-db` ... `ds-training-v4` | ToS §7.4 + legitimate-interest | 12 mo in raw, 12 mo in redacted, 12 mo in training (recompiled on retrain) | redacted by `pii-redactor-v3`; residual rate 0.3% sampled | user deletion → exclusion list → next retrain within 30 d (model does not unlearn; next-version model omits user's data) |
| User-id (internal) | low (indirect identifier) | all nodes | legitimate-interest, internal operational key | per-node retention | pseudonymized in training-set dataset (hashed with rotating salt) | exclusion list |
| Agent name | low | source + raw (as reply author) | ToS + employment data | per-node retention | redacted before entering training dataset | agent deletion → exclusion list |
| Tickets flagged `restricted` / `safety-sensitive` | restricted content category | source only | not approved for model training | filtered out at `pipe-ticket-export` | — | — |

## Deletion / rectification mechanism (detail)

- **Trigger.** A deletion or rectification request enters `user-deletion-service` via the user data-rights portal or a legal-register entry.
- **Fan-out.** `user-deletion-service` writes the user id to an exclusion list consumed by `pipe-ticket-export` (soft-delete in raw dataset within 7 d) and `pipe-training-build` (next recompile excludes user, within 30 d of deletion request per internal SLA).
- **Model behaviour.** The deployed model is not re-trained on request; the next-scheduled retrain excludes the user's data. If a regulatory requirement forces immediate unlearning, the escalation path is a hot retrain under governance exception (documented `gov-escalations/README.md`).
- **Serving sink.** Deletion within the sink's retention window (30 d hot) is handled by a daily sweep; deletions after cold-archive are handled on the 90-d cold expiry.
- **Rectification.** Customer name or address rectifications propagate the same way; the rectified value does not retroactively update prior training snapshots but is applied to the next snapshot.

## Jurisdictional considerations

- EU-resident data is excluded from v4 training per legal guidance (2026-07); the exclusion is implemented at `pipe-training-build` via a region-tag filter.
- GDPR data-subject-rights (access, deletion, rectification) propagate through `user-deletion-service` within 30 days of request.
- UK-resident data: same treatment as EU pending UK-AI guidance.
- US-state law (CA, VA, CO, CT, UT) data-subject-rights handled by the same deletion / rectification mechanism.

## PII register governance

- Owner: `team-agent-assist` (consumer) + `team-data-platform` (data owner).
- Sign-off: `team-agent-assist` tech-lead (2026-11-02); `team-data-platform` tech-lead (2026-11-02); `team-legal` reviewer (2026-10-28, documented `legal-register/entry-0412`).
- Change log: dated per entry change.
```

### 2.4 Lineage audit: `lineage-audit.md`

The audit is where the lineage and PII artefacts are read together and gaps are named. It is written by the surface's L30 owner and signed by the data owner and the legal reviewer.

**Audit structure:**

1. **System under audit.** Surface, model version, lineage artefact revision, PII register revision, date of audit, signatures.
2. **Prior audit, if any.** Link and the delta.
3. **Lineage completeness check.** For each node in the lineage: is every input source authenticated, documented, and in-policy? Are there orphan datasets (datasets with no forward edge to a model or sink)? Are there untraced training inputs (model with an input dataset whose provenance is not back-traceable to a source)?
4. **PII coverage check.** For each PII category in the register: does every node that bears it declare it? Does every node where it exits the pipeline (sinks) have a retention policy that satisfies the register? Does every deletion path reach every node where the category is stored?
5. **Legal-basis check.** Every PII category has a documented legal basis; each basis is dated, reviewed, and signed. A category whose basis has lapsed is flagged.
6. **Deletion-path audit.** For each category, trace the deletion path through the graph and verify that every PII-bearing node is reachable from `user-deletion-service`. A node not reachable is a broken deletion path and is a P0 finding.
7. **Feedback-loop audit.** Every sink → pipeline edge has a `governance_review` reference. Any sink being read by a pipeline without governance review is a P0 finding.
8. **Third-party-source audit.** Every source of kind `third-party` has a licence, a purpose limitation, and an expiry. Third-party sources with no governance review are a P0 finding (OWASP ML Top 10 ML02 Data Poisoning + supply-chain risk).
9. **Action list.** Prioritized gaps: `P0` (deletion / consent / unauthenticated-source / governance-missing issues), `P1` (coverage gaps, stale annotations), `P2` (documentation / lint issues). Each action has an owner (role), target date, and success criterion.
10. **Review cadence.** Quarterly, with named triggers for out-of-cycle re-audit (model retrain, new source, legal-register update, PII incident, platform change).

### 2.5 Illustrative worked example (not a real system)

> **Illustrative feature** — `agent-assist/draft`, an internal LLM-backed support-agent assistant. The lineage, register, and audit excerpts below use the `agent-assist` composite from exercise-01; do not treat it as a real system or incident.

**Lineage audit excerpt (gaps found):**

- **P0 — Broken deletion path on `sink-serving-logs`.** The sink carries `request_payload_residual` and `user_id` but the daily deletion sweep has been documented only for the hot-tier (30 d); the cold-tier (90 d) deletion sweep is undocumented. Action: document the cold-tier sweep as a scheduled job in the data-platform runbook, with an SLA of 7 days post-cold-expiry. Owner: `team-data-platform`. Target: 2026-12-15. Success: cold-sweep job registered in the platform scheduler and verified by a dry-run deletion.
- **P0 — Feedback-loop pipeline consumes sink before audit refresh.** `pipe-feedback-loop` is approved with a time-boxed review ending 2027-Q1. The audit confirms it is operating correctly today but flags the renewal: the governance reference `gov-req-2026-09-feedback-loop` must be reviewed before the time-box expires. Owner: `team-agent-assist` L30. Target: 2027-01-15. Success: renewal approved or pipeline paused.
- **P1 — Legal-basis expiry on ToS §7.4.** Current ToS revision is 2023-05. A new ToS revision is planned for 2027-Q1; the register's legal-basis cell must be updated and signed by legal on the new revision date. Action: track in `team-legal` roadmap. Owner: `team-legal` + `team-agent-assist` governance reviewer. Target: aligned with ToS revision date. Success: register entry updated and signed within 7 days of ToS revision.
- **P1 — Residual-PII sample audit cadence.** `pipe-pii-redact` reports a 0.3% residual rate audited 2026-10. Audit cadence is currently "ad hoc"; the audit recommends adopting a monthly sampling cadence with a tracked metric. Owner: `team-data-platform`. Target: 2026-12-31. Success: monthly sample audit job registered; results pushed to `team-agent-assist`.
- **P2 — Lineage artefact lint error.** `lineage.yaml` had a missing `data_statement_ref` on `ds-feedback-v4` (fixed in this audit's PR). Owner: `team-agent-assist`. Target: closed. Success: `lineage_validate.py` passes in CI.

**Deletion-path trace for "customer email":**

- Source: `src-support-ticket-db` — soft-delete on exclusion-list propagation within 7 d.
- Raw dataset: `ds-support-tickets-raw` — row-level soft-delete on next pipeline run, within 24 h of exclusion-list propagation.
- Redacted dataset: `ds-support-tickets-redacted` — email already redacted; no PII to delete, but user_id-level row is soft-deleted to preserve referential integrity.
- Training dataset: `ds-training-v4` — next recompile excludes user's data within 30 d of deletion request.
- Deployed model: `model-assist-llm-v4.2.1` — not re-trained on request; next retrain omits user data.
- Serving sink: `sink-serving-logs` — residual-redacted request payload is soft-deleted on hot-sweep within 24 h; cold-tier sweep per P0 action.
- Feedback-loop dataset: `ds-feedback-v4` — recompiled with deletion propagation re-applied on each loop iteration.

Every node bearing the "customer email" category is reachable by the deletion path. The only residual risk (the deployed model retains memorized content from training data) is the risk row in Model Card §8 (data-privacy), mitigated by the redactor and bounded by the residual-PII audit.

### 2.6 How the three artefacts connect to exercise-01 and NIST AI RMF

- Lineage graph → Model Card §1 links and Data Statement §3 provenance.
- PII register → Data Statement §7 preprocessing + §8 consent / retention / deletion + Model Card §8 data-privacy risk row.
- Lineage audit → fed into the governance-review packet for the surface; actions tracked against the module's governance registry.
- NIST AI RMF GOVERN function: the lineage artefact *is* the governance substrate of the ML surface.
- NIST AI RMF MAP function: the PII register characterizes the data-side risk.
- NIST AI RMF MANAGE function: the audit's action list is the risk-control surface.
- OWASP ML Top 10 ML02 (Data Poisoning): the lineage audit's third-party-source check is the operational control against unauthenticated upstream poisoning.

## 3. Validation steps

Walk the three artefacts through this checklist before considering them done:

1. **`lineage_validate.py` passes.** The validator is a CI gate; a failing lineage is a rejected lineage.
2. **Every source has a `legal_basis` and a `retention`.** A source without a legal basis cannot be an input.
3. **Every dataset has a `data_statement_ref`.** Datasets without Data Statements are undocumented.
4. **Every pipeline has a `code_ref` (git URL + tag).** Un-versioned pipelines are un-reviewable.
5. **The graph is a DAG.** Cycles indicate an unlabelled feedback path; feedback is modelled as a sink → pipeline → new dataset.
6. **Every PII category in the register is represented in at least one lineage node's `pii_categories`.** A category in the register with no lineage node is either stale or a gap in the graph.
7. **Every lineage node whose type is `source | dataset | sink` has its `pii_categories` field aligned with the register.** A node with PII but no category is either wrong in the YAML or stale in the register.
8. **Every deletion path reaches every PII-bearing node.** The audit's §6 trace must cover every category.
9. **Every `legal_basis` is dated and signed by legal.** A basis without a date is a draft, not a basis.
10. **Every sink → pipeline edge has a `governance_review`.** Feedback loops without governance are unauthorized data reuse.
11. **Third-party sources have an expiry.** A third-party source without an expiry is a drift risk.
12. **Audit §4 and §5 reconcile.** PII coverage and legal-basis coverage are two sides of the same audit; a gap in one is a finding in the other.
13. **NIST AI RMF MAP / MANAGE / GOVERN cited.** The audit names which function each finding addresses.
14. **OWASP ML Top 10 ML02 cited** on the third-party-source check.

Static checks a CI step can enforce:

```bash
python lineage_validate.py lineage.yaml
grep -ciE "legal basis|retention|deletion"            pii-register.md
grep -ciE "governance|review cadence|sign.?off"       pii-register.md
grep -ciE "deletion path|p0|p1|p2|action"             lineage-audit.md
grep -ciE "nist ai rmf|govern|map|manage"             lineage-audit.md
grep -ciE "owasp.*ml.*top.*10|ml02|data poisoning"    lineage-audit.md
```

## 4. Rubric / review checklist

- [ ] `lineage.yaml` validates with `lineage_validate.py` (no errors).
- [ ] Every source node has `system`, `owner`, `pii_categories`, `legal_basis`, `retention`, and `access_control`.
- [ ] Every pipeline node has `code_ref` (git URL + tag), `inputs`, `outputs`, and a bullet list of material `transformations`.
- [ ] Every dataset node has `storage`, `owner`, `retention`, and `data_statement_ref` tying to exercise-01's Data Statement.
- [ ] Every model node has `registry_ref` and `base_model_license`.
- [ ] Every sink node has `storage`, `retention`, and either a "NOT approved for feedback-loop training" statement or a `governance_review` reference on any pipeline consuming it.
- [ ] Graph is a DAG; feedback paths are modelled as `sink → pipeline → new dataset`.
- [ ] `pii-register.md` enumerates every PII category: sensitivity, lineage nodes, legal basis, retention, minimization treatment, deletion-propagation path.
- [ ] Register has a deletion / rectification mechanism section naming trigger, fan-out, model behaviour, serving-sink sweep, rectification semantics.
- [ ] Register has a jurisdictional-considerations section naming EU / UK / US-state treatment.
- [ ] Register is signed: data-owner, surface-owner, legal reviewer, all dated.
- [ ] `lineage-audit.md` names the surface, lineage artefact revision, PII register revision, date, signatures, and prior-audit delta.
- [ ] Audit includes: lineage-completeness check, PII-coverage check, legal-basis check, deletion-path audit, feedback-loop audit, third-party-source audit.
- [ ] Audit includes a traced deletion path per PII category, demonstrating every PII-bearing node is reachable.
- [ ] Audit's action list uses `P0 / P1 / P2`, role-owners, target dates, and success criteria.
- [ ] Audit has a review cadence (default quarterly) and trigger-based re-audit conditions.
- [ ] NIST AI RMF GOVERN / MAP / MANAGE cited in the audit; OWASP ML Top 10 ML02 cited on the third-party-source check.
- [ ] Cross-reference to exercise-01 (Data Statement § provenance and § preprocessing), exercise-03 (slice-cadence that reads the training dataset), exercise-04 (threat model that reads the lineage for supply-chain and poisoning vectors).

Review-weighted points for grading:

- Lineage graph completeness and validator-passing: 20%
- Lineage node discipline (code_ref, retention, data_statement_ref, governance_review on sinks): 15%
- PII register coverage and legal-basis discipline: 20%
- Deletion / rectification mechanism clarity: 15%
- Lineage audit structure (six checks, prioritized actions, review cadence): 15%
- Jurisdictional coverage and sign-offs: 5%
- Standards alignment (NIST AI RMF GOVERN/MAP/MANAGE, OWASP ML Top 10 ML02): 5%
- Cross-consistency with exercise-01 Data Statement and §8 risk rows: 5%

## 5. Common mistakes

- **Lineage as prose.** A lineage "section" written as paragraphs rather than a machine-readable graph is not an artefact — it is a story. A graph is validatable; prose is not.
- **Source without legal basis.** "It's our customer data" is not a legal basis; the ToS clause + legitimate-interest / consent / exemption + date is.
- **Pipeline without `code_ref`.** An un-versioned transformation is un-reviewable; a reviewer cannot see what the pipeline did last quarter if the pipeline is not pinned to a git tag.
- **Dataset without Data Statement.** The lineage names *where* the data is; the Data Statement names *what* it is. Both are required.
- **Feedback loop as a cycle.** Treating serving logs → training as an edge in a cycle hides the governance-review gate. The correct model is sink → pipeline (with governance reference) → new dataset.
- **PII register unreconciled with lineage.** A category in the register with no lineage node means the register is stale or the graph is missing a node. Reconciliation is an audit check for a reason.
- **"We redact PII" without a residual rate.** Redaction is never perfect; a residual rate (sampled and tracked over time) is the honest form. "Redacted" with no rate is a vibe.
- **Deletion mechanism as aspiration.** "We honor deletion requests" is not a mechanism. "User-deletion-service → exclusion list → next retrain within 30 d" is.
- **Model unlearning as a deletion mechanism.** Deployed models are not unlearned on request; the next retrain excludes the user. Any claim of "unlearning" requires a specific technical procedure and an evaluation of its effectiveness.
- **Cold-tier sinks forgotten.** Hot-tier retention is usually visible; cold-tier is usually invisible. The audit's job is to make cold-tier visible.
- **Jurisdictional silence.** An audit that does not name EU / UK / US-state treatment is not approvable for a globally-serving surface.
- **Third-party source without an expiry.** Licences lapse. A third-party source without an expiry is a drift risk and a P0 finding.
- **No sign-off.** An unsigned PII register or audit is a draft. The signatures (data owner, surface owner, legal reviewer) are what make the artefact the governance contract.
- **Audit as backlog, not decision.** An action list without P0 / P1 / P2 priorities and dated owners is a backlog. The audit's job is to force prioritized decisions, not to list every observation.

## 6. References

Official standards cited in the module's source policy:

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework>. GOVERN function grounds the lineage artefact as the governance substrate for the ML surface. MAP function grounds the PII register as the characterization of data-side risk. MANAGE function grounds the audit's action list as the risk-control surface.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/>. ML02 Data Poisoning anchors the third-party-source and feedback-loop audits — an unauthenticated upstream source or an un-reviewed feedback-loop pipeline is an operational poisoning surface. ML05 Model Theft and ML06 AI Supply Chain Attacks are cross-referenced on the registry / base-model-licence rows.
- NIST AI 600-1 (Generative AI Profile) — <https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf>. The data-privacy and information-integrity GAI risk rows in the Model Card §8 (exercise-01) pay back into the PII register's retention / deletion policy here.
- MITRE ATLAS — <https://atlas.mitre.org/>. Referenced on the audit's feedback-loop and third-party-source checks for the adversarial tactics those surfaces present (e.g., AML.T0019 Publish Poisoned Datasets, AML.T0010 ML Supply Chain Compromise).

Practitioner implementation examples:

- VeriSwarm Trust Center — <https://veriswarm.ai/trust>. Example of a practitioner publishing a data-handling / trust page that references lineage artefacts externally; useful only as a shape-of-artefact reference, not as a standard.

Local context (authoritative for the artefact shape and cross-references):

- Paired lesson: `lessons/mod-309-responsible-ai-governance/exercises/exercise-02-data-lineage-and-pii-audit.md`.
- Module README: `modules/mod-309-responsible-ai-governance/README.md`.
- Related exercise (Model Card + Data Statement that the lineage and PII register cross-reference): `modules/mod-309-responsible-ai-governance/exercise-01-model-card-authoring/SOLUTION.md`.
- Related exercise (fairness / robustness slice cadence that reads the training dataset): `modules/mod-309-responsible-ai-governance/exercise-03-fairness-and-robustness-slice-cadence/SOLUTION.md`.
- Related exercise (misuse and adversarial threat model that reads the lineage for supply-chain and poisoning vectors): `modules/mod-309-responsible-ai-governance/exercise-04-misuse-and-adversarial-threat-model/SOLUTION.md`.
- Related exercise (data quality / contracts paved-road row of the audit): `modules/mod-308-platform-collaboration/exercise-01-paved-road-consumption-audit/SOLUTION.md`.
- Related exercise (offline evaluation harness whose slice-dataset lineage cross-references this exercise): `modules/mod-305-advanced-evaluation/exercise-01-offline-harness-with-slices-and-guardrails/SOLUTION.md`.
