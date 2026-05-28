# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: ML Model Registry & Serving · Created: 2026-05-20

## Philosophy

This approach uses a lean set of relational tables for core entities and relationships, with JSONB columns handling the highly variable, domain-specific, and framework-specific metadata that differs across ML frameworks, deployment targets, LLM configurations, and regulatory jurisdictions. The relational skeleton provides referential integrity and efficient JOINs for the most common queries, while JSONB columns absorb the long tail of metadata without requiring schema migrations.

This is the pragmatic middle ground between the rigidity of full normalisation (Suggestion 1) and the complexity of event sourcing (Suggestion 2). It is inspired by how modern SaaS platforms like Stripe, Shopify, and GitHub model extensible entities -- a fixed set of typed columns for the fields you always query, plus a `metadata` or `properties` JSONB column for everything else. In the ML registry context, this means model versions have typed columns for `framework`, `stage`, and `artifact_uri` (which you filter on constantly), but framework-specific fields (PyTorch checkpoint format, HuggingFace model card fields, ONNX opset version) live in JSONB.

This pattern is particularly well-suited for a multi-framework, multi-cloud registry where the metadata shape varies wildly between a scikit-learn pickle, a PyTorch TorchScript model, a HuggingFace transformer, and an ONNX graph. Rather than creating separate tables or columns for each framework's metadata, a single `framework_metadata` JSONB column handles them all.

**Best for:** Rapid MVP development, multi-framework registries where metadata shapes vary widely, and teams that want relational integrity without the migration burden of full normalisation.

**Trade-offs:**
- (+) Fewer tables and columns; faster to develop and iterate on
- (+) New framework or jurisdiction metadata requires no schema migration
- (+) JSONB columns support GIN indexing for containment queries
- (+) Relational core enables standard JOINs for common access patterns
- (+) Natural fit for REST APIs that return flexible JSON responses
- (-) JSONB fields are not type-checked at the database level
- (-) Complex queries on nested JSONB fields are slower than indexed relational columns
- (-) Data quality depends on application-level validation (JSON Schema)
- (-) Reporting and analytics on JSONB fields requires extraction (JSONB operators, materialised views)
- (-) JSONB columns can become "junk drawers" without disciplined schema governance

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MLflow REST API 2.0 | Core relational fields align with MLflow's registered_models and model_versions; tags stored as JSONB |
| ONNX | ONNX-specific metadata (opset_version, ir_version, producer) stored in `framework_metadata` JSONB |
| EU AI Act Annex IV | Compliance metadata stored as structured JSONB with JSON Schema validation; flexible per-jurisdiction |
| NIST AI RMF | Governance fields as JSONB properties on registered_models, validated against NIST categories |
| Open Inference Protocol | Serving configuration stored as JSONB, supporting V2 protocol fields and extensions |
| OpenTelemetry | Metric labels and custom dimensions stored as JSONB attributes |
| ISO 3166-1 | Jurisdiction codes validated at application level within JSONB governance metadata |
| Model Card Schema | Model card fields stored as structured JSONB, aligned with Google Model Card Toolkit JSON Schema |

---

## Core Registry Tables

```sql
CREATE TABLE organizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL UNIQUE,
    slug VARCHAR(255) NOT NULL UNIQUE,
    -- Flexible org-level settings: default compliance framework, cloud regions, etc.
    settings JSONB DEFAULT '{}',
    -- Example settings:
    -- {
    --   "default_compliance_framework": "eu_ai_act",
    --   "allowed_artifact_stores": ["s3://models-prod", "gs://models-prod"],
    --   "default_serving_runtime": "kserve",
    --   "jurisdictions": ["EU", "US", "UK"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    email VARCHAR(320) NOT NULL,
    display_name VARCHAR(255),
    role VARCHAR(50) NOT NULL DEFAULT 'member',
    is_service_principal BOOLEAN NOT NULL DEFAULT FALSE,
    -- Extensible user profile/preferences
    profile JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);

CREATE INDEX idx_users_org ON users(organization_id);
```

