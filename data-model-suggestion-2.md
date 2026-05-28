# Data Model Suggestion 2: Event-Sourced / Audit-First

> Project: ML Model Registry & Serving · Created: 2026-05-20

## Philosophy

This approach treats every state change in the model lifecycle as an immutable domain event stored in an append-only event store. The current state of any model, version, endpoint, or compliance record is derived by replaying events rather than reading a mutable row. Read-optimised materialised views (projections) provide fast query access for the REST API and UI, following the CQRS (Command Query Responsibility Segregation) pattern.

This architecture is inspired by financial ledger systems and regulatory audit platforms where the ability to answer "what was the state at time T?" is a first-class requirement. For an ML model registry operating under EU AI Act mandates (Article 12 requires automatic event logging; Article 18 mandates 10-year documentation retention), event sourcing provides a natural and tamper-evident audit trail. Every model registration, stage transition, deployment, drift alert, and rollback is a permanent record that cannot be silently modified.

The trade-off is increased complexity: the application must maintain projections, handle eventual consistency between the event store and read models, and developers must think in terms of events rather than CRUD operations. However, for regulated ML deployments, the compliance benefits often outweigh the engineering overhead.

**Best for:** Regulated industries (financial services, healthcare, government) where complete audit trails, temporal queries, and EU AI Act Article 12 automatic logging are non-negotiable requirements.

**Trade-offs:**
- (+) Complete, immutable audit trail -- every change is permanently recorded
- (+) Temporal queries ("what models were in production on 2025-12-01?") are trivial
- (+) Natural fit for EU AI Act Article 12 automatic event logging
- (+) Events can be replayed to build new read models without schema migration
- (+) Enables AI-powered analytics on change patterns and degradation root cause analysis
- (-) Higher implementation complexity (event handlers, projectors, eventual consistency)
- (-) Read model projections must be maintained and can become stale
- (-) Debugging requires understanding event replay, not just reading a table
- (-) Storage grows linearly with every event (compaction strategies needed for long-lived systems)
- (-) More challenging for simple CRUD operations that don't benefit from event history

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| EU AI Act Article 12 | The event store IS the automatic event log required by the regulation -- no separate logging system needed |
| EU AI Act Annex IV | Compliance documentation snapshots are events; the full documentation history is preserved for 10-year retention |
| NIST AI RMF | Govern/Manage lifecycle events are first-class domain events with full provenance |
| ISO/IEC 42001 | AI management system audit trail directly maps to the event store |
| MLflow REST API 2.0 | Read projections expose an MLflow-compatible API surface; events map to MLflow stage transitions |
| OpenTelemetry | Serving events emit OTel-compatible trace and metric signals |
| Open Inference Protocol | Endpoint deployment/rollback events track V2 protocol configuration changes |

---

## Event Store (Source of Truth)

```sql
-- The single source of truth: an append-only event log
CREATE TABLE events (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Aggregate identification
    aggregate_type VARCHAR(100) NOT NULL,  -- 'registered_model', 'model_version', 'serving_endpoint', 'drift_monitor'
    aggregate_id UUID NOT NULL,            -- ID of the entity this event belongs to
    -- Event metadata
    event_type VARCHAR(200) NOT NULL,      -- e.g., 'model.registered', 'version.stage_changed', 'endpoint.variant_traffic_updated'
    event_version INTEGER NOT NULL DEFAULT 1,  -- schema version of this event type
    sequence_number BIGINT NOT NULL,       -- monotonically increasing per aggregate
    -- Event payload
    payload JSONB NOT NULL,               -- event-specific data (see examples below)
    metadata JSONB DEFAULT '{}',          -- cross-cutting concerns: correlation_id, causation_id, user_agent
    -- Actor
    actor_id UUID,                        -- user or service principal who caused this event
    actor_type VARCHAR(50),               -- 'user', 'service_principal', 'system', 'ai_agent'
    -- Timestamps
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when the event happened in the domain
    recorded_at TIMESTAMPTZ NOT NULL DEFAULT now(),  -- when the event was persisted (wall clock)
    -- Optimistic concurrency
    UNIQUE(aggregate_type, aggregate_id, sequence_number)
);

-- Primary query: replay all events for an aggregate in order
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_number);

-- Query: find all events of a type across aggregates (e.g., all stage transitions)
CREATE INDEX idx_events_type ON events(event_type, occurred_at);

-- Query: find all events by actor (audit: "what did user X do?")
CREATE INDEX idx_events_actor ON events(actor_id, occurred_at);

-- Query: time-range scans for compliance reporting
CREATE INDEX idx_events_occurred ON events(occurred_at);

-- Partition by month for efficient time-range queries and archival
-- In production: CREATE TABLE events (...) PARTITION BY RANGE (occurred_at);
```

