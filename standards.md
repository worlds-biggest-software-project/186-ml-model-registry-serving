# Standards & API Reference

> Project: ML Model Registry & Serving · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 42001:2023 — Artificial Intelligence Management System**
- URL: https://www.iso.org/standard/81230.html
- Specifies requirements for establishing, implementing, maintaining, and continually improving an AI management system within organisations. Directly relevant for governance metadata captured by a model registry (risk classification, intended use, monitoring requirements).

**ISO/IEC 25012:2008 — Software and Systems Engineering: Data Quality Model**
- URL: https://www.iso.org/standard/35736.html
- Defines data quality characteristics relevant to model training data documentation and dataset provenance requirements in a registry.

**ISO/IEC 27001:2022 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- Relevant for access control, audit logging, and credential management in a model registry handling proprietary model artefacts.

---

### W3C & IETF Standards

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Baseline HTTP semantics governing the REST API design for model registry and serving endpoints.

**RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines link relations used in REST APIs for hypermedia navigation between model versions, registry resources, and serving endpoints.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- Standard for delegated authorisation used by all major model registries (SageMaker IAM, Vertex AI IAM, Databricks Unity Catalog) to control access to registry resources.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Token format used by most model registry APIs for service authentication.

**OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0; used for enterprise SSO integration with model registry platforms.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1.0 / 3.2.0**
- URLs: https://spec.openapis.org/oas/v3.1.0.html · https://spec.openapis.org/oas/v3.2.0.html
- The standard for describing REST APIs. Kubeflow Model Registry uses a contract-first design with an OpenAPI specification (`model-registry.yaml`). MLflow's REST API and most commercial registries publish OpenAPI specs. Increasingly critical as AI agents consume APIs described by OpenAPI documents.

**MLflow REST API (model-registry/2.0)**
- URL: https://mlflow.org/docs/latest/rest-api.html
- De facto REST API standard for model registry operations. Defines endpoints for registered models, model versions, transitions, aliases, and tags. Broadly implemented or referenced by competing platforms.

**MLmodel Format (MLflow Model Packaging Specification)**
- URL: https://mlflow.org/docs/latest/ml/deployment/
- De facto standard for packaging ML models with a `MLmodel` YAML manifest declaring model flavours, input/output signatures, dependencies, and metadata. Supported by SageMaker, Azure ML, Databricks, and most serving platforms.

**ONNX (Open Neural Network Exchange)**
- URL: https://onnx.ai · https://github.com/onnx/onnx
- Framework-agnostic model serialisation format enabling cross-platform model portability. Supported as a first-class flavour in MLflow and by all major serving platforms. The ONNX spec defines the operator set, graph IR, and runtime interface.

**Protocol Buffers (protobuf) 3**
- URL: https://protobuf.dev/programming-guides/proto3/
- Binary serialisation format used by gRPC inference APIs (KServe V2 gRPC, TensorFlow Serving). Relevant for high-throughput serving where JSON overhead is prohibitive.

**Kubeflow Model Registry OpenAPI Specification (v1alpha3)**
- URL: https://www.kubeflow.org/docs/components/model-registry/reference/rest-api/
- Contract-first REST API specification for the open-source Kubeflow Model Registry, defining endpoints for registered models, model versions, and model artefacts.

---

### Serving Inference Protocols

**Open Inference Protocol (KServe V2 Inference Protocol)**
- URL: https://kserve.github.io/website/docs/concepts/architecture/data-plane/v2-protocol · https://github.com/kserve/open-inference-protocol
- CNCF-backed standard REST and gRPC inference protocol. Defines `/health`, `/metadata`, `/infer`, and `/explain` endpoints. Implemented by KServe, Seldon Core, Triton Inference Server, and BentoML. Promotes interoperability across serving runtimes. OpenAPI YAML spec published at `rest_predict_v2.yaml`.

**TensorFlow Serving REST & gRPC API**
- URL: https://www.tensorflow.org/tfx/serving/api_rest
- Widely referenced REST inference standard from Google; the precursor to the V2 Open Inference Protocol. Still used by Vertex AI and TF-native serving pipelines.

**TorchServe Management and Inference API**
- URL: https://pytorch.org/serve/rest_api.html
- PyTorch's native serving standard. Defines management endpoints for model registration, versioning, and scaling, and inference endpoints for prediction.

---

### Regulatory & Governance Standards