### Registered Models

```sql
CREATE TABLE registered_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID REFERENCES users(id),
    -- Typed fields for common queries
    risk_classification VARCHAR(50),  -- 'minimal', 'limited', 'high', 'unacceptable'
    ai_system_type VARCHAR(100),
    -- Flexible metadata: tags, governance fields, custom attributes
    tags JSONB DEFAULT '{}',
    -- Example tags: {"team": "risk-engineering", "domain": "payments", "cost_center": "CC-1234"}
    governance JSONB DEFAULT '{}',
    -- Example governance:
    -- {
    --   "intended_purpose": "Real-time fraud scoring for payment transactions",
    --   "nist_functions": ["govern", "map", "measure", "manage"],
    --   "iso_42001_scope": "Payment fraud detection subsystem",
    --   "data_protection_impact_assessment": true,
    --   "human_oversight_required": true,
    --   "jurisdictions": ["DE", "FR", "NL"]
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, name)
);

CREATE INDEX idx_models_org ON registered_models(organization_id);
CREATE INDEX idx_models_risk ON registered_models(risk_classification);
CREATE INDEX idx_models_tags ON registered_models USING GIN (tags);
CREATE INDEX idx_models_governance ON registered_models USING GIN (governance);
```

### Model Versions

```sql
CREATE TABLE model_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    description TEXT,
    -- Core typed fields (always queried, always present)
    artifact_uri TEXT NOT NULL,
    artifact_format VARCHAR(50) NOT NULL DEFAULT 'mlmodel',
    artifact_hash VARCHAR(128),
    artifact_size_bytes BIGINT,
    framework VARCHAR(100) NOT NULL,
    stage VARCHAR(50) NOT NULL DEFAULT 'development',
    status VARCHAR(50) NOT NULL DEFAULT 'pending_review',
    created_by UUID REFERENCES users(id),
    -- Lineage (typed for JOINs)
    run_id VARCHAR(255),
    source_uri TEXT,
    -- Framework-specific metadata (shape varies by framework)
    framework_metadata JSONB DEFAULT '{}',
    -- Example for PyTorch:
    -- {
    --   "framework_version": "2.3.0",
    --   "torch_dtype": "float16",
    --   "device_map": "auto",
    --   "checkpoint_format": "safetensors",
    --   "model_class": "TransformerForSequenceClassification"
    -- }
    -- Example for ONNX:
    -- {
    --   "opset_version": 18,
    --   "ir_version": 9,
    --   "producer_name": "pytorch",
    --   "producer_version": "2.3.0",
    --   "graph_input_shapes": {"input_ids": [-1, 512], "attention_mask": [-1, 512]}
    -- }
    -- Example for HuggingFace:
    -- {
    --   "model_type": "llama",
    --   "architecture": "LlamaForCausalLM",
    --   "tokenizer": "LlamaTokenizerFast",
    --   "vocab_size": 32000,
    --   "max_position_embeddings": 4096,
    --   "hf_model_id": "meta-llama/Llama-3-8B"
    -- }
    -- Input/output schema
    input_schema JSONB,
    output_schema JSONB,
    -- Tags and custom metadata
    tags JSONB DEFAULT '{}',
    -- Evaluation metrics (denormalised for fast read)
    metrics JSONB DEFAULT '{}',
    -- Example: {"accuracy": 0.943, "f1_score": 0.891, "latency_p99_ms": 12.4, "rmse": 0.032}
    -- Training parameters
    parameters JSONB DEFAULT '{}',
    -- Example: {"learning_rate": 0.0001, "epochs": 50, "batch_size": 64, "optimizer": "adamw"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(model_id, version)
);

CREATE INDEX idx_versions_model ON model_versions(model_id);
CREATE INDEX idx_versions_stage ON model_versions(stage);
CREATE INDEX idx_versions_framework ON model_versions(framework);
CREATE INDEX idx_versions_created ON model_versions(created_at DESC);
CREATE INDEX idx_versions_tags ON model_versions USING GIN (tags);
CREATE INDEX idx_versions_metrics ON model_versions USING GIN (metrics);
CREATE INDEX idx_versions_framework_meta ON model_versions USING GIN (framework_metadata);
```

