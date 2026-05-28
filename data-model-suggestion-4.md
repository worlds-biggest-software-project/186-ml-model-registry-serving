# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: ML Model Registry & Serving · Created: 2026-05-20

## Philosophy

This approach combines a relational backbone for operational CRUD (registry, serving, compliance) with a property graph layer for relationship-heavy queries that are central to ML lifecycle management. The graph layer models lineage, dependency chains, impact analysis, and conflict-of-interest relationships that are awkward to express in pure relational schemas but natural in graph form.

ML model registries are inherently graph problems: models are trained on datasets, which are derived from feature pipelines, which consume upstream data sources. Models are deployed to endpoints, which serve variants, which participate in A/B experiments. A drift alert on an endpoint traces back through the model version, to the training dataset, to the feature store update that caused the distribution shift. These multi-hop traversals -- "what upstream changes could have caused this model degradation?" -- are the sweet spot for graph queries.

Rather than requiring a separate graph database (Neo4j, Amazon Neptune), this design uses PostgreSQL with a `graph_nodes` / `graph_edges` table pair and recursive CTEs for traversal queries. This avoids the operational burden of a second database while enabling graph-style queries within the same transaction boundary as CRUD operations. For teams that outgrow this pattern, the graph tables can be synced to a dedicated graph database.

**Best for:** Teams that need rich lineage traversal, impact analysis ("what breaks if we retrain this model?"), root cause analysis for drift, and dependency graph visualisation across the ML lifecycle.

**Trade-offs:**
- (+) Multi-hop relationship queries (lineage, impact analysis, root cause) are natural and efficient
- (+) Graph layer enables visualisation of model-dataset-endpoint dependency networks
- (+) Flexible edge types allow new relationship kinds without schema changes
- (+) Single database (PostgreSQL) -- no operational overhead of a separate graph DB
- (+) Relational tables provide ACID guarantees for operational data
- (-) Recursive CTEs are less performant than native graph databases for deep traversals (>10 hops)
- (-) Two conceptual models (relational + graph) increase cognitive load for developers
- (-) Graph data must be kept consistent with relational tables (dual-write risk)
- (-) No native graph query language (Cypher/Gremlin); SQL recursive CTEs are verbose
- (-) Graph indexes (GiST/ltree) are less mature than B-tree for complex traversals

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| MLflow REST API 2.0 | Relational tables align with MLflow entities; graph layer adds lineage not present in MLflow |
| ONNX | ONNX models are graph nodes with edges to their training datasets and serving endpoints |
| EU AI Act Annex IV | Section 3 (training/validation data) lineage is expressed as graph edges from model to dataset nodes |
| NIST AI RMF | "Map" function (identify AI system dependencies) directly maps to graph traversal queries |
| Open Inference Protocol | Serving endpoints are graph nodes connected to model version nodes |
| W3C PROV-O Ontology | Graph edge types align with PROV-O provenance relationships (wasDerivedFrom, wasGeneratedBy, used) |
| ISO/IEC 42001 | AI management system scope boundaries modeled as subgraphs |

---

## Graph Layer

```sql
-- Generic property graph overlay
-- Every significant entity in the system has a corresponding graph node
CREATE TABLE graph_nodes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Node type and identity
    node_type VARCHAR(100) NOT NULL,
    -- Types: 'registered_model', 'model_version', 'dataset', 'dataset_version',
    --        'serving_endpoint', 'serving_variant', 'experiment_run',
    --        'code_commit', 'feature_pipeline', 'prompt_version',
    --        'adapter', 'compliance_document', 'drift_alert', 'user'
    external_id UUID,           -- FK to the corresponding relational table row
    external_table VARCHAR(100), -- which relational table this node represents
    -- Display metadata
    label VARCHAR(500) NOT NULL,  -- human-readable label for graph visualisation
    -- Searchable properties
    properties JSONB DEFAULT '{}',
    -- Example for a model_version node:
    -- {
    --   "model_name": "fraud-detector",
    --   "version": 3,
    --   "framework": "pytorch",
    --   "stage": "production",
    --   "artifact_uri": "s3://models/fraud-detector/v3",
    --   "created_at": "2025-11-01T10:00:00Z"
    -- }
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_external ON graph_nodes(external_table, external_id);
CREATE INDEX idx_graph_nodes_label ON graph_nodes USING gin(to_tsvector('english', label));
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN (properties);

CREATE TABLE graph_edges (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Edge endpoints
    source_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    -- Edge type (W3C PROV-O aligned where applicable)
    edge_type VARCHAR(100) NOT NULL,
    -- Types:
    --   Lineage (PROV-O): 'was_derived_from', 'was_generated_by', 'used',
    --                      'was_attributed_to', 'was_informed_by'
    --   Deployment:        'deployed_to', 'serves_on', 'participates_in'
    --   Governance:        'approved_by', 'reviewed_by', 'owns'
    --   Dependency:        'depends_on', 'is_adapter_of', 'is_quantisation_of'
    --   Monitoring:        'monitors', 'alerted_on', 'triggered_rollback_of'
    -- Edge properties
    properties JSONB DEFAULT '{}',
    -- Example for 'was_derived_from':
    -- {
    --   "relationship": "trained_on",
    --   "dataset_split": "train",
    --   "num_samples": 1200000,
    --   "sampling_strategy": "stratified"
    -- }
    -- Temporal validity (when was this edge true?)
    valid_from TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_until TIMESTAMPTZ,  -- NULL means currently valid
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_graph_edges_valid ON graph_edges(valid_from, valid_until);
-- Composite index for traversal queries
CREATE INDEX idx_graph_edges_traverse ON graph_edges(source_id, edge_type, target_id);
CREATE INDEX idx_graph_edges_reverse ON graph_edges(target_id, edge_type, source_id);
```

