# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: ML Model Registry & Serving · Created: 2026-05-20

## Philosophy

This approach follows a traditional normalized relational design where every domain concept gets its own table with explicit foreign key relationships. It mirrors the proven patterns used by MLflow's backend store and Kubeflow Model Registry, extending them with first-class governance, serving, and LLM lifecycle tables. Every relationship is explicit, every constraint is enforced at the database level, and every query can be expressed as a standard SQL JOIN.

The design prioritises data integrity and query flexibility over write throughput. It is well-suited for teams that value strong schema guarantees, have SQL expertise, and need complex cross-entity queries (e.g., "find all models trained on dataset X that are currently serving in production with drift alerts"). This is the most familiar pattern for teams coming from MLflow or SageMaker backgrounds.

**Best for:** Teams prioritising data integrity, complex ad-hoc queries, and regulatory compliance in a single-cloud or on-premises deployment.

**Trade-offs:**
- (+) Strong referential integrity enforced at database level
- (+) Familiar SQL patterns; easy to reason about and debug
- (+) Straightforward mapping to REST API resources
- (+) Excellent tooling support (ORMs, migration tools, query builders)
- (-) Schema migrations required for every new entity or relationship
- (-) Many JOIN operations for common read paths (model + version + tags + metrics + lineage)
- (-) Jurisdiction-specific or framework-specific fields require ALTER TABLE or new tables
- (-) Higher table count increases operational complexity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MLflow REST API 2.0 | Table structure mirrors MLflow's registered_models, model_versions, and tags tables for API compatibility |
| ONNX | Model artifacts reference ONNX format via `artifact_format` enum; ONNX opset version tracked |
| EU AI Act Annex IV | Dedicated `compliance_documents` and `compliance_checklists` tables map to the 9 mandatory documentation categories |
| NIST AI RMF | Risk classification fields on `registered_models` align with NIST Govern/Map/Measure/Manage functions |
| ISO/IEC 42001 | AI management system metadata captured in `model_governance` table |
| OpenTelemetry | Serving metrics table schema aligns with OTel metric naming conventions |
| Open Inference Protocol (KServe V2) | Serving endpoint configuration stores V2 protocol settings |
| ISO 3166-1 | Jurisdiction codes use ISO 3166-1 alpha-2 standard |
| OAuth 2.0 / OIDC | User and service principal authentication references stored in `users` table |

---

## Core Registry Tables

### Organizations and Access Control

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    slug VARCHAR(255) NOT NULL UNIQUE,
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(255),
    role VARCHAR(50) NOT NULL DEFAULT 'member',  -- 'admin', 'member', 'viewer'
    auth_provider VARCHAR(50),  -- 'oidc', 'saml', 'api_key'
    auth_subject VARCHAR(512),  -- external identity provider subject ID
    is_service_principal BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);

CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_email ON users(email);

CREATE TABLE teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, name)
);

CREATE TABLE team_members (
    team_id UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role VARCHAR(50) NOT NULL DEFAULT 'member',  -- 'lead', 'member'
    PRIMARY KEY (team_id, user_id)
);

CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    key_hash VARCHAR(128) NOT NULL,  -- bcrypt hash of the API key
    name VARCHAR(255) NOT NULL,
    scopes TEXT[] DEFAULT '{}',  -- e.g., '{registry:read, registry:write, serving:manage}'
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);
```

### Registered Models

```sql
CREATE TABLE registered_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID REFERENCES users(id),
    team_id UUID REFERENCES teams(id),
    -- AI governance fields (NIST AI RMF / ISO 42001)
    risk_classification VARCHAR(50),  -- 'minimal', 'limited', 'high', 'unacceptable' (EU AI Act)
    intended_purpose TEXT,
    ai_system_type VARCHAR(100),  -- 'classification', 'regression', 'generation', 'recommendation'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, name)
);

CREATE INDEX idx_registered_models_org ON registered_models(organization_id);
CREATE INDEX idx_registered_models_owner ON registered_models(owner_id);
CREATE INDEX idx_registered_models_name ON registered_models(organization_id, name);