### Model Aliases

```sql
CREATE TABLE model_aliases (
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    alias VARCHAR(255) NOT NULL,
    version_id UUID NOT NULL REFERENCES model_versions(id),
    updated_by UUID REFERENCES users(id),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (model_id, alias)
);
```

---

## Dataset and Lineage Tables

```sql
CREATE TABLE datasets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    source_type VARCHAR(50),
    source_uri TEXT,
    -- Flexible dataset metadata
    metadata JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "num_rows": 1500000,
    --   "num_features": 128,
    --   "schema": [{"name": "amount", "type": "float64"}, {"name": "merchant_id", "type": "string"}],
    --   "statistics": {"mean_amount": 45.67, "null_rate": 0.002},
    --   "provenance": {"source_system": "data_warehouse", "extraction_query": "SELECT ..."},
    --   "data_quality_score": 0.97
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, name)
);

CREATE TABLE dataset_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dataset_id UUID NOT NULL REFERENCES datasets(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    uri TEXT NOT NULL,
    hash VARCHAR(128),
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(dataset_id, version)
);

-- Polymorphic lineage using JSONB for flexible link metadata
CREATE TABLE lineage_links (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type VARCHAR(50) NOT NULL,
    source_id UUID NOT NULL,
    target_type VARCHAR(50) NOT NULL,
    target_id UUID NOT NULL,
    relationship VARCHAR(50) NOT NULL,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lineage_source ON lineage_links(source_type, source_id);
CREATE INDEX idx_lineage_target ON lineage_links(target_type, target_id);
```

---

## Serving and Traffic Management