**EU AI Act — Regulation (EU) 2024/1689**
- URL: https://artificialintelligenceact.eu · https://digital-strategy.ec.europa.eu/en/policies/regulatory-framework-ai
- Mandatory for high-risk AI systems operating in EU markets (enforceable from August 2, 2026). Article 12 requires automatic event logging integrated into the system's core design. Article 18 mandates technical documentation be drawn up before market placement, kept up-to-date, and retained for 10 years. Annex IV prescribes nine documentation categories that map directly to model registry metadata (intended purpose, training data, performance metrics, human oversight measures, post-market monitoring). Model registries are the natural implementation layer for EU AI Act compliance artefacts.

**NIST AI Risk Management Framework (AI RMF 1.0) — NIST AI 100-1**
- URL: https://nvlpubs.nist.gov/nistpubs/ai/nist.ai.100-1.pdf · https://airc.nist.gov/airmf-resources/airmf/
- US voluntary framework for AI governance structured around four functions: Govern, Map, Measure, and Manage. Defines accountability, transparency, and lifecycle management requirements that a model registry directly addresses through versioning, lineage, and stage approval workflows.

**NIST AI 600-1 — Generative AI Profile**
- URL: https://nvlpubs.nistpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf
- 2024 extension of the AI RMF specifically addressing generative AI risks. Relevant for LLM model registry requirements including prompt versioning and fine-tuned adapter tracking.

**OWASP Top 10 for ML Security**
- URL: https://owasp.org/www-project-machine-learning-security-top-10/
- Identifies ML-specific security risks including model poisoning, supply chain attacks on model artefacts, and inference API vulnerabilities. Relevant to access control and integrity verification in a model registry.

---

### Observability Standards

**OpenTelemetry**
- URL: https://opentelemetry.io · https://opentelemetry.io/docs/specs/otel/
- Emerging standard for distributed tracing, metrics, and logs. Increasingly adopted by model serving platforms (BentoML, TrueFoundry) for capturing inference latency, throughput, and error rates. A model registry should emit OpenTelemetry signals for deployment events and serving health.

**Prometheus Data Model and Exposition Format**
- URL: https://prometheus.io/docs/concepts/data_model/ · https://prometheus.io/docs/instrumenting/exposition_formats/
- De facto standard for Kubernetes-native metrics exposition. KServe and Seldon Core emit Prometheus metrics for per-model and per-version inference statistics, enabling drift and latency monitoring via Grafana dashboards.

---

## Similar Products — Developer Documentation & APIs

### MLflow

- **Description:** Open-source AI platform for the full ML lifecycle. The Model Registry component provides versioning, aliasing, stage management, and lineage for ML models.
- **API Documentation:** https://mlflow.org/docs/latest/rest-api.html
- **SDKs/Libraries:** Python (`pip install mlflow`), R client; also accessible via Databricks SDK
- **Developer Guide:** https://mlflow.org/docs/latest/ml/model-registry/workflow
- **Standards:** REST/JSON, OpenAPI (unofficial), MLmodel packaging format
- **Authentication:** Bearer token (Databricks PAT), basic auth for OSS deployments

---

### Kubeflow Model Registry

- **Description:** Open-source Kubernetes-native metadata registry for ML models, versions, and artefacts. Contract-first REST API defined in OpenAPI; Python and Go client libraries available.
- **API Documentation:** https://www.kubeflow.org/docs/components/model-registry/reference/rest-api/
- **SDKs/Libraries:** Python client (`pip install model-registry`), Go library
- **Developer Guide:** https://www.kubeflow.org/docs/components/model-registry/getting-started/
- **Standards:** REST/JSON, OpenAPI 3.x (`model-registry.yaml`)
- **Authentication:** Kubernetes RBAC; service account tokens

---

### Weights & Biases (W&B) Registry

- **Description:** Commercial MLOps platform with experiment tracking, artifact registry, and model lifecycle management. Python SDK with tight framework integrations.
- **API Documentation:** https://docs.wandb.ai/ · https://docs.wandb.ai/models/registry/model_registry
- **SDKs/Libraries:** Python (`pip install wandb`); JavaScript client (limited)
- **Developer Guide:** https://docs.wandb.ai/guides/models/
- **Standards:** REST/JSON, GraphQL (internal query interface)
- **Authentication:** API keys; enterprise SSO via SAML/OIDC

---

### Amazon SageMaker Model Registry

- **Description:** Fully-managed AWS registry for ML model cataloguing, approval workflows, lineage, and deployment. Deep integration with the broader AWS ecosystem.
- **API Documentation:** https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html
- **SDKs/Libraries:** Python (`pip install sagemaker`, `boto3`), AWS CLI
- **Developer Guide:** https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html
- **Standards:** REST/JSON (AWS API style), AWS SigV4 authentication
- **Authentication:** AWS IAM (SigV4 request signing, roles, and policies)

