# ML Model Registry & Serving — Feature & Functionality Survey

> Candidate #186 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| MLflow | Open-source ML lifecycle platform | Apache 2.0 / Databricks managed | https://mlflow.org |
| Weights & Biases (W&B) Registry | Commercial MLOps platform | Proprietary (acquired by CoreWeave 2025) | https://wandb.ai |
| Amazon SageMaker Model Registry | Managed cloud MLOps | Proprietary (AWS) | https://aws.amazon.com/sagemaker |
| Neptune.ai | Commercial metadata store | Proprietary (acquired by OpenAI 2025–2026) | https://neptune.ai |
| ClearML | Open-source MLOps platform | Apache 2.0 / Managed hosting | https://clear.ml |
| Databricks Mosaic AI | Managed MLflow + Unity Catalog | Proprietary (Databricks) | https://databricks.com |
| Vertex AI Model Registry | Managed cloud ML platform | Proprietary (GCP) | https://cloud.google.com/vertex-ai |
| Azure ML Model Registry | Managed cloud ML platform | Proprietary (Microsoft Azure) | https://azure.microsoft.com/en-us/products/machine-learning |
| Kubeflow Model Registry | Open-source Kubernetes-native registry | Apache 2.0 | https://github.com/kubeflow/model-registry |
| KServe | Open-source Kubernetes model serving | Apache 2.0 | https://kserve.github.io |
| Seldon Core | Open-source Kubernetes model serving | Apache 2.0 / Commercial (Seldon) | https://seldon.io |
| BentoML | Open-source Python model serving framework | Apache 2.0 | https://bentoml.com |
| TrueFoundry | Commercial LLMOps / MLOps platform | Proprietary | https://truefoundry.com |
| Verta.ai | Commercial enterprise MLOps / governance | Proprietary | https://verta.ai |

---

## Feature Analysis by Solution

### MLflow

**Core features**
- Centralised model registry with versioning (automatic version increment per registration)
- Model aliasing: mutable named references (e.g. `champion`, `challenger`) to specific versions
- Model lifecycle stages: `Staging`, `Production`, `Archived` with approval workflow
- Full lineage tracing: each version links to the originating run, logged model, or notebook
- Tag and annotation support (key-value metadata on models and versions)
- REST API and Python SDK for all registry operations
- Model serving via `mlflow models serve` — automatic REST endpoint generation
- Support for multiple model flavours: Python function, ONNX, TensorFlow, PyTorch, scikit-learn, and more
- Databricks Unity Catalog integration for enterprise governance
- MLflow 3 release (2026): major upgrade with enhanced LLM and agent tracking capabilities

**Differentiating features**
- De facto open standard: the `MLmodel` format is supported across most competing platforms
- Managed as part of the broader MLflow AI Platform (experiment tracking, evaluation, deployment)
- Unity Catalog integration provides enterprise-grade access control and data lineage across the Databricks ecosystem

**UX patterns**
- Web UI with searchable model list, version comparison, and stage transition interface
- Approval workflows for promoting models between stages with comment history
- Experiment-to-registry linking displayed in both the experiments UI and registry UI

**Integration points**
- REST API documented at `/api/2.0/mlflow/...`
- Python SDK (`mlflow.register_model`, `mlflow.deployments`)
- Databricks SDK integration
- SageMaker, Azure ML, and other cloud platform deployment plugins
- Delta Lake and Unity Catalog data governance integrations