---

## Relational Core (Operational Tables)

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
    role VARCHAR(50) NOT NULL DEFAULT 'member',
    is_service_principal BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);

CREATE INDEX idx_users_org ON users(organization_id);

CREATE TABLE registered_models (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID REFERENCES users(id),
    risk_classification VARCHAR(50),
    intended_purpose TEXT,
    tags JSONB DEFAULT '{}',
    governance JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, name)
);

CREATE INDEX idx_models_org ON registered_models(organization_id);

CREATE TABLE model_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    description TEXT,
    artifact_uri TEXT NOT NULL,
    artifact_format VARCHAR(50) NOT NULL DEFAULT 'mlmodel',
    artifact_hash VARCHAR(128),
    framework VARCHAR(100) NOT NULL,
    stage VARCHAR(50) NOT NULL DEFAULT 'development',
    status VARCHAR(50) NOT NULL DEFAULT 'pending_review',
    run_id VARCHAR(255),
    source_uri TEXT,
    framework_metadata JSONB DEFAULT '{}',
    input_schema JSONB,
    output_schema JSONB,
    tags JSONB DEFAULT '{}',
    metrics JSONB DEFAULT '{}',
    parameters JSONB DEFAULT '{}',
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(model_id, version)
);

CREATE INDEX idx_versions_model ON model_versions(model_id);
CREATE INDEX idx_versions_stage ON model_versions(stage);

CREATE TABLE model_aliases (
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    alias VARCHAR(255) NOT NULL,
    version_id UUID NOT NULL REFERENCES model_versions(id),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (model_id, alias)
);

CREATE TABLE datasets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    source_type VARCHAR(50),
    source_uri TEXT,
    metadata JSONB DEFAULT '{}',
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
```

### Serving Tables

```sql
CREATE TABLE serving_endpoints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name VARCHAR(255) NOT NULL,
    protocol VARCHAR(50) NOT NULL DEFAULT 'v2',
    status VARCHAR(50) NOT NULL DEFAULT 'creating',
    endpoint_url TEXT,
    config JSONB NOT NULL DEFAULT '{}',
    traffic_split JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, name)
);

CREATE TABLE serving_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES serving_endpoints(id) ON DELETE CASCADE,
    version_id UUID NOT NULL REFERENCES model_versions(id),
    variant_name VARCHAR(255) NOT NULL,
    traffic_percentage NUMERIC(5,2) NOT NULL DEFAULT 0,
    is_shadow BOOLEAN NOT NULL DEFAULT FALSE,
    status VARCHAR(50) NOT NULL DEFAULT 'creating',
    config JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(endpoint_id, variant_name)
);

CREATE TABLE drift_monitors (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    endpoint_id UUID NOT NULL REFERENCES serving_endpoints(id) ON DELETE CASCADE,
    monitor_type VARCHAR(50) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    config JSONB NOT NULL DEFAULT '{}',
    last_check_at TIMESTAMPTZ,
    last_result JSONB,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE drift_alerts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    monitor_id UUID NOT NULL REFERENCES drift_monitors(id) ON DELETE CASCADE,
    severity VARCHAR(50) NOT NULL,
    drift_score NUMERIC(8,4),
    details JSONB NOT NULL,
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
    PRIMARY KEY (endpoint_id, variant_name, bucket)
);
```

### Governance and Compliance

```sql
CREATE TABLE compliance_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version_id UUID REFERENCES model_versions(id),
    framework VARCHAR(100) NOT NULL,
    documentation JSONB NOT NULL DEFAULT '{}',
    overall_status VARCHAR(50) DEFAULT 'incomplete',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_model ON compliance_records(model_id);