```sql
CREATE TABLE serving_endpoints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    -- Core typed fields
    protocol VARCHAR(50) NOT NULL DEFAULT 'v2',
    status VARCHAR(50) NOT NULL DEFAULT 'creating',
    endpoint_url TEXT,
    -- All configuration as structured JSONB
    config JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "compute_type": "kubernetes",
    --   "min_replicas": 2,
    --   "max_replicas": 20,
    --   "scale_to_zero": false,
    --   "gpu_type": "nvidia-a10g",
    --   "memory_limit": "16Gi",
    --   "timeout_seconds": 30,
    --   "max_batch_size": 32,
    --   "kserve_config": {
    --     "predictor": {"model_format": "onnx", "runtime": "triton"},
    --     "transformer": {"image": "registry/preprocess:v2"}
    --   }
    -- }
    -- Current traffic split (denormalised for fast reads)
    traffic_split JSONB DEFAULT '{}',
    -- Example: {"control": {"version_id": "uuid", "percentage": 80}, "canary": {"version_id": "uuid", "percentage": 20}}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, name)
);

CREATE INDEX idx_endpoints_org ON serving_endpoints(organization_id);
CREATE INDEX idx_endpoints_status ON serving_endpoints(status);

CREATE TABLE serving_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES serving_endpoints(id) ON DELETE CASCADE,
    version_id UUID NOT NULL REFERENCES model_versions(id),
    variant_name VARCHAR(255) NOT NULL,
    traffic_percentage NUMERIC(5,2) NOT NULL DEFAULT 0,
    is_shadow BOOLEAN NOT NULL DEFAULT FALSE,
    status VARCHAR(50) NOT NULL DEFAULT 'creating',
    -- Variant-specific runtime config
    config JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "runtime": "kserve",
    --   "instance_type": "gpu.a10.1x",
    --   "replicas": 3,
    --   "pre_processor_uri": "s3://transforms/tokenizer-v2",
    --   "post_processor_uri": "s3://transforms/label-decoder-v1",
    --   "environment_variables": {"MAX_BATCH_SIZE": "32"}
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(endpoint_id, variant_name)
);

CREATE TABLE ab_experiments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES serving_endpoints(id) ON DELETE CASCADE,
    name VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft',
    -- Experiment configuration as JSONB (flexible per experiment type)
    config JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "hypothesis": "New feature engineering improves fraud detection precision",
    --   "primary_metric": "precision_at_95_recall",
    --   "significance_level": 0.05,
    --   "minimum_sample_size": 100000,
    --   "control_variant": "control",
    --   "treatment_variants": ["treatment-a"],
    --   "guardrail_metrics": ["latency_p99", "error_rate"],
    --   "guardrail_thresholds": {"latency_p99": 50, "error_rate": 0.01}
    -- }
    results JSONB,
    -- Example results:
    -- {
    --   "control_metric": 0.891,
    --   "treatment_metric": 0.912,
    --   "lift": 0.024,
    --   "p_value": 0.003,
    --   "significant": true,
    --   "sample_size": 250000,
    --   "conclusion": "Treatment shows statistically significant improvement"
    -- }
    started_at TIMESTAMPTZ,
    concluded_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Monitoring

```sql
CREATE TABLE drift_monitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES serving_endpoints(id) ON DELETE CASCADE,
    monitor_type VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    -- All monitor configuration in JSONB
    config JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "reference_dataset_id": "uuid",
    --   "alert_threshold": 0.2,
    --   "critical_threshold": 0.35,
    --   "check_interval_minutes": 60,
    --   "features_to_monitor": ["amount", "merchant_category", "hour_of_day"],
    --   "statistical_test": "psi",
    --   "auto_rollback": true,
    --   "rollback_conditions": {
    --     "compound": true,
    --     "drift_threshold": 0.35,
    --     "business_metric": "revenue_per_transaction",
    --     "business_metric_drop_pct": 5.0,
    --     "evaluation_window_minutes": 120
    --   }
    -- }
    last_check_at TIMESTAMPTZ,
    last_result JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE drift_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    monitor_id UUID NOT NULL REFERENCES drift_monitors(id) ON DELETE CASCADE,
    severity VARCHAR(50) NOT NULL,
    drift_score NUMERIC(8,4),
    -- Full alert details in JSONB
    details JSONB NOT NULL,
    -- Example:
    -- {
    --   "feature_scores": {"amount": 0.42, "merchant_category": 0.31, "hour_of_day": 0.05},
    --   "reference_distribution": {"amount": {"mean": 45.6, "std": 22.1}},
    --   "current_distribution": {"amount": {"mean": 62.3, "std": 31.4}},
    --   "recommended_action": "Investigate amount feature drift; consider retraining"
    -- }
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_drift_alerts_monitor ON drift_alerts(monitor_id, created_at DESC);