**Known gaps**
- Native EU AI Act compliance fields absent (open GitHub issue #21022 requests this explicitly)
- Serving infrastructure is minimal compared to dedicated serving platforms (no built-in autoscaling or traffic splitting without Databricks)
- No built-in drift detection or monitoring; requires external tools
- Multi-cloud or multi-cluster serving requires custom integration
- UI becomes unwieldy at very large scale (thousands of models)

**Licence / IP notes**
- Apache 2.0. Owned by Databricks; core OSS is community-maintained but Databricks controls the roadmap.

---

### Weights & Biases (W&B) Registry

**Core features**
- Registry for trained model artifacts with versioning and lineage to originating experiments
- Model promotion workflow: publish candidates for production from experiment tracking runs
- Webhook and automation triggers on version events (new version registered, stage transition)
- Lineage graph: models linked to datasets, experiments, hyperparameter sweeps
- Role-based access control and team workspaces
- LLM-specific tooling: prompt tracking, fine-tune run tracking
- Automated hyperparameter sweeps with Bayesian/grid/random strategies
- Interactive visualisation dashboards for run comparison

**Differentiating features**
- Strongest developer experience and UI in the category; favourite for research and fast-iteration teams
- Tight integration between experiment tracking and model registry (single lineage graph)
- CoreWeave acquisition (2025) enables deep integration with GPU compute infrastructure
- W&B Registry replaces legacy Model Registry, broadening artifact scope beyond models

**UX patterns**
- Polished, consumer-grade UI with interactive charts and dashboards
- Low-friction onboarding: `wandb.init()` + `wandb.log()` in three lines
- Model card creation embedded in the registry workflow
- Progressive disclosure: basic logging works immediately, advanced features (sweeps, artifacts, alerts) are discoverable

**Integration points**
- Python SDK (`wandb` library), REST API
- Native integrations: PyTorch, TensorFlow, Keras, HuggingFace, JAX
- Webhook outputs for CI/CD pipeline integration
- CoreWeave compute cluster integration post-acquisition

**Known gaps**
- Deployment and serving workflows weaker than MLflow or SageMaker; the registry is a staging hub, not a serving engine
- Enterprise governance (RBAC, audit logs at org level) requires Enterprise plan
- Acquisition by CoreWeave creates vendor lock-in risk; non-CoreWeave serving requires custom pipelines
- Service continuity risk after SaaS shutdown of Neptune accelerated concern about single-vendor dependency

**Licence / IP notes**
- Proprietary. The `wandb` Python client library is MIT-licensed on PyPI, but the platform itself is closed-source.

---

### Amazon SageMaker Model Registry

**Core features**
- Model Package Groups for cataloguing multiple versions of a model solving a given problem
- Model approval status workflow: `Pending`, `Approved`, `Rejected`
- Full model lineage and audit trail integrated with SageMaker Experiments and Pipelines
- One-click deployment from registry to real-time endpoint, batch transform, or serverless inference
- A/B testing via multi-variant endpoints: traffic distribution configurable per variant
- Integration with Amazon CloudWatch for per-variant latency and invocation metrics
- Cross-account model sharing via AWS Resource Access Manager
- Pipeline integration: CI/CD-style approval gates before deployment

**Differentiating features**
- Deepest AWS ecosystem integration: IAM, CloudWatch, S3, Step Functions, EventBridge
- Multi-variant endpoints with fine-grained traffic splitting for production A/B experiments
- Serverless inference endpoint type (pay-per-invocation, no idle cost)
- Shadow testing: route a shadow copy of production traffic to a new variant for testing

**UX patterns**
- AWS Management Console UI with registry-to-deployment pipeline wizard
- EventBridge triggers for automated promotion on metric thresholds
- Cross-team model sharing via approval workflows documented in audit trail

**Integration points**
- AWS SDK (boto3) and SageMaker Python SDK
- AWS CLI for all registry operations
- EventBridge, Lambda, Step Functions for workflow automation
- CloudWatch Metrics and Logs for observability
- SageMaker Pipelines for end-to-end CI/CD

**Known gaps**
- Steep learning curve: complex pricing and deep AWS service graph
- Vendor lock-in: models registered in SageMaker are difficult to move to other platforms
- UI is functional but less polished than W&B
- Serving inference at very high throughput often requires careful instance-type selection and reserved capacity planning
- No native LLM-specific registry features (prompt versions, adapter weights)

**Licence / IP notes**
- Proprietary AWS service. No open-source component.

---

### Neptune.ai

**Core features**
- Metadata store for ML experiments, runs, and model versions
- Model registry with version tagging and stage management
- Dataset and artifact versioning with lineage tracking (code commits, data versions, configs)
- Custom metadata logging: metrics, images, audio, video, HTML, dataframes
- Team collaboration: shared workspaces, run comparison, custom dashboards

**Differentiating features**
- Infrastructure-agnostic: works with any training framework without imposing a pipeline structure
- Very granular metadata capture including partial-run logging for long-running jobs
- Strong notebook integration

**UX patterns**
- Advanced interactive WebUI for experiment comparison with customisable columns and charts
- Queryable run database via SDK; supports complex filtering and aggregation

**Integration points**
- Python SDK, R client
- Integrations with TensorFlow, PyTorch, Keras, XGBoost, LightGBM, Optuna, and more
- REST API

**Known gaps**
- SaaS service shutdown announced for March 2026; OpenAI acquisition dramatically alters the roadmap
- Weaker deployment/serving capabilities compared to MLflow or SageMaker
- Separate registry service required; not unified with experiment tracking at data model level
- No built-in serving infrastructure

**Licence / IP notes**
- Proprietary SaaS. Client SDK is open-source (Apache 2.0); platform is closed.

---

### ClearML

**Core features**
- Full MLOps platform: experiment tracking, data versioning, model registry, pipeline orchestration, and model serving
- Model registry with versioning, tagging, and lifecycle management
- Automated hyperparameter optimisation (HPO) with multiple search strategies
- Data management with versioning and dataset lineage
- Pipeline/DAG management for reproducible workflows
- Kubernetes and on-premise deployment support
- Auto-scaling agents for compute management

**Differentiating features**
- Broadest scope of any open-source MLOps platform
- Self-hosted deployment option with no external SaaS dependency
- Automation-first design: ClearML Agents for distributed experiment execution

**UX patterns**
- Comprehensive but complex UI; less polished than W&B for new users
- Configuration via YAML/SDK; lower barrier for engineers comfortable with code
- Progressive capability: start with experiment tracking, add orchestration and serving as needed

**Integration points**
- Python SDK, REST API
- Kubernetes, Docker, and on-premise compute
- Integration with most major ML frameworks
- Webhooks for external notifications

**Known gaps**
- UI complexity can overwhelm teams not needing the full MLOps stack
- Documentation quality uneven across features
- Smaller community than MLflow
- Serving capabilities exist but are less mature than dedicated serving platforms

**Licence / IP notes**
- Apache 2.0 (open-source). Commercial managed hosting available separately.

---

### Databricks Mosaic AI (Managed MLflow + Unity Catalog)

**Core features**
- Managed MLflow registry with Unity Catalog for enterprise access control and governance
- Model serving endpoints: real-time REST inference with auto-scaling
- A/B testing and traffic splitting between model versions on serving endpoints
- Foundation model APIs: hosted access to Llama, Mistral, DBRX, and other models
- Automated model monitoring: data drift, prediction drift, and custom metrics
- Integration with Delta Lake for data lineage and feature store

**Differentiating features**
- Unity Catalog as a universal governance layer across data, features, and models
- Integrated with Databricks Lakehouse Platform — single platform from raw data to production inference
- Tecton acquisition (2025) adds mature feature store capabilities

**UX patterns**
- Databricks workspace as unified environment; no context-switching between tools
- One-click endpoint creation from registered model UI
- Automated monitoring dashboards built into the platform

**Integration points**
- Databricks SDK, MLflow SDK, REST API
- Delta Lake, Unity Catalog, Databricks Feature Store
- Integrated with Databricks Jobs, Workflows, and Notebooks

**Known gaps**
- Severe vendor lock-in: deeply integrated with Databricks infrastructure
- Compute-based pricing is opaque and can be expensive at scale
- Non-Databricks users cannot benefit without migrating their data platform

**Licence / IP notes**
- Proprietary. Built on open-source MLflow (Apache 2.0) but the managed platform is closed.

---

### Vertex AI Model Registry

**Core features**
- Centralised model management with version tracking and lifecycle management
- Support for custom models (all frameworks) and AutoML models, including BigQuery ML
- Integrated model evaluation: validation metrics and explainability (Shapley values, integrated gradients)
- Model monitoring: input skew and prediction drift detection post-deployment
- Model cards generated from registry metadata
- Online prediction endpoints with autoscaling and traffic splitting
- Batch prediction jobs for large-scale offline inference

**Differentiating features**
- Deep integration with Google Cloud services (BigQuery, Vertex Feature Store, Vertex Pipelines)
- Explainability-first: built-in attribution methods tied to the registry
- AutoML-to-custom model continuum within a single registry interface

**UX patterns**
- Google Cloud Console UI with model cards, evaluation charts, and monitoring dashboards
- Vertex Pipelines for CI/CD-style promotion workflows

**Integration points**
- Python SDK (`google-cloud-aiplatform`), REST API, gRPC API
- BigQuery ML, Vertex Feature Store, Vertex Pipelines, Vertex Experiments
- Cloud Build and Cloud Functions for workflow automation

**Known gaps**
- GCP lock-in; multi-cloud serving requires custom engineering
- Less developer-friendly than W&B for experiment tracking
- LLM-specific registry features (prompt versioning, adapter management) are nascent

**Licence / IP notes**
- Proprietary GCP service.

---

### Kubeflow Model Registry

**Core features**
- Kubernetes-native open-source registry exposing a REST API (OpenAPI specification)
- Index and manage models, versions, and ML artifact metadata
- Logical model — registered model → model version → model artifact hierarchy
- Python client library and Go library for integration
- Web UI for browsing registered models and versions
- Designed as a neutral hub between experimentation and production

**Differentiating features**
- Only fully open-source, Kubernetes-native registry with a published REST API spec
- Vendor-neutral: no lock-in; integrates with any serving platform
- Contract-first API design using OpenAPI (model-registry.yaml)

**UX patterns**
- Lightweight UI focused on discoverability rather than rich visualisation
- SDK-first: most interaction expected via Python client
- Acts as metadata hub rather than serving engine

**Integration points**
- REST API (v1alpha3), Python client, Go library
- Integrates with KServe, Seldon, and other serving platforms
- Can receive model registrations from MLflow, W&B, or custom training pipelines

**Known gaps**
- No built-in serving; purely a metadata registry
- UI is minimal; lacks experiment tracking, comparison, and monitoring
- Still in alpha API maturity (v1alpha3)
- No built-in access control; relies on Kubernetes RBAC

**Licence / IP notes**
- Apache 2.0.

---

### KServe

**Core features**
- Kubernetes-native serverless model serving with Open Inference Protocol (V2)
- Support for TensorFlow, PyTorch, scikit-learn, XGBoost, PMML, Spark MLlib, LightGBM, HuggingFace
- Canary rollout and traffic splitting between model versions
- Horizontal Pod Autoscaling (HPA) and scale-to-zero
- Pre/post processing transformer pipeline support
- gRPC and REST inference endpoints
- Model explainability integration (SHAP, LIME)
- InferenceGraph for ensemble and multi-model pipelines

**Differentiating features**
- Implements the CNCF Open Inference Protocol — truly standards-based serving
- Scale-to-zero (serverless) for cost efficiency
- Lightweight compared to Seldon Core; easier to set up

**UX patterns**
- Kubernetes CRD-based configuration (`InferenceService` YAML)
- Minimal UI; monitoring via Prometheus/Grafana stack

**Integration points**
- Kubernetes, Istio, Knative for service mesh and serverless
- Prometheus and Grafana for metrics
- Open Inference Protocol clients in Python, Java, Go
- Integrates with Kubeflow Model Registry

**Known gaps**
- No built-in model registry; needs external registry (Kubeflow, MLflow)
- No native batch inference support (Seldon is preferred for batch)
- PyTorch direct support requires Triton Server intermediary
- Operational complexity: requires Kubernetes, Istio, and Knative expertise

**Licence / IP notes**
- Apache 2.0. CNCF sandbox project.

---

### Seldon Core

**Core features**
- Kubernetes-native model serving with canary, A/B, and multi-armed bandit deployment patterns
- Seldon Deployment CRD for defining inference graphs (pre/post processing, ensembles, routing)
- Out-of-the-box support: scikit-learn, XGBoost, LightGBM, Spark MLlib, MLflow, HuggingFace, Triton
- Horizontal Pod Autoscaling and batch processing integration
- Outlier detection and drift monitoring via Alibi Detect
- Explainability via Alibi Explain (SHAP, integrated gradients, anchors)
- Request/response payload logging to object stores (S3, GCS)

**Differentiating features**
- Most feature-complete open-source serving platform for batch and complex inference pipelines
- Alibi ecosystem (Detect + Explain) is the most mature open-source ML monitoring suite
- Multi-armed bandit routing for dynamic traffic optimisation

**UX patterns**
- YAML CRD-based deployment configuration
- Seldon Deploy (commercial UI) available for non-Kubernetes users
- Monitoring dashboards via Prometheus/Grafana

**Integration points**
- Kubernetes, Istio
- Alibi Detect and Alibi Explain
- MLflow model format support
- REST and gRPC inference endpoints

**Known gaps**
- No built-in model registry; needs external registry
- No native PyTorch support without Triton Server
- Operational complexity similar to KServe
- Commercial features (Seldon Deploy, enterprise support) require paid licence

**Licence / IP notes**
- Apache 2.0 (core). Seldon Deploy is proprietary commercial product.

---

### BentoML

**Core features**
- Python framework for wrapping any ML model in a deployable service
- Model store with versioning per framework (sklearn, PyTorch, TensorFlow, HuggingFace, ONNX, custom)
- Bento packaging: bundles model, code, dependencies, and configuration into a deployable artefact
- Runner architecture: separates model inference from serving logic for multi-model pipelines
- Adaptive batching for throughput optimisation
- BentoCloud: managed deployment target for Bentos
- REST API auto-generation from Python service class

**Differentiating features**
- Framework-agnostic: any Python object can be wrapped as a model
- Lowest barrier to packaging: implement one Python class, get REST API and Docker image
- Best choice for small teams and rapid prototyping

**UX patterns**
- Code-first: define service as a Python class with `@svc.api` decorators
- CLI for saving, loading, and serving models (`bentoml serve`)
- BentoCloud dashboard for managed deployment monitoring

**Integration points**
- Python SDK; integrates with any ML framework
- Docker and Kubernetes deployment targets
- BentoCloud managed hosting
- OpenTelemetry support for observability

**Known gaps**
- No built-in model registry governance features (approval workflows, role-based access)
- No native A/B traffic splitting; requires external load balancer
- Limited monitoring and drift detection compared to Seldon
- BentoCloud required for managed serving (self-hosting on Kubernetes is more complex)

**Licence / IP notes**
- Apache 2.0.

---

### TrueFoundry

**Core features**
- Model registry with versioning, access rules, and gradual traffic shifting
- LLM serving with vLLM, SGLang, and TRT-LLM inference server backends
- GPU auto-scaling for inference workloads
- AI Gateway combining LLM, MCP, and Agent Gateways
- Built-in logging, request tracking, rate limiting, and role-based permissions
- Model caching and dynamic batching for low-latency inference
- Integration with Triton for GPU model serving

**Differentiating features**
- LLMOps-first design: purpose-built for large language models and agentic AI
- AI Gateway provides unified control plane for multi-provider LLM routing
- Enterprise-grade observability and access control out of the box

**UX patterns**
- Platform UI with agent/model catalogue and version visualisation
- SDK-based deployment with auto-scaling configuration

**Integration points**
- Python SDK, REST API
- vLLM, SGLang, TRT-LLM, Triton
- Kubernetes backend
- Multi-cloud support

**Known gaps**
- Smaller community and ecosystem than MLflow or SageMaker
- Weaker traditional ML (tabular/CV) model registry features compared to MLflow
- Less mature experiment tracking compared to W&B

**Licence / IP notes**
- Proprietary commercial platform.

---

### Verta.ai

**Core features**
- Enterprise model catalogue with centralised organisation of model versions
- EU AI Act compliance documentation: configurable checklists and governance fields
- Model monitoring and drift detection
- Approval and review workflows for model promotion
- Collaboration features for cross-functional stakeholders (legal, risk, IT, data science)
- Role-based access control
- Dataset provenance and lineage tracking

**Differentiating features**
- Only platform explicitly built around regulatory compliance (EU AI Act, NIST AI RMF)
- Configurable compliance checklists that map to regulatory artefact requirements
- Designed for regulated industries (financial services, healthcare)

**UX patterns**
- Governance-centric UI: approval workflows, compliance status, and documentation templates are first-class
- Stakeholder collaboration tools for non-technical reviewers

**Integration points**
- Python SDK, REST API
- Integrates with common ML training frameworks
- Enterprise SSO and access control

**Known gaps**
- Smaller market presence; less community and third-party tooling than MLflow or W&B
- Less experiment tracking capability; focused on governance rather than training workflow
- Custom pricing; limited transparency

**Licence / IP notes**
- Proprietary. No open-source component identified.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Model versioning with automatic version increment
- Metadata tagging and annotation on models and versions
- Lineage tracing from model version to training run, dataset, and code commit
- Lifecycle stage management (staging, production, archived) with approval transitions
- REST API for programmatic access
- Python SDK for integration with training workflows
- Web UI for browsing and searching registered models

### Differentiating Features
- A/B traffic splitting and canary deployment at the serving layer (SageMaker, Seldon, KServe, Databricks)
- Regulatory compliance documentation fields mapped to EU AI Act Annex IV requirements (Verta.ai; emerging in MLflow)
- Scale-to-zero serverless inference (KServe, SageMaker Serverless)
- Multi-armed bandit routing for adaptive traffic optimisation (Seldon Core)
- Explainability integration at serving time (Vertex AI, Seldon + Alibi, W&B)
- LLM-specific registry features: prompt version tracking, adapter (LoRA) weight management (TrueFoundry, Databricks)
- Automated drift monitoring with rollback triggers (Seldon + Alibi Detect, Vertex AI)
- Cross-account or cross-team model sharing with governance (SageMaker, Databricks Unity Catalog)

### Underserved Areas / Opportunities
- **EU AI Act compliance automation**: All platforms are behind on auto-generating the Annex IV technical documentation bundle from existing registry metadata; Verta.ai is the only purpose-built option but has limited market reach
- **Intelligent rollback**: No platform automatically triggers rollback based on compound signals (prediction drift + downstream business KPI degradation); this is manual or rule-based everywhere
- **LLM model management**: Prompt versioning, adapter weight tracking, and quantisation variant management are nascent features; no single platform handles the full LLM lifecycle alongside traditional ML
- **Natural-language querying**: Non-technical stakeholders cannot query model registries in plain language; all interfaces require familiarity with MLOps concepts
- **Unified multi-cloud registry**: No vendor-neutral open-source registry spans multiple cloud providers' serving infrastructure without custom integration
- **Root cause analysis for degradation**: No platform automatically correlates model degradation with upstream changes (feature store updates, data schema changes, infrastructure events)

### AI-Augmentation Candidates
- Auto-generating model cards and EU AI Act Annex IV documentation from structured registry metadata, training run data, and evaluation results
- AI-assisted A/B experiment design: recommending traffic allocation and stopping criteria based on historical variance and business objective
- Drift root cause analysis: correlating prediction distribution changes with upstream data, code, and infrastructure change events
- Natural-language search and comparison interface for model registries
- Automated rollback policy generation: learning optimal rollback thresholds from historical degradation events

---

## Legal & IP Summary

No patent concerns were identified across the open-source tools surveyed (MLflow, ClearML, Kubeflow Model Registry, KServe, Seldon Core, BentoML — all Apache 2.0). The commercial platforms (W&B, SageMaker, Vertex AI, Azure ML, Databricks, TrueFoundry, Verta.ai) are proprietary but their APIs and protocols are publicly documented. The Open Inference Protocol (KServe/CNCF) and ONNX format are open standards with no known IP encumbrances. Teams building on top of these open-source foundations face no licence compatibility risks. The primary risk is roadmap dependency on Databricks for MLflow governance direction.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Model versioning with automatic increment and immutable version metadata
- Alias-based deployment references (e.g. `champion`, `challenger`) decoupling consuming systems from version numbers
- Lineage tracing: model version → training run → dataset → code commit
- Stage lifecycle with approval workflow (staging → production → archived)
- REST API with OpenAPI specification; Python SDK
- Basic web UI for browsing, searching, and tagging models

**Should-have (v1.1)**
- EU AI Act Annex IV documentation auto-generation from registry metadata
- A/B traffic splitting configuration integrated with serving layer
- Drift monitoring integration with alert thresholds
- Role-based access control and team namespacing
- Webhook/event triggers on model version registration and stage transitions

**Nice-to-have (backlog)**
- Natural-language search and model comparison interface
- AI-assisted rollback policy generation and automated rollback on compound degradation signals
- LLM-specific registry fields: prompt version, adapter weights, quantisation variant
- Multi-armed bandit traffic routing with live business metric feedback
- Root cause analysis assistant for model degradation events