CREATE TABLE audit_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type VARCHAR(100) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(100) NOT NULL,
    actor_id UUID REFERENCES users(id),
    changes JSONB,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at DESC);

CREATE TABLE prompt_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_id UUID NOT NULL REFERENCES registered_models(id) ON DELETE CASCADE,
    version INTEGER NOT NULL,
    name VARCHAR(255),
    template TEXT NOT NULL,
    system_prompt TEXT,
    config JSONB DEFAULT '{}',
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(model_id, version)
);

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

## Graph Synchronisation

```sql
-- Trigger function to auto-create graph nodes when relational entities are created
-- (implemented as application-level middleware or database triggers)

-- Example: When a model_version is inserted, create corresponding graph node and edges
CREATE OR REPLACE FUNCTION sync_model_version_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    node_id UUID;
    model_node_id UUID;
BEGIN
    -- Create graph node for the new version
    INSERT INTO graph_nodes (node_type, external_id, external_table, label, properties)
    VALUES (
        'model_version',
        NEW.id,
        'model_versions',
        (SELECT name FROM registered_models WHERE id = NEW.model_id) || ' v' || NEW.version,
        jsonb_build_object(
            'model_id', NEW.model_id,
            'version', NEW.version,
            'framework', NEW.framework,
            'stage', NEW.stage,
            'artifact_uri', NEW.artifact_uri
        )
    )
    RETURNING id INTO node_id;

    -- Find the model's graph node
    SELECT id INTO model_node_id
    FROM graph_nodes
    WHERE node_type = 'registered_model' AND external_id = NEW.model_id;

    -- Create edge: model_version --[was_generated_by]--> registered_model
    IF model_node_id IS NOT NULL THEN
        INSERT INTO graph_edges (source_id, target_id, edge_type)
        VALUES (node_id, model_node_id, 'was_generated_by');
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_model_version_graph
    AFTER INSERT ON model_versions
    FOR EACH ROW
    EXECUTE FUNCTION sync_model_version_to_graph();
```

---

## Example Graph Queries

### Full lineage traversal (upstream dependencies)

```sql
-- "What datasets and feature pipelines were used to train the model currently in production?"
WITH RECURSIVE upstream AS (
    -- Start from the production model version's graph node
    SELECT gn.id, gn.node_type, gn.label, gn.properties, 0 AS depth
    FROM graph_nodes gn
    WHERE gn.external_id = (
        SELECT ma.version_id FROM model_aliases ma
        JOIN registered_models rm ON ma.model_id = rm.id
        WHERE rm.name = 'fraud-detector' AND ma.alias = 'champion'
    )
    AND gn.node_type = 'model_version'

    UNION ALL

    -- Traverse upstream edges
    SELECT parent.id, parent.node_type, parent.label, parent.properties, u.depth + 1
    FROM upstream u
    JOIN graph_edges e ON e.source_id = u.id
        AND e.edge_type IN ('was_derived_from', 'used', 'was_generated_by')
        AND (e.valid_until IS NULL OR e.valid_until > now())
    JOIN graph_nodes parent ON parent.id = e.target_id
    WHERE u.depth < 10  -- prevent infinite loops
)
SELECT node_type, label, properties, depth
FROM upstream
ORDER BY depth ASC;
```

### Impact analysis (downstream dependents)

```sql
-- "What models and endpoints would be affected if we update dataset 'transaction-features-v5'?"
WITH RECURSIVE downstream AS (
    SELECT gn.id, gn.node_type, gn.label, gn.properties, 0 AS depth
    FROM graph_nodes gn
    WHERE gn.node_type = 'dataset_version'
      AND gn.properties @> '{"dataset_name": "transaction-features", "version": 5}'

    UNION ALL

    SELECT child.id, child.node_type, child.label, child.properties, d.depth + 1
    FROM downstream d
    JOIN graph_edges e ON e.target_id = d.id
        AND e.edge_type IN ('was_derived_from', 'used', 'deployed_to', 'serves_on')
        AND (e.valid_until IS NULL OR e.valid_until > now())
    JOIN graph_nodes child ON child.id = e.source_id
    WHERE d.depth < 10
)
SELECT node_type, label, properties, depth
FROM downstream
WHERE node_type IN ('model_version', 'serving_endpoint')
ORDER BY depth ASC;
```

### Root cause analysis for drift