CREATE TABLE serving_metrics (
    endpoint_id UUID NOT NULL,
    variant_name VARCHAR(255) NOT NULL,
    bucket TIMESTAMPTZ NOT NULL,
    metrics JSONB NOT NULL,
    -- Example:
    -- {
    --   "request_count": 15420,
    --   "error_count": 12,
    --   "latency_p50_ms": 8.2,
    --   "latency_p95_ms": 18.7,
    --   "latency_p99_ms": 42.1,
    --   "prediction_mean": 0.23,
    --   "prediction_stddev": 0.18,
    --   "gpu_utilization": 0.67,
    --   "batch_fill_rate": 0.82
    -- }
    PRIMARY KEY (endpoint_id, variant_name, bucket)
);
```

---

## Governance and Compliance

```sql
-- Compliance as structured JSONB -- flexible per regulatory framework
CREATE TABLE compliance_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version_id UUID REFERENCES model_versions(id),
    framework VARCHAR(100) NOT NULL,  -- 'eu_ai_act', 'nist_ai_rmf', 'iso_42001'
    -- Full compliance documentation as structured JSONB
    documentation JSONB NOT NULL DEFAULT '{}',
    -- Example for EU AI Act:
    -- {
    --   "annex_iv": {
    --     "section_1_general_description": {
    --       "status": "approved",
    --       "document_uri": "s3://compliance/fraud-detector/annex-iv-s1.pdf",
    --       "reviewed_by": "user-uuid",
    --       "reviewed_at": "2025-11-15T10:30:00Z",
    --       "content_hash": "sha256:..."
    --     },
    --     "section_2_detailed_description": {
    --       "status": "in_review",
    --       "document_uri": "s3://compliance/fraud-detector/annex-iv-s2.pdf"
    --     },
    --     "section_3_training_validation_data": {"status": "draft"},
    --     ...
    --   },
    --   "risk_classification": "high",
    --   "conformity_assessment": {"type": "third_party", "notified_body": "TUV SUD", "status": "pending"},
    --   "eu_database_registration": {"registered": false, "registration_deadline": "2026-08-02"}
    -- }
    overall_status VARCHAR(50) DEFAULT 'incomplete',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_model ON compliance_records(model_id);
CREATE INDEX idx_compliance_framework ON compliance_records(framework);
CREATE INDEX idx_compliance_documentation ON compliance_records USING GIN (documentation);

-- Audit log for all state-changing operations
CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(100) NOT NULL,  -- 'created', 'updated', 'stage_changed', 'deployed', 'rolled_back'
    actor_id UUID REFERENCES users(id),
    -- Changes captured as JSONB diff
    changes JSONB,
    -- Example: {"stage": {"old": "staging", "new": "production"}, "status": {"old": "pending", "new": "approved"}}
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id);
CREATE INDEX idx_audit_action ON audit_log(action, created_at DESC);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);
```

---

## LLM Lifecycle

```sql
-- LLM-specific metadata stored alongside model versions via JSONB
-- No separate tables needed -- the framework_metadata JSONB on model_versions handles it:
--
-- For a base LLM version:
-- framework_metadata = {
--   "model_type": "causal_lm",
--   "architecture": "LlamaForCausalLM",
--   "vocab_size": 128256,
--   "context_length": 131072,
--   "num_parameters": 70000000000,
--   "quantisation": {"method": "awq", "bits": 4, "group_size": 128},
--   "serving_engine": "vllm",
--   "tensor_parallel_size": 4
-- }
--
-- For an adapter:
-- framework_metadata = {
--   "adapter_type": "lora",
--   "base_model": "meta-llama/Llama-3-70B",
--   "rank": 16,
--   "alpha": 32,
--   "target_modules": ["q_proj", "v_proj", "k_proj", "o_proj"],
--   "trainable_parameters": 8388608,
--   "merged": false
-- }

-- Prompt versions get their own table (high query frequency, distinct lifecycle)
CREATE TABLE prompt_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    name VARCHAR(255),
    -- Prompt content
    template TEXT NOT NULL,
    system_prompt TEXT,
    -- Flexible prompt configuration
    config JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "variables": ["transaction", "customer_history"],
    --   "temperature": 0.1,
    --   "max_tokens": 256,
    --   "top_p": 0.95,
    --   "stop_sequences": ["\n\n"],
    --   "few_shot_examples": [
    --     {"input": "...", "output": "..."},
    --     {"input": "...", "output": "..."}
    --   ]
    -- }
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(model_id, version)
);

-- Webhooks
CREATE TABLE webhooks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    url TEXT NOT NULL,
    secret_hash VARCHAR(128),
    event_types TEXT[] NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Example Queries

### Find models with specific governance properties

