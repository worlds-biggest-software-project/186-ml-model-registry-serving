# ML Model Registry & Serving

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native model registry and serving platform that unifies versioning, A/B traffic management, monitoring, and rollback across clouds and frameworks.

ML Model Registry & Serving is a vendor-neutral platform for cataloguing, governing, and deploying machine learning models. It targets ML engineers, MLOps teams, and platform groups who need production-grade model lifecycle management without the lock-in of cloud-specific or compute-bundled offerings.

---

## Why ML Model Registry & Serving?

- Incumbent registries are increasingly tied to specific compute or data platforms — MLflow's roadmap is steered by Databricks, W&B was acquired by CoreWeave in 2025, and Neptune.ai's SaaS is shutting down in March 2026 following its OpenAI acquisition.
- Cloud-native registries (SageMaker, Vertex AI, Azure ML) deliver strong features but create deep lock-in and bill on opaque, compute-based pricing.
- No platform automatically generates EU AI Act Annex IV technical documentation from existing registry metadata — Verta.ai is the only purpose-built compliance option but has limited market reach.
- LLM lifecycle artefacts (prompt versions, LoRA adapters, quantisation variants) are nascent across all incumbents; no single registry handles traditional ML and LLMs uniformly.
- Commercial SaaS pricing of $49–$99/user/month for teams (and six-figure enterprise contracts) leaves a clear gap for an open-source, self-hostable alternative built on open standards.

---

## Key Features

### Registry Core

- Model versioning with automatic version increment and immutable version metadata
- Alias-based deployment references (e.g. `champion`, `challenger`) decoupling consumers from version numbers
- Lifecycle stages (staging, production, archived) with approval workflow and comment history
- Tag and annotation support across models and versions
- Lineage tracing from model version to training run, dataset, and code commit

### APIs and SDKs

- REST API with OpenAPI specification
- Python SDK for integration with training pipelines
- Webhook and event triggers on version registration and stage transitions
- Compatibility with the MLflow `MLmodel` format and ONNX serialisation

### Serving and Traffic Management

- A/B traffic splitting and canary rollout configuration integrated with serving layers
- Support for Open Inference Protocol targets (KServe) and Kubernetes-native serving (Seldon Core)
- Drift monitoring integration with alert thresholds
- Pre/post-processing transformer pipeline support via standard serving runtimes

### Governance and Compliance

- EU AI Act Annex IV documentation auto-generation from registry metadata
- Configurable compliance checklists mapped to regulatory artefact requirements
- Role-based access control and team namespacing
- Dataset provenance and lineage tracking
- Audit trail for stage transitions, approvals, and deployments

### LLM Lifecycle Support

- Prompt version tracking
- Adapter (LoRA) weight management
- Quantisation variant registration

---

## AI-Native Advantage

The platform applies AI to the parts of the model lifecycle that incumbents leave manual: auto-generating model cards and EU AI Act Annex IV documentation from structured registry metadata, training run data, and evaluation results; correlating prediction drift with upstream feature store, data, and infrastructure changes for root cause analysis; and supporting natural-language search and comparison so non-technical stakeholders can query the registry directly. Intelligent rollback policies learn thresholds from historical degradation events and trigger on compound signals (prediction drift plus downstream business KPI degradation) rather than fixed rules.

---

## Tech Stack & Deployment

The project is designed to be self-hostable on Kubernetes with a managed cloud option, integrating with the Open Inference Protocol (KServe), Seldon Core, and the MLflow `MLmodel` and ONNX standards. APIs are contract-first via OpenAPI, with Python and Go client libraries. Observability uses OpenTelemetry for serving metrics. The registry is deliberately neutral: it integrates with — rather than replaces — existing serving engines (KServe, Seldon Core, BentoML) and training frameworks (PyTorch, TensorFlow, scikit-learn, HuggingFace).

---

## Market Context

The broader MLOps market is valued at approximately $4–6 billion in 2026 and is projected to exceed $20 billion by 2030, growing at roughly 40% CAGR (research.md). Commercial registry pricing ranges from $49/user/month (Neptune.ai Teams) to $99/user/month (Comet ML) to custom enterprise contracts; cloud provider offerings bill on compute consumption. Primary buyers are ML engineers and MLOps teams running multiple production models, and enterprises in regulated industries (financial services, healthcare) needing EU AI Act compliance documentation.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