```sql
-- "What changed in the 24 hours before this drift alert?"
WITH alert_time AS (
    SELECT da.created_at AS alert_at, gn.id AS alert_node_id
    FROM drift_alerts da
    JOIN drift_monitors dm ON da.monitor_id = dm.id
    JOIN graph_nodes gn ON gn.external_id = da.id AND gn.node_type = 'drift_alert'
    WHERE da.id = 'alert-uuid'
),
-- Find the endpoint and model version connected to this alert
alert_context AS (
    SELECT gn.id, gn.node_type, gn.external_id
    FROM alert_time at
    JOIN graph_edges e ON e.source_id = at.alert_node_id
    JOIN graph_nodes gn ON gn.id = e.target_id
    WHERE gn.node_type IN ('serving_endpoint', 'model_version')
),
-- Find upstream changes in the 24-hour window
upstream_changes AS (
    SELECT gn.node_type, gn.label, gn.properties, e.edge_type, e.created_at
    FROM alert_context ac
    JOIN graph_edges e ON (e.source_id = ac.id OR e.target_id = ac.id)
    JOIN graph_nodes gn ON gn.id = CASE WHEN e.source_id = ac.id THEN e.target_id ELSE e.source_id END
    JOIN alert_time at ON TRUE
    WHERE e.created_at BETWEEN at.alert_at - INTERVAL '24 hours' AND at.alert_at
)
SELECT * FROM upstream_changes ORDER BY created_at ASC;
```

### Ownership and approval chains

```sql
-- "Who approved the model version that's currently serving on the fraud-scorer endpoint?"
SELECT gn.label, e.edge_type, approver.label AS approver,
       e.properties->>'comment' AS approval_comment,
       e.created_at AS approved_at
FROM graph_nodes endpoint_node
JOIN graph_edges e1 ON e1.source_id = endpoint_node.id AND e1.edge_type = 'serves_on'
JOIN graph_nodes version_node ON version_node.id = e1.target_id
JOIN graph_edges e ON e.source_id = version_node.id AND e.edge_type = 'approved_by'
JOIN graph_nodes approver ON approver.id = e.target_id
JOIN graph_nodes gn ON gn.id = version_node.id
WHERE endpoint_node.node_type = 'serving_endpoint'
  AND endpoint_node.properties->>'name' = 'fraud-scorer-prod';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_nodes, graph_edges |
| Access Control | 2 | organizations, users |
| Registry Core | 3 | registered_models, model_versions, model_aliases |
| Datasets | 2 | datasets, dataset_versions |
| Serving & Traffic | 2 | serving_endpoints, serving_variants |
| Monitoring | 3 | drift_monitors, drift_alerts, serving_metrics |
| Governance | 2 | compliance_records, audit_log |
| LLM & Events | 2 | prompt_versions, webhooks |
| **Total** | **20** | 18 relational + 2 graph overlay |

---

## Key Design Decisions

1. **Graph as an overlay, not a replacement** -- the graph layer (graph_nodes + graph_edges) overlays the relational tables rather than replacing them. Operational CRUD uses relational tables directly; graph queries are used for lineage, impact analysis, and visualisation. This avoids the performance penalty of routing all queries through graph traversal.

2. **W3C PROV-O aligned edge types** -- edge types like `was_derived_from`, `was_generated_by`, and `used` align with the W3C PROV-O (Provenance Ontology) standard, making lineage data interoperable with other provenance-aware systems and tools.

3. **Temporal edges with valid_from/valid_until** -- edges have temporal validity, enabling point-in-time lineage queries ("what was the dependency graph on date X?"). When a model is redeployed to a different endpoint, the old edge gets a `valid_until` timestamp and a new edge is created.

4. **Database triggers for graph synchronisation** -- triggers on relational tables automatically create/update graph nodes and edges, preventing the dual-write consistency problem. The graph is a derived data structure maintained by the database itself.

5. **Recursive CTEs for traversal** -- PostgreSQL recursive CTEs enable multi-hop traversals (upstream lineage, downstream impact) within standard SQL. A depth limit prevents infinite loops. For teams needing faster traversals, the graph tables can be synced to Neo4j or Apache AGE (PostgreSQL graph extension).

6. **Properties as JSONB on both nodes and edges** -- node and edge properties use JSONB with GIN indexes, enabling flexible metadata and containment queries without schema changes. This is analogous to property graph databases like Neo4j.

7. **Graph-powered root cause analysis** -- the graph layer enables the AI-native root cause analysis feature described in the project README. When drift is detected, the system traverses upstream edges to find recent changes (new model deployments, dataset updates, feature pipeline modifications) that may have caused the degradation.

8. **External ID mapping** -- every graph node stores an `external_id` and `external_table` reference back to the relational source row. This enables bi-directional navigation: from a relational query result to its graph context, and from a graph traversal back to relational detail.