```sql
-- "Find all high-risk models in the EU jurisdiction that need AI Act compliance"
SELECT m.name, m.risk_classification,
       m.governance->>'intended_purpose' AS purpose,
       m.governance->'jurisdictions' AS jurisdictions
FROM registered_models m
WHERE m.risk_classification = 'high'
  AND m.governance @> '{"jurisdictions": ["DE"]}'::jsonb;
```

### Compare framework-specific metadata across versions

```sql
-- "Show all PyTorch versions with their checkpoint formats"
SELECT mv.model_id, mv.version,
       mv.framework_metadata->>'framework_version' AS pytorch_version,
       mv.framework_metadata->>'checkpoint_format' AS checkpoint_format,
       mv.framework_metadata->>'torch_dtype' AS dtype,
       mv.metrics->>'accuracy' AS accuracy
FROM model_versions mv
WHERE mv.framework = 'pytorch'
  AND mv.stage = 'production';
```

### Query compliance status using JSONB containment

```sql
-- "Which models have incomplete EU AI Act Section 3 documentation?"
SELECT m.name, cr.documentation->'annex_iv'->'section_3_training_validation_data'->>'status' AS section_3_status
FROM compliance_records cr
JOIN registered_models m ON cr.model_id = m.id
WHERE cr.framework = 'eu_ai_act'
  AND NOT (cr.documentation @> '{"annex_iv": {"section_3_training_validation_data": {"status": "approved"}}}');
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Access Control | 2 | organizations, users |
| Registry Core | 3 | registered_models, model_versions, model_aliases |
| Datasets & Lineage | 3 | datasets, dataset_versions, lineage_links |
| Serving & Traffic | 3 | serving_endpoints, serving_variants, ab_experiments |
| Monitoring | 3 | drift_monitors, drift_alerts, serving_metrics |
| Governance | 2 | compliance_records, audit_log |
| LLM & Events | 2 | prompt_versions, webhooks |
| **Total** | **18** | Significantly fewer than fully normalised (30 tables) |

---

## Key Design Decisions

1. **JSONB for variable metadata, typed columns for filtered fields** -- fields that appear in WHERE clauses (stage, framework, risk_classification, status) are typed columns with B-tree indexes. Fields that vary by framework, jurisdiction, or experiment type go into JSONB with GIN indexes. This keeps the schema stable while allowing infinite extensibility.

2. **No separate tags tables** -- unlike MLflow which uses model_version_tags and registered_model_tags junction tables, tags are JSONB columns on the parent entity. This eliminates two tables and avoids N+1 query patterns when fetching models with their tags.

3. **Metrics and parameters as JSONB on model_versions** -- rather than separate metrics and parameters tables (saving 2 more tables), evaluation results are denormalised onto the version record. This works well when the number of metrics per version is small (typically 5-20 metrics), which is the common case.

4. **Compliance as a single structured JSONB document** -- rather than separate tables for each Annex IV section, each compliance checklist item, and each framework, a single compliance_records table with a deeply structured documentation JSONB column handles all regulatory frameworks. JSON Schema validation at the application layer ensures structure.

5. **LLM adapter and quantisation metadata in framework_metadata** -- rather than dedicated adapter_weights and quantisation_variants tables, LLM-specific metadata lives in the model_versions.framework_metadata JSONB column. This works because adapters and quantisation variants ARE model versions; they just have different metadata shapes.

6. **Audit log with JSONB change diffs** -- the audit_log table captures changes as JSONB diffs (old value / new value), providing a pragmatic audit trail without the full complexity of event sourcing. This is "80% of the compliance value at 20% of the implementation cost."

7. **Serving configuration as JSONB** -- endpoint and variant configurations vary enormously by runtime (KServe vs. Seldon vs. Triton vs. BentoML). Rather than trying to normalise all possible configuration fields, a config JSONB column holds runtime-specific settings validated by the application.

8. **18 tables vs. 30** -- this design achieves the same functional coverage as the fully normalised model with 40% fewer tables, at the cost of weaker database-level type checking and constraint enforcement on JSONB fields.