### Event Type Catalogue

```sql
-- Example event payloads (stored in the `payload` JSONB column):

-- event_type: 'model.registered'
-- {
--   "name": "fraud-detector",
--   "organization_id": "org-uuid",
--   "description": "Credit card fraud detection model",
--   "risk_classification": "high",
--   "intended_purpose": "Real-time fraud scoring for payment transactions",
--   "owner_id": "user-uuid",
--   "tags": {"team": "risk-engineering", "domain": "payments"}
-- }

-- event_type: 'model.metadata_updated'
-- {
--   "changes": {"description": {"old": "...", "new": "..."}, "risk_classification": {"old": "limited", "new": "high"}}
-- }

-- event_type: 'version.created'
-- {
--   "model_id": "model-uuid",
--   "version": 3,
--   "artifact_uri": "s3://bucket/models/fraud-detector/v3",
--   "artifact_format": "onnx",
--   "artifact_hash": "sha256:abc123...",
--   "framework": "pytorch",
--   "framework_version": "2.3.0",
--   "run_id": "mlflow-run-id",
--   "input_schema": {"type": "tensor", "shape": [-1, 128]},
--   "output_schema": {"type": "tensor", "shape": [-1, 1]}
-- }

-- event_type: 'version.stage_changed'
-- {
--   "model_id": "model-uuid",
--   "version": 3,
--   "from_stage": "staging",
--   "to_stage": "production",
--   "comment": "Approved after shadow testing: +2.1% precision, no latency regression",
--   "approval_checklist": {"unit_tests": true, "integration_tests": true, "shadow_test": true, "compliance_review": true}
-- }

-- event_type: 'version.alias_assigned'
-- {
--   "model_id": "model-uuid",
--   "alias": "champion",
--   "version": 3,
--   "previous_version": 2
-- }

-- event_type: 'endpoint.created'
-- {
--   "name": "fraud-scorer-prod",
--   "protocol": "v2",
--   "compute_type": "kubernetes",
--   "min_replicas": 2,
--   "max_replicas": 20
-- }

-- event_type: 'endpoint.variant_deployed'
-- {
--   "endpoint_id": "endpoint-uuid",
--   "variant_name": "control",
--   "version_id": "version-uuid",
--   "traffic_percentage": 90.0,
--   "runtime": "kserve",
--   "instance_type": "gpu.a10.1x"
-- }

-- event_type: 'endpoint.variant_traffic_updated'
-- {
--   "endpoint_id": "endpoint-uuid",
--   "traffic_split": {"control": 80.0, "treatment-a": 20.0},
--   "reason": "Starting A/B experiment: new feature engineering pipeline"
-- }

-- event_type: 'drift.detected'
-- {
--   "endpoint_id": "endpoint-uuid",
--   "variant_id": "variant-uuid",
--   "drift_type": "prediction_drift",
--   "drift_score": 0.35,
--   "threshold": 0.20,
--   "severity": "critical",
--   "feature_scores": {"feature_a": 0.42, "feature_b": 0.31, "feature_c": 0.05}
-- }

-- event_type: 'endpoint.rollback_triggered'
-- {
--   "endpoint_id": "endpoint-uuid",
--   "from_version_id": "version-uuid-new",
--   "to_version_id": "version-uuid-old",
--   "trigger": "automatic",
--   "reason": "Compound degradation: PSI 0.35 + revenue drop -4.2% over 2 hours"
-- }

-- event_type: 'compliance.document_submitted'
-- {
--   "model_id": "model-uuid",
--   "annex_iv_section": 3,
--   "section_name": "Training and validation data",
--   "document_uri": "s3://docs/fraud-detector/annex-iv/section-3.pdf",
--   "framework": "eu_ai_act"
-- }

-- event_type: 'compliance.document_approved'
-- {
--   "document_id": "doc-uuid",
--   "reviewer_id": "user-uuid",
--   "comment": "Dataset provenance verified against data governance records"
-- }

-- event_type: 'llm.prompt_version_created'
-- {
--   "model_id": "model-uuid",
--   "prompt_version": 5,
--   "template": "Given the transaction details: {{transaction}}, assess fraud risk...",
--   "variables": ["transaction"],
--   "system_prompt": "You are a fraud risk assessment agent..."
-- }

-- event_type: 'llm.adapter_registered'
-- {
--   "version_id": "version-uuid",
--   "adapter_type": "lora",
--   "adapter_uri": "s3://adapters/fraud-lora-v2",
--   "base_model": "meta-llama/Llama-3-70B",
--   "rank": 16,
--   "target_modules": ["q_proj", "v_proj"]
-- }
```

