# ML Model Registry & Serving

> Candidate #186 · Researched: 2026-05-02

## Existing Products and Software Packages

| Tool | Description | Type | Pricing | Strengths / Weaknesses |
|------|-------------|------|---------|------------------------|
| MLflow | Open-source end-to-end ML lifecycle: experiment tracking, registry, serving | Open-source / Databricks managed | Free (OSS); Databricks pricing varies | Industry-standard registry; Databricks acquisition adds enterprise governance via Unity Catalog |
| Weights & Biases (W&B) | Experiment tracking, model registry, lineage, evaluation | Commercial | ~$50/user/month (Teams); enterprise custom | Developer-favourite UI; acquired by CoreWeave in 2025, tightly integrated with that compute stack |
| Neptune.ai | Metadata store and model registry for large-scale ML teams | Commercial | $49/user/month (Teams); enterprise custom | Infrastructure-agnostic; strong governance and scalability; less end-to-end than W&B |
| Comet ML | Experiment tracking with model registry and production monitoring | Commercial | ~$99/user/month | Full-featured; higher cost than alternatives at the same tier |
| ClearML | Open-source MLOps platform with integrated model registry | Open-source / Commercial | Free (OSS); managed hosting available | Good all-in-one option; less polished UI than W&B |
| Databricks Mosaic AI | Managed MLflow + Unity Catalog registry + model serving endpoints | Commercial | Databricks compute-based billing | Best choice for Databricks-native teams; vendor lock-in risk |
| Amazon SageMaker | AWS-managed ML platform with model registry, endpoints, A/B testing | Commercial | AWS usage-based pricing | Deep AWS integration; complex pricing; best for AWS-committed enterprises |
| Azure ML | Azure-managed ML platform with model registry and managed endpoints | Commercial | Azure compute-based pricing | Strong enterprise governance; requires Azure ecosystem buy-in |
| Vertex AI (Google) | GCP-managed ML platform with model registry and online prediction | Commercial | GCP usage-based pricing | Strong for TensorFlow and AutoML workflows; GCP-native teams |
| Verta.ai | Enterprise MLOps with governance, EU AI Act reporting, model serving | Commercial | Custom enterprise | Built for regulated industries and compliance documentation; smaller market presence |

## Relevant Industry Standards or Protocols

- **MLflow Model Format (MLmodel)** — De facto standard for packaging ML models with metadata; supported by most major platforms
- **ONNX (Open Neural Network Exchange)** — Framework-agnostic model serialisation format enabling cross-platform deployment
- **Seldon Core / KServe** — Open-source Kubernetes-native model serving standards increasingly used as deployment targets
- **EU AI Act Technical Documentation Requirements** — Mandates versioned model metadata, dataset provenance, and performance documentation for high-risk AI systems
- **NIST AI RMF (AI Risk Management Framework)** — US framework for AI governance including model versioning and lifecycle accountability
- **OpenTelemetry** — Emerging standard for model serving observability (latency, drift, prediction distribution metrics)

## Available Research Materials

1. MLflow (2026). *ML Model Registry Documentation*. https://mlflow.org/docs/latest/ml/model-registry
2. Dynatrace (2026). *Introducing AI Model Versioning and A/B Testing for Smarter LLM Services*. https://www.dynatrace.com/news/blog/the-rise-of-agentic-ai-part-6-introducing-ai-model-versioning-and-a-b-testing-for-smarter-llm-services/
3. Gurukul Galaxy (2026). *Top 10 Model Registry Tools: Features, Pros, Cons & Comparison*. https://gurukulgalaxy.com/blog/top-10-model-registry-tools-features-pros-cons-comparison/
4. Neptune.ai (2026). *Best ML Model Registry Tools*. https://neptune.ai/blog/ml-model-registry-best-tools
5. Azumo (2026). *Top 10 MLOps Platforms for Scalable AI in Summer 2026*. https://azumo.com/artificial-intelligence/ai-insights/mlops-platforms
6. Uplatz (2025). *The 2025 MLOps Landscape: A Comparative Analysis of MLflow, W&B, and Neptune*. https://uplatz.com/blog/the-2025-mlops-landscape-a-comparative-analysis-of-mlflow-weights-biases-and-neptune/
7. OneUptime (2026). *How to Implement Model Registry*. https://oneuptime.com/blog/post/2026-01-25-model-registry/view
8. Inference Systems Authority (2026). *Inference Model Versioning and Rollback Strategies*. https://inferencesystemsauthority.com/inference-versioning-and-rollback

## Market Research

**Market Size:** The broader MLOps market is valued at approximately $4–6 billion in 2026 and is projected to exceed $20 billion by 2030, growing at a CAGR of around 40%. Model registry and serving represent a core layer within this stack.

**Funding:** Weights & Biases raised over $250M before being acquired by CoreWeave in 2025. Databricks (parent of MLflow) raised $10B+ in total funding. Verta.ai raised $16M. Most specialist registries are now features within larger ML platform companies rather than standalone funded entities.

**Pricing Landscape:** Open-source MLflow is free to self-host; Databricks-managed pricing is compute-based and opaque. Commercial SaaS tools range from $49–$99/user/month for teams to custom six-figure enterprise contracts. Cloud provider offerings (SageMaker, Vertex, Azure ML) bill on compute consumption.

**Key Buyer Personas:** ML engineers and MLOps teams at companies with multiple models in production; enterprises in regulated industries (financial services, healthcare) requiring EU AI Act compliance documentation; platform teams building internal ML infrastructure for large data science organisations.

**Notable Trends:** EU AI Act now requires detailed technical documentation for high-risk AI models, driving demand for governance-ready registries. CoreWeave's acquisition of W&B creates a compute-plus-tooling bundle play. Tecton (feature store closely related to model serving) was acquired by Databricks in August 2025, consolidating the ML infrastructure market. LLM model management (prompt versioning, adapter tracking) is expanding the registry concept beyond traditional ML.

## AI-Native Opportunity

- Automated model card generation that synthesises training metadata, evaluation metrics, and dataset lineage into compliance-ready documentation for EU AI Act requirements
- Intelligent A/B traffic routing that dynamically adjusts serving weights based on live business metric signals rather than fixed experimental schedules
- Drift-triggered rollback that monitors prediction distribution and downstream KPIs simultaneously, initiating automatic rollback when compound degradation is detected
- Natural-language model comparison interface enabling non-technical stakeholders to query differences between model versions and understand business impact
- LLM-powered root cause analysis for model degradation events — correlating data drift, infrastructure changes, and upstream feature store updates to identify failure causes