CREATE TABLE registered_model_tags (
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    key VARCHAR(255) NOT NULL,
    value VARCHAR(5000),
    PRIMARY KEY (model_id, key)
);
```

### Model Versions

```sql
CREATE TABLE model_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    description TEXT,
    -- Artifact location
    artifact_uri TEXT NOT NULL,  -- e.g., 's3://bucket/models/my-model/v3'
    artifact_format VARCHAR(50) NOT NULL DEFAULT 'mlmodel',  -- 'mlmodel', 'onnx', 'torchscript', 'savedmodel', 'custom'
    artifact_size_bytes BIGINT,
    artifact_hash VARCHAR(128),  -- SHA-256 of the model artifact for integrity verification
    -- Model metadata
    framework VARCHAR(100),  -- 'pytorch', 'tensorflow', 'sklearn', 'xgboost', 'huggingface', 'custom'
    framework_version VARCHAR(50),
    python_version VARCHAR(20),
    input_schema JSONB,  -- MLflow-style input signature
    output_schema JSONB, -- MLflow-style output signature
    -- Lifecycle
    stage VARCHAR(50) NOT NULL DEFAULT 'development',  -- 'development', 'staging', 'production', 'archived'
    status VARCHAR(50) NOT NULL DEFAULT 'pending_review',  -- 'pending_review', 'approved', 'rejected'
    -- Lineage
    run_id VARCHAR(255),  -- link to experiment tracking run
    source_uri TEXT,  -- Git commit URL or notebook URI
    dataset_version_id UUID REFERENCES dataset_versions(id),
    -- Creator
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(model_id, version)
);

CREATE INDEX idx_model_versions_model ON model_versions(model_id);
CREATE INDEX idx_model_versions_stage ON model_versions(stage);
CREATE INDEX idx_model_versions_status ON model_versions(status);
CREATE INDEX idx_model_versions_created ON model_versions(created_at DESC);

CREATE TABLE model_version_tags (
    version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    key VARCHAR(255) NOT NULL,
    value VARCHAR(5000),
    PRIMARY KEY (version_id, key)
);
```

### Model Aliases

```sql
-- Mutable named references decoupling consumers from version numbers
-- e.g., 'champion', 'challenger', 'shadow'
CREATE TABLE model_aliases (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    alias VARCHAR(255) NOT NULL,  -- 'champion', 'challenger', 'shadow', 'canary'
    version_id UUID NOT NULL REFERENCES model_versions(id),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(model_id, alias)
);

CREATE INDEX idx_model_aliases_version ON model_aliases(version_id);
```

### Model Metrics and Evaluation

```sql
CREATE TABLE model_version_metrics (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    key VARCHAR(255) NOT NULL,  -- 'accuracy', 'f1_score', 'latency_p99', 'rmse'
    value DOUBLE PRECISION NOT NULL,
    step INTEGER,  -- for metrics logged over training steps
    timestamp TIMESTAMPTZ NOT NULL DEFAULT now(),
    dataset_name VARCHAR(255),  -- which evaluation dataset produced this metric
    UNIQUE(version_id, key, step)
);

CREATE INDEX idx_version_metrics_version ON model_version_metrics(version_id);
CREATE INDEX idx_version_metrics_key ON model_version_metrics(key);

CREATE TABLE model_version_parameters (
    version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    key VARCHAR(255) NOT NULL,
    value TEXT NOT NULL,
    PRIMARY KEY (version_id, key)
);
```

---

## Dataset and Lineage Tables

```sql
CREATE TABLE datasets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    source_type VARCHAR(50),  -- 'delta_table', 's3', 'gcs', 'bigquery', 'postgres'
    source_uri TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, name)
);

CREATE TABLE dataset_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    uri TEXT NOT NULL,
    hash VARCHAR(128),  -- content hash for reproducibility
    num_rows BIGINT,
    num_features INTEGER,
    schema_definition JSONB,  -- column names, types, statistics
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(dataset_id, version)
);