---

## Read Model Projections (Materialised Views)

These tables are rebuilt from the event store. They can be dropped and reconstructed at any time by replaying events.

```sql
-- Projection: current state of all registered models
CREATE TABLE projection_registered_models (
    id UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID,
    risk_classification VARCHAR(50),
    intended_purpose TEXT,
    tags JSONB DEFAULT '{}',
    latest_version INTEGER DEFAULT 0,
    -- Derived counts
    version_count INTEGER DEFAULT 0,
    production_version_id UUID,  -- currently promoted version
    -- Timestamps from events
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    -- Projection bookkeeping
    last_event_sequence BIGINT NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_models_org ON projection_registered_models(organization_id);
CREATE INDEX idx_proj_models_name ON projection_registered_models(organization_id, name);

-- Projection: current state of all model versions
CREATE TABLE projection_model_versions (
    id UUID PRIMARY KEY,
    model_id UUID NOT NULL,
    version INTEGER NOT NULL,
    description TEXT,
    artifact_uri TEXT,
    artifact_format VARCHAR(50),
    artifact_hash VARCHAR(128),
    framework VARCHAR(100),
    framework_version VARCHAR(50),
    input_schema JSONB,
    output_schema JSONB,
    stage VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL,
    run_id VARCHAR(255),
    source_uri TEXT,
    tags JSONB DEFAULT '{}',
    metrics JSONB DEFAULT '{}',
    parameters JSONB DEFAULT '{}',
    created_by UUID,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    last_event_sequence BIGINT NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_versions_model ON projection_model_versions(model_id);
CREATE INDEX idx_proj_versions_stage ON projection_model_versions(stage);

-- Projection: current model aliases
CREATE TABLE projection_model_aliases (
    model_id UUID NOT NULL,
    alias VARCHAR(255) NOT NULL,
    version_id UUID NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (model_id, alias)
);

-- Projection: current serving endpoints with traffic split
CREATE TABLE projection_serving_endpoints (
    id UUID PRIMARY KEY,
    organization_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    protocol VARCHAR(50),
    status VARCHAR(50),
    endpoint_url TEXT,
    -- Denormalised traffic split for fast API responses
    traffic_split JSONB DEFAULT '{}',  -- {"control": {"version_id": "...", "percentage": 80}, "treatment": {...}}
    variant_count INTEGER DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL,
    last_event_sequence BIGINT NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_endpoints_org ON projection_serving_endpoints(organization_id);

-- Projection: compliance status per model
CREATE TABLE projection_compliance_status (
    model_id UUID PRIMARY KEY,
    framework VARCHAR(100) NOT NULL,
    total_sections INTEGER NOT NULL,
    completed_sections INTEGER DEFAULT 0,
    approved_sections INTEGER DEFAULT 0,
    -- Per-section status (denormalised for dashboard)
    section_status JSONB DEFAULT '{}',
    -- Example: {"1": {"status": "approved", "reviewed_at": "..."}, "2": {"status": "draft"}, ...}
    overall_status VARCHAR(50) DEFAULT 'incomplete',  -- 'incomplete', 'in_review', 'compliant'
    last_updated TIMESTAMPTZ,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Snapshot Tables (Point-in-Time Reconstruction)

```sql
-- Periodic snapshots for fast reconstruction without full replay
CREATE TABLE aggregate_snapshots (
    aggregate_type VARCHAR(100) NOT NULL,
    aggregate_id UUID NOT NULL,
    snapshot_at TIMESTAMPTZ NOT NULL,
    sequence_number BIGINT NOT NULL,  -- event sequence at snapshot time
    state JSONB NOT NULL,            -- full aggregate state at this point
    PRIMARY KEY (aggregate_type, aggregate_id, snapshot_at)
);

-- Keep the last N snapshots per aggregate for fast startup
CREATE INDEX idx_snapshots_latest ON aggregate_snapshots(aggregate_type, aggregate_id, sequence_number DESC);
```

---

## Supporting Tables (Mutable, Non-Event-Sourced)

```sql
-- These tables store operational data that doesn't benefit from event sourcing

CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    slug VARCHAR(255) NOT NULL UNIQUE,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(255),
    role VARCHAR(50) NOT NULL DEFAULT 'member',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);