---

### Vertex AI Model Registry

- **Description:** GCP-managed model registry with version management, evaluation, explainability, and online/batch serving endpoints.
- **API Documentation:** https://cloud.google.com/vertex-ai/docs/reference/rest · https://docs.cloud.google.com/vertex-ai/docs/model-registry/introduction
- **SDKs/Libraries:** Python (`pip install google-cloud-aiplatform`), Go, Java, Node.js, .NET
- **Developer Guide:** https://cloud.google.com/vertex-ai/docs/model-registry/introduction
- **Standards:** REST/JSON, gRPC, OpenAPI
- **Authentication:** Google Cloud IAM (OAuth 2.0 service account credentials)

---

### KServe (Open Inference Protocol)

- **Description:** Kubernetes-native serverless model serving platform implementing the CNCF Open Inference Protocol. Provides standardised REST and gRPC inference endpoints for any ML framework.
- **API Documentation:** https://kserve.github.io/website/docs/concepts/architecture/data-plane/v2-protocol
- **SDKs/Libraries:** Python inference client (`pip install kserve`); gRPC client libraries in multiple languages
- **Developer Guide:** https://kserve.github.io/website/docs/
- **Standards:** Open Inference Protocol V2 (REST/JSON + gRPC/protobuf), OpenAPI
- **Authentication:** Kubernetes RBAC; Istio mTLS for service-to-service

---

### Seldon Core

- **Description:** Kubernetes-native model serving with canary, A/B, and multi-armed bandit patterns. Includes Alibi Detect for drift monitoring and Alibi Explain for explainability.
- **API Documentation:** https://docs.seldon.io/projects/seldon-core/en/latest/
- **SDKs/Libraries:** Python (`pip install seldon-core`); REST and gRPC inference clients
- **Developer Guide:** https://docs.seldon.io/projects/seldon-core/en/latest/workflow/overview.html
- **Standards:** V2 Inference Protocol (REST/JSON, gRPC), Kubernetes CRD
- **Authentication:** Kubernetes RBAC; Istio mTLS

---

### BentoML

- **Description:** Python framework for packaging and serving any ML model as a production-grade REST API. Supports adaptive batching and deploys to BentoCloud or any Kubernetes cluster.
- **API Documentation:** https://docs.bentoml.com/en/latest/
- **SDKs/Libraries:** Python (`pip install bentoml`); auto-generated REST client from service definition
- **Developer Guide:** https://docs.bentoml.com/en/latest/get-started/quickstart.html
- **Standards:** REST/JSON, OpenTelemetry (observability), Docker/OCI images
- **Authentication:** BentoCloud API tokens; custom auth middleware for self-hosted

---

### Databricks Mosaic AI Model Serving

- **Description:** Managed MLflow + Unity Catalog registry with auto-scaling REST serving endpoints, A/B testing, and integrated monitoring. Part of the Databricks Lakehouse Platform.
- **API Documentation:** https://docs.databricks.com/aws/en/machine-learning/model-serving/ · https://docs.databricks.com/aws/en/mlflow/
- **SDKs/Libraries:** Python (`pip install databricks-sdk`, `pip install mlflow`), Databricks CLI
- **Developer Guide:** https://docs.databricks.com/aws/en/machine-learning/model-serving/create-manage-serving-endpoints.html
- **Standards:** REST/JSON, MLflow REST API
- **Authentication:** Databricks Personal Access Tokens (PATs); OAuth M2M for service principals

---

## Notes

- **Open Inference Protocol adoption**: The KServe V2 / Open Inference Protocol is gaining adoption as the de facto open standard for model inference APIs. NVIDIA Triton, Seldon, and BentoML all implement it. A new model registry and serving platform should consider implementing or at minimum exposing a V2-compatible inference endpoint.

- **EU AI Act deadline pressure**: August 2, 2026 is the enforcement date for high-risk AI systems under the EU AI Act. Model registries are the natural home for the required Annex IV documentation (intended purpose, training data provenance, performance metrics, human oversight mechanisms). This is the most significant near-term regulatory driver for registry feature investment.

- **OpenTelemetry for ML observability**: The OpenTelemetry specification does not yet have a stable semantic convention for ML inference metrics (input shapes, prediction distributions, feature drift). The CNCF ML observability working group is developing conventions; this is an area where an AI-native registry could define emerging standards.

- **MLflow REST API versioning**: The MLflow REST API is at `/api/2.0/mlflow/...` and is effectively frozen at this version for backward compatibility. MLflow 3 introduces new capabilities but maintains the 2.0 REST API surface. Any compatible platform should align with this API version for interoperability.