CREATE TABLE lineage_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type VARCHAR(50) NOT NULL,  -- 'dataset_version', 'model_version', 'experiment_run', 'code_commit'
    source_id UUID NOT NULL,
    target_type VARCHAR(50) NOT NULL,  -- 'model_version', 'serving_endpoint'
    target_id UUID NOT NULL,
    relationship VARCHAR(50) NOT NULL,  -- 'trained_on', 'evaluated_on', 'derived_from', 'deployed_to'
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lineage_source ON lineage_links(source_type, source_id);
CREATE INDEX idx_lineage_target ON lineage_links(target_type, target_id);
```

---

## Serving and Traffic Management Tables

```sql
CREATE TABLE serving_endpoints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    -- Endpoint configuration
    protocol VARCHAR(50) NOT NULL DEFAULT 'v2',  -- 'v2' (Open Inference Protocol), 'rest', 'grpc'
    endpoint_url TEXT,
    -- Infrastructure
    compute_type VARCHAR(50),  -- 'kubernetes', 'serverless', 'gpu_cluster'
    compute_config JSONB,  -- cluster-specific settings (replicas, GPU type, memory)
    -- Status
    status VARCHAR(50) NOT NULL DEFAULT 'creating',  -- 'creating', 'ready', 'updating', 'failed', 'deleting'
    -- Autoscaling
    min_replicas INTEGER DEFAULT 0,
    max_replicas INTEGER DEFAULT 10,
    scale_to_zero BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, name)
);

CREATE INDEX idx_serving_endpoints_org ON serving_endpoints(organization_id);

CREATE TABLE serving_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES serving_endpoints(id) ON DELETE CASCADE,
    version_id UUID NOT NULL REFERENCES model_versions(id),
    variant_name VARCHAR(255) NOT NULL,  -- 'control', 'treatment-a', 'shadow'
    -- Traffic configuration
    traffic_percentage NUMERIC(5,2) NOT NULL DEFAULT 0,  -- 0.00 to 100.00
    is_shadow BOOLEAN NOT NULL DEFAULT FALSE,  -- shadow traffic (no response to caller)
    -- Runtime config
    instance_type VARCHAR(100),
    runtime VARCHAR(100),  -- 'kserve', 'seldon', 'triton', 'bentoml', 'torchserve'
    runtime_config JSONB DEFAULT '{}',
    -- Transformer pipeline
    pre_processor_uri TEXT,
    post_processor_uri TEXT,
    -- Status
    status VARCHAR(50) NOT NULL DEFAULT 'creating',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(endpoint_id, variant_name)
);

CREATE INDEX idx_serving_variants_endpoint ON serving_variants(endpoint_id);
CREATE INDEX idx_serving_variants_version ON serving_variants(version_id);