-- Time-series metrics (append-only by nature, not event-sourced)
CREATE TABLE serving_metrics (
    endpoint_id UUID NOT NULL,
    variant_name VARCHAR(255) NOT NULL,
    bucket TIMESTAMPTZ NOT NULL,
    request_count BIGINT DEFAULT 0,
    error_count BIGINT DEFAULT 0,
    latency_p50_ms NUMERIC(10,2),
    latency_p95_ms NUMERIC(10,2),
    latency_p99_ms NUMERIC(10,2),
    prediction_mean DOUBLE PRECISION,
    prediction_stddev DOUBLE PRECISION,
    PRIMARY KEY (endpoint_id, variant_name, bucket)
);

-- Webhook configuration (mutable operational config)
CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    event_types TEXT[] NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Replay model state at a specific point in time

```sql
-- "What was the state of model 'fraud-detector' on 2025-12-01?"
SELECT payload
FROM events
WHERE aggregate_type = 'registered_model'
  AND aggregate_id = '...'
  AND occurred_at <= '2025-12-01T23:59:59Z'
ORDER BY sequence_number ASC;
-- Application code replays these events to reconstruct state
```

### Find all stage transitions for compliance audit

```sql
-- "Show all production promotions in the last 90 days with who approved them"
SELECT
    e.aggregate_id AS model_version_id,
    e.payload->>'from_stage' AS from_stage,
    e.payload->>'to_stage' AS to_stage,
    e.payload->>'comment' AS comment,
    u.email AS approved_by,
    e.occurred_at
FROM events e
LEFT JOIN users u ON e.actor_id = u.id
WHERE e.event_type = 'version.stage_changed'
  AND e.payload->>'to_stage' = 'production'
  AND e.occurred_at >= now() - INTERVAL '90 days'
ORDER BY e.occurred_at DESC;
```

### Correlate drift with upstream changes (root cause analysis)

```sql
-- "What changed before the drift alert on endpoint X?"
WITH drift_event AS (
    SELECT occurred_at
    FROM events
    WHERE event_type = 'drift.detected'
      AND aggregate_id = 'endpoint-uuid'
    ORDER BY occurred_at DESC
    LIMIT 1
)
SELECT event_type, payload, occurred_at, actor_id
FROM events, drift_event
WHERE occurred_at BETWEEN drift_event.occurred_at - INTERVAL '24 hours'
  AND drift_event.occurred_at
  AND event_type IN (
    'version.created', 'endpoint.variant_deployed',
    'endpoint.variant_traffic_updated', 'version.stage_changed'
  )
ORDER BY occurred_at ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by occurred_at) |
| Snapshots | 1 | aggregate_snapshots |
| Read Projections | 5 | projection_registered_models, projection_model_versions, projection_model_aliases, projection_serving_endpoints, projection_compliance_status |
| Supporting (Mutable) | 4 | organizations, users, serving_metrics, webhooks |
| **Total** | **11** | Plus application-level event handlers and projectors |

---

## Key Design Decisions

1. **Single events table as the source of truth** -- all domain state changes flow through one append-only table. This dramatically simplifies audit compliance (EU AI Act Article 12) because the event log IS the automatic event log the regulation requires.

2. **JSONB payload with event_type discriminator** -- rather than separate tables per event type, a single JSONB payload column with a typed event_type field keeps the schema stable while allowing new event types without migration. The event_version field supports payload schema evolution.

3. **Read projections are disposable** -- projection tables can be dropped and rebuilt from the event store at any time. This enables adding new query patterns (e.g., a new dashboard view) without altering the source of truth.

4. **Aggregate-based partitioning** -- events are grouped by (aggregate_type, aggregate_id) with a monotonic sequence_number, providing optimistic concurrency control and enabling efficient per-entity replay.

5. **Temporal queries are first-class** -- answering "what was true at time T?" requires filtering events by occurred_at and replaying. Periodic snapshots (aggregate_snapshots) prevent full replay from genesis for long-lived aggregates.

6. **Serving metrics are NOT event-sourced** -- time-series data (request counts, latencies) is naturally append-only and high-volume. Storing each metric data point as a domain event would bloat the event store. These stay in a dedicated time-series table.

7. **Mutable operational tables coexist** -- user accounts, webhook configurations, and org settings are standard CRUD entities. Not everything benefits from event sourcing; the pattern is applied where audit and temporal value justifies the complexity.

8. **Event correlation for root cause analysis** -- the metadata JSONB field stores correlation_id and causation_id, enabling AI-powered analysis to trace chains of related events (e.g., "this drift alert was caused by this deployment, which was triggered by this stage transition").