CREATE TABLE ab_experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES serving_endpoints(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    hypothesis TEXT,
    -- Variants participating in this experiment
    control_variant_id UUID NOT NULL REFERENCES serving_variants(id),
    -- Configuration
    primary_metric VARCHAR(255) NOT NULL,  -- business metric to optimise
    significance_level NUMERIC(4,3) DEFAULT 0.050,  -- alpha, default 5%
    minimum_sample_size BIGINT,
    -- Lifecycle
    status VARCHAR(50) NOT NULL DEFAULT 'draft',  -- 'draft', 'running', 'concluded', 'cancelled'
    started_at TIMESTAMPTZ,
    concluded_at TIMESTAMPTZ,
    conclusion TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ab_experiment_variants (
    experiment_id UUID NOT NULL REFERENCES ab_experiments(id) ON DELETE CASCADE,
    variant_id UUID NOT NULL REFERENCES serving_variants(id),
    PRIMARY KEY (experiment_id, variant_id)
);
```

---

## Monitoring and Drift Tables

```sql
CREATE TABLE drift_monitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES serving_endpoints(id) ON DELETE CASCADE,
    monitor_type VARCHAR(50) NOT NULL,  -- 'data_drift', 'prediction_drift', 'performance_drift'
    reference_dataset_id UUID REFERENCES dataset_versions(id),
    -- Thresholds
    alert_threshold NUMERIC(8,4),  -- e.g., PSI > 0.2
    critical_threshold NUMERIC(8,4),
    check_interval_minutes INTEGER DEFAULT 60,
    -- Status
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    last_check_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE drift_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    monitor_id UUID NOT NULL REFERENCES drift_monitors(id) ON DELETE CASCADE,
    severity VARCHAR(50) NOT NULL,  -- 'warning', 'critical'
    drift_score NUMERIC(8,4),
    details JSONB NOT NULL,  -- feature-level drift scores, distribution comparisons
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drift_alerts_monitor ON drift_alerts(monitor_id);
CREATE INDEX idx_drift_alerts_created ON drift_alerts(created_at DESC);

CREATE TABLE serving_metrics (
    endpoint_id UUID NOT NULL REFERENCES serving_endpoints(id) ON DELETE CASCADE,
    variant_id UUID NOT NULL REFERENCES serving_variants(id),
    -- Time bucket (1-minute granularity)
    bucket TIMESTAMPTZ NOT NULL,
    -- Request metrics (OTel-aligned names)
    request_count BIGINT DEFAULT 0,
    error_count BIGINT DEFAULT 0,
    latency_p50_ms NUMERIC(10,2),
    latency_p95_ms NUMERIC(10,2),
    latency_p99_ms NUMERIC(10,2),
    -- Prediction distribution
    prediction_mean DOUBLE PRECISION,
    prediction_stddev DOUBLE PRECISION,
    PRIMARY KEY (endpoint_id, variant_id, bucket)
);

-- Partition by time for efficient range queries
-- CREATE TABLE serving_metrics ... PARTITION BY RANGE (bucket);
```

---

## Governance and Compliance Tables

```sql
-- EU AI Act Annex IV documentation tracking
CREATE TABLE compliance_documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version_id UUID REFERENCES model_versions(id),
    -- Annex IV section mapping
    annex_iv_section INTEGER NOT NULL,  -- 1-9 per Annex IV categories
    section_name VARCHAR(255) NOT NULL,
    -- Content
    document_uri TEXT,  -- link to stored document
    content_summary TEXT,
    -- Review
    status VARCHAR(50) NOT NULL DEFAULT 'draft',  -- 'draft', 'review', 'approved', 'expired'
    reviewed_by UUID REFERENCES users(id),
    reviewed_at TIMESTAMPTZ,
    -- Validity
    valid_from TIMESTAMPTZ,
    valid_until TIMESTAMPTZ,  -- 10-year retention per Article 18
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_docs_model ON compliance_documents(model_id);

CREATE TABLE compliance_checklists (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    framework VARCHAR(100) NOT NULL,  -- 'eu_ai_act', 'nist_ai_rmf', 'iso_42001'
    checklist_item VARCHAR(500) NOT NULL,
    is_satisfied BOOLEAN NOT NULL DEFAULT FALSE,
    evidence_uri TEXT,
    notes TEXT,
    checked_by UUID REFERENCES users(id),
    checked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_checklists_model ON compliance_checklists(model_id);

-- Stage transition approvals (audit trail)
CREATE TABLE stage_transitions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id UUID NOT NULL REFERENCES model_versions(id),
    from_stage VARCHAR(50),
    to_stage VARCHAR(50) NOT NULL,
    approved_by UUID REFERENCES users(id),
    comment TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_stage_transitions_version ON stage_transitions(version_id);
CREATE INDEX idx_stage_transitions_created ON stage_transitions(created_at DESC);
```

---

## LLM Lifecycle Tables

```sql
CREATE TABLE prompt_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    name VARCHAR(255),
    template TEXT NOT NULL,
    variables TEXT[],  -- list of template variable names
    system_prompt TEXT,
    temperature NUMERIC(3,2),
    max_tokens INTEGER,
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(model_id, version)
);

CREATE TABLE adapter_weights (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    adapter_type VARCHAR(50) NOT NULL,  -- 'lora', 'qlora', 'prefix_tuning', 'full_finetune'
    adapter_uri TEXT NOT NULL,
    base_model_name VARCHAR(255),  -- e.g., 'meta-llama/Llama-3-70B'
    rank INTEGER,  -- LoRA rank
    target_modules TEXT[],  -- e.g., '{q_proj, v_proj}'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE quantisation_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    version_id UUID NOT NULL REFERENCES model_versions(id) ON DELETE CASCADE,
    method VARCHAR(50) NOT NULL,  -- 'gptq', 'awq', 'gguf', 'fp16', 'int8', 'int4'
    bits INTEGER,
    artifact_uri TEXT NOT NULL,
    artifact_size_bytes BIGINT,
    benchmark_throughput DOUBLE PRECISION,  -- tokens/sec
    benchmark_latency_ms DOUBLE PRECISION,  -- per-token latency
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Webhook and Event Tables

```sql
CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    secret_hash VARCHAR(128),  -- HMAC signing secret hash
    events TEXT[] NOT NULL,  -- '{model.registered, version.created, version.stage_changed, endpoint.deployed}'
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhook_deliveries (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    webhook_id UUID NOT NULL REFERENCES webhooks(id) ON DELETE CASCADE,
    event_type VARCHAR(100) NOT NULL,
    payload JSONB NOT NULL,
    response_status INTEGER,
    response_body TEXT,
    delivered_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    retry_count INTEGER DEFAULT 0
);

CREATE INDEX idx_webhook_deliveries_webhook ON webhook_deliveries(webhook_id);
CREATE INDEX idx_webhook_deliveries_event ON webhook_deliveries(event_type);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Access Control | 5 | organizations, users, teams, team_members, api_keys |
| Registry Core | 5 | registered_models, registered_model_tags, model_versions, model_version_tags, model_aliases |
| Metrics & Evaluation | 2 | model_version_metrics, model_version_parameters |
| Datasets & Lineage | 3 | datasets, dataset_versions, lineage_links |
| Serving & Traffic | 5 | serving_endpoints, serving_variants, ab_experiments, ab_experiment_variants, serving_metrics |
| Monitoring | 2 | drift_monitors, drift_alerts |
| Governance | 3 | compliance_documents, compliance_checklists, stage_transitions |
| LLM Lifecycle | 3 | prompt_versions, adapter_weights, quantisation_variants |
| Webhooks | 2 | webhooks, webhook_deliveries |
| **Total** | **30** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** -- enables distributed ID generation without coordination, important for multi-region deployments and Kubernetes-native architecture.

2. **Model aliases as a separate table** -- follows MLflow's pattern of mutable named references (champion/challenger) decoupled from version numbers, allowing atomic alias swaps without modifying the version record.

3. **Explicit lineage_links junction table** -- uses a polymorphic source/target pattern rather than embedding lineage in model_versions, enabling flexible graph-style lineage queries (dataset -> model -> endpoint) without schema changes.

4. **EU AI Act Annex IV as structured records** -- each of the 9 Annex IV sections is a separate row in compliance_documents, enabling per-section review workflows and status tracking rather than a single monolithic document.

5. **Serving variants as separate entities from model versions** -- a single model version can be deployed as multiple variants across different endpoints with different traffic percentages, runtime configurations, and pre/post processors.

6. **Time-bucketed serving_metrics with composite primary key** -- uses (endpoint_id, variant_id, bucket) as the primary key for efficient time-range queries on serving performance, compatible with PostgreSQL table partitioning.

7. **Separate LLM lifecycle tables** -- prompt_versions, adapter_weights, and quantisation_variants are first-class tables rather than metadata tags, enabling typed queries and constraints specific to LLM workflows.

8. **Stage transitions as an append-only audit log** -- every stage change (development -> staging -> production) is recorded as a new row with the approver and comment, providing a complete compliance-ready audit trail.
