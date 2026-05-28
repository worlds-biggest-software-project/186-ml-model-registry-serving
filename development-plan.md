# ML Model Registry & Serving — Phased Development Plan

> Project: 186-ml-model-registry-serving · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (API) | Python 3.12+ | Dominant language in the ML ecosystem; all target users (ML engineers, MLOps teams) work in Python; every competing product (MLflow, W&B, ClearML, BentoML) provides Python SDKs; enables native integration with PyTorch, TensorFlow, scikit-learn, ONNX Runtime |
| Language (CLI/SDK) | Python 3.12+ | Same language as the API server eliminates context-switching; Pydantic v2 for shared type definitions between SDK and server; `click` for CLI |
| API framework | FastAPI 0.115+ | Native OpenAPI 3.1 spec generation (contract-first is a project requirement); async support for long-running model uploads and serving proxy calls; Pydantic v2 integration for request/response validation; dependency injection for auth and database sessions |
| Database | PostgreSQL 16 | JSONB + GIN indexes power the hybrid relational/JSONB data model (data-model-suggestion-3); partitioning for time-series serving_metrics and audit_log; pg_trgm for full-text search on model names and tags; mature migration tooling (Alembic); referential integrity for governance and compliance records |
| ORM / query builder | SQLAlchemy 2.0 + Alembic | Async support via asyncpg; mapped dataclasses for type safety; Alembic for versioned migrations; JSONB column support with type-aware operators |
| Task queue | Celery 5.x + Redis 7 | Handles async model artifact uploads, drift detection checks, webhook delivery, compliance document generation; Redis as both broker and result backend; beat scheduler for periodic drift monitoring checks |
| Object storage | S3-compatible (MinIO for self-hosted, AWS S3 / GCS for cloud) | Model artifacts range from MBs (sklearn) to hundreds of GBs (LLMs); S3 API is the de facto standard supported by all cloud providers; MinIO provides identical API for self-hosted Kubernetes deployments |
| Frontend | React 19 + Vite 6 | Lightweight SPA for registry browsing, model comparison, lineage visualisation, and compliance dashboards; Vite for fast development builds; no SSR needed (internal tool, not SEO-sensitive) |
| UI components | shadcn/ui + Tailwind CSS 4 | Accessible, composable components; TanStack Table for model/version lists with sorting and filtering; React Flow for lineage graph visualisation; Recharts for metrics dashboards |
| Authentication | OAuth 2.0 / OIDC via `authlib` + JWT | Enterprise SSO requirement from features.md; OIDC for identity federation; JWT (RS256) for stateless API authentication; API key support for service principals and CI/CD pipelines |
| Serving integration | KServe + Seldon Core (Kubernetes CRDs) | Open Inference Protocol (V2) compliance; KServe for serverless scale-to-zero; Seldon for advanced A/B and multi-armed bandit routing; the registry orchestrates these — it does not re-implement serving |
| Observability | OpenTelemetry SDK + Prometheus metrics endpoint | OTel for distributed tracing across registry API, artifact storage, and serving proxy; Prometheus exposition format for serving metrics (aligned with KServe/Seldon native metrics); Grafana dashboards |
| Containerisation | Docker + docker-compose + Helm chart | `docker compose up` for local development (PostgreSQL, Redis, MinIO, API server, worker, frontend); Helm chart for production Kubernetes deployment |
| Testing | pytest + httpx (async) + Playwright | pytest for unit and integration tests; httpx.AsyncClient for FastAPI test client; Playwright for frontend E2E; testcontainers-python for PostgreSQL integration tests |
| Code quality | Ruff (linter + formatter) + mypy (strict) | Ruff replaces flake8 + black + isort with a single tool; mypy strict mode catches type errors in Pydantic models and SQLAlchemy queries |
| Package manager | uv | Fast dependency resolution; lockfile support; replaces pip + pip-tools; compatible with pyproject.toml |
| Documentation | MkDocs Material | API reference auto-generated from OpenAPI spec; user guides in Markdown; deployed alongside the application |

### Project Structure

```
ml-model-registry/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── helm/
│   └── ml-model-registry/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
├── alembic.ini
├── alembic/
│   └── versions/
├── .env.example
├── src/
│   ├── registry/
│   │   ├── __init__.py
│   │   ├── main.py                    # FastAPI app factory
│   │   ├── config.py                  # Settings via pydantic-settings
│   │   ├── db/
│   │   │   ├── __init__.py
│   │   │   ├── engine.py              # SQLAlchemy async engine
│   │   │   ├── session.py             # Session factory + dependency
│   │   │   └── models/
│   │   │       ├── __init__.py
│   │   │       ├── organization.py
│   │   │       ├── user.py
│   │   │       ├── registered_model.py
│   │   │       ├── model_version.py
│   │   │       ├── model_alias.py
│   │   │       ├── dataset.py
│   │   │       ├── lineage.py
│   │   │       ├── serving.py
│   │   │       ├── monitoring.py
│   │   │       ├── compliance.py
│   │   │       ├── prompt_version.py
│   │   │       ├── webhook.py
│   │   │       └── audit.py
│   │   ├── api/
│   │   │   ├── __init__.py
│   │   │   ├── deps.py                # FastAPI dependencies (auth, db)
│   │   │   ├── v1/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── router.py          # v1 router aggregation
│   │   │   │   ├── models.py          # Registered models endpoints
│   │   │   │   ├── versions.py        # Model versions endpoints
│   │   │   │   ├── aliases.py         # Alias management
│   │   │   │   ├── datasets.py
│   │   │   │   ├── lineage.py
│   │   │   │   ├── serving.py
│   │   │   │   ├── monitoring.py
│   │   │   │   ├── compliance.py
│   │   │   │   ├── prompts.py
│   │   │   │   ├── webhooks.py
│   │   │   │   └── search.py
│   │   │   └── schemas/
│   │   │       ├── __init__.py
│   │   │       ├── models.py
│   │   │       ├── versions.py
│   │   │       ├── serving.py
│   │   │       ├── monitoring.py
│   │   │       ├── compliance.py
│   │   │       └── common.py
│   │   ├── services/
│   │   │   ├── __init__.py
│   │   │   ├── model_service.py
│   │   │   ├── version_service.py
│   │   │   ├── artifact_service.py    # S3 upload/download/hash
│   │   │   ├── alias_service.py
│   │   │   ├── lineage_service.py
│   │   │   ├── serving_service.py     # KServe/Seldon orchestration
│   │   │   ├── traffic_service.py     # A/B split management
│   │   │   ├── drift_service.py       # Drift detection engine
│   │   │   ├── compliance_service.py  # EU AI Act doc generation
│   │   │   ├── webhook_service.py
│   │   │   ├── search_service.py
│   │   │   └── audit_service.py
│   │   ├── workers/
│   │   │   ├── __init__.py
│   │   │   ├── celery_app.py
│   │   │   ├── artifact_tasks.py
│   │   │   ├── drift_tasks.py
│   │   │   ├── webhook_tasks.py
│   │   │   ├── compliance_tasks.py
│   │   │   └── rollback_tasks.py
│   │   ├── auth/
│   │   │   ├── __init__.py
│   │   │   ├── jwt.py
│   │   │   ├── api_key.py
│   │   │   ├── oidc.py
│   │   │   └── rbac.py
│   │   ├── integrations/
│   │   │   ├── __init__.py
│   │   │   ├── kserve/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── client.py
│   │   │   │   └── models.py
│   │   │   ├── seldon/
│   │   │   │   ├── __init__.py
│   │   │   │   ├── client.py
│   │   │   │   └── models.py
│   │   │   ├── mlflow/
│   │   │   │   ├── __init__.py
│   │   │   │   └── compat.py          # MLflow REST API compatibility layer
│   │   │   └── storage/
│   │   │       ├── __init__.py
│   │   │       ├── base.py
│   │   │       ├── s3.py
│   │   │       └── gcs.py
│   │   └── telemetry/
│   │       ├── __init__.py
│   │       ├── otel.py
│   │       └── metrics.py
│   └── sdk/
│       ├── __init__.py
│       ├── client.py                  # Python SDK client
│       ├── models.py                  # SDK data classes
│       └── cli.py                     # CLI entry point
├── frontend/
│   ├── package.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── components/
│   │   │   ├── models/
│   │   │   ├── versions/
│   │   │   ├── serving/
│   │   │   ├── lineage/
│   │   │   ├── compliance/
│   │   │   └── common/
│   │   ├── pages/
│   │   ├── hooks/
│   │   ├── api/
│   │   └── lib/
│   └── tests/
├── tests/
│   ├── conftest.py
│   ├── unit/
│   │   ├── test_model_service.py
│   │   ├── test_version_service.py
│   │   ├── test_artifact_service.py
│   │   ├── test_alias_service.py
│   │   ├── test_drift_service.py
│   │   └── test_compliance_service.py
│   ├── integration/
│   │   ├── test_api_models.py
│   │   ├── test_api_versions.py
│   │   ├── test_api_serving.py
│   │   └── test_api_webhooks.py
│   ├── e2e/
│   │   └── test_model_lifecycle.py
│   └── fixtures/
│       ├── sample_model.pkl
│       ├── sample_model.onnx
│       └── sample_mlmodel.yaml
└── docs/
    ├── mkdocs.yml
    └── docs/
        ├── index.md
        ├── quickstart.md
        ├── api-reference.md
        └── deployment.md
```

---

## Phase 1: Foundation — Project Scaffolding, Database, and Configuration

### Purpose

Establish the project skeleton, database connectivity, configuration management, and the core SQLAlchemy models for the registry. After this phase, the application boots, connects to PostgreSQL, runs migrations, and exposes a health-check endpoint. Every subsequent phase builds on this foundation.

### Tasks

#### 1.1 — Project Scaffolding and Configuration

**What**: Create the pyproject.toml, Docker Compose stack, environment configuration, and FastAPI application factory.

**Design**:

```python
# src/registry/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Application
    app_name: str = "ml-model-registry"
    app_version: str = "0.1.0"
    debug: bool = False
    api_prefix: str = "/api/v1"

    # Database
    database_url: str = "postgresql+asyncpg://registry:registry@localhost:5432/registry"
    database_pool_size: int = 20
    database_max_overflow: int = 10

    # Redis
    redis_url: str = "redis://localhost:6379/0"

    # Object Storage
    storage_backend: str = "s3"  # "s3", "gcs", "local"
    s3_endpoint_url: str = "http://localhost:9000"
    s3_access_key: str = "minioadmin"
    s3_secret_key: str = "minioadmin"
    s3_bucket: str = "model-artifacts"
    s3_region: str = "us-east-1"

    # Auth
    jwt_secret_key: str = "change-me-in-production"
    jwt_algorithm: str = "RS256"
    jwt_expiry_minutes: int = 60
    oidc_discovery_url: str | None = None
    oidc_client_id: str | None = None
    oidc_client_secret: str | None = None

    # Telemetry
    otel_enabled: bool = False
    otel_exporter_endpoint: str = "http://localhost:4317"

    model_config = {"env_prefix": "REGISTRY_", "env_file": ".env"}
```

```python
# src/registry/main.py
from fastapi import FastAPI
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: initialise DB pool, Redis, OTel
    await init_db_engine(app.state.settings)
    yield
    # Shutdown: close pools
    await close_db_engine()

def create_app(settings: Settings | None = None) -> FastAPI:
    settings = settings or Settings()
    app = FastAPI(
        title="ML Model Registry & Serving",
        version=settings.app_version,
        lifespan=lifespan,
    )
    app.state.settings = settings
    app.include_router(health_router)
    app.include_router(v1_router, prefix=settings.api_prefix)
    return app
```

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: registry
      POSTGRES_PASSWORD: registry
      POSTGRES_DB: registry
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

  api:
    build: .
    command: uvicorn registry.main:app --host 0.0.0.0 --port 8000 --reload
    ports: ["8000:8000"]
    env_file: .env
    depends_on: [postgres, redis, minio]

  worker:
    build: .
    command: celery -A registry.workers.celery_app worker --loglevel=info
    env_file: .env
    depends_on: [postgres, redis, minio]

volumes:
  pgdata:
  miniodata:
```

**Testing**:
- `Unit: Settings loads defaults when no env vars set`
- `Unit: Settings overrides from REGISTRY_ prefixed env vars`
- `Unit: Settings reads from .env file`
- `Integration: docker compose up starts all services without errors`
- `Integration: FastAPI app starts and /health returns 200 with {"status": "ok"}`

#### 1.2 — Database Engine, Session Management, and Migrations

**What**: Set up SQLAlchemy 2.0 async engine, session dependency for FastAPI, and Alembic migration infrastructure.

**Design**:

```python
# src/registry/db/engine.py
from sqlalchemy.ext.asyncio import (
    AsyncEngine, AsyncSession, async_sessionmaker, create_async_engine,
)

engine: AsyncEngine | None = None
async_session_factory: async_sessionmaker[AsyncSession] | None = None

async def init_db_engine(settings: Settings) -> None:
    global engine, async_session_factory
    engine = create_async_engine(
        settings.database_url,
        pool_size=settings.database_pool_size,
        max_overflow=settings.database_max_overflow,
        echo=settings.debug,
    )
    async_session_factory = async_sessionmaker(engine, expire_on_commit=False)

async def close_db_engine() -> None:
    global engine
    if engine:
        await engine.dispose()

# src/registry/db/session.py
from fastapi import Depends

async def get_db_session() -> AsyncGenerator[AsyncSession, None]:
    async with async_session_factory() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

```python
# src/registry/db/models/__init__.py
from sqlalchemy.orm import DeclarativeBase, MappedAsDataclass
import sqlalchemy as sa

class Base(DeclarativeBase, MappedAsDataclass):
    """Base class for all ORM models."""
    pass

# Common mixins
class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )
    updated_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(),
        onupdate=sa.func.now(), init=False
    )
```

Alembic configured with `alembic.ini` pointing to the async engine and `env.py` importing all models for autogenerate.

**Testing**:
- `Unit: get_db_session yields a session and commits on success`
- `Unit: get_db_session rolls back on exception`
- `Integration (testcontainers): Alembic upgrade head runs all migrations without error`
- `Integration (testcontainers): Alembic downgrade base and re-upgrade succeeds (migration reversibility)`

#### 1.3 — Core Database Models: Organizations, Users, Registered Models

**What**: Define the SQLAlchemy ORM models for organizations, users, and registered_models tables following the hybrid relational + JSONB pattern from data-model-suggestion-3.

**Design**:

```python
# src/registry/db/models/organization.py
import uuid
from sqlalchemy import String, JSONB
from sqlalchemy.orm import Mapped, mapped_column, relationship

class Organization(Base, TimestampMixin):
    __tablename__ = "organizations"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    name: Mapped[str] = mapped_column(String(255), unique=True)
    slug: Mapped[str] = mapped_column(String(255), unique=True)
    settings: Mapped[dict] = mapped_column(JSONB, default_factory=dict)

    users: Mapped[list["User"]] = relationship(back_populates="organization", init=False)
    models: Mapped[list["RegisteredModel"]] = relationship(back_populates="organization", init=False)


# src/registry/db/models/user.py
class User(Base, TimestampMixin):
    __tablename__ = "users"
    __table_args__ = (sa.UniqueConstraint("organization_id", "email"),)

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    organization_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("organizations.id"))
    email: Mapped[str] = mapped_column(String(320))
    display_name: Mapped[str | None] = mapped_column(String(255), default=None)
    role: Mapped[str] = mapped_column(String(50), default="member")
    is_service_principal: Mapped[bool] = mapped_column(default=False)
    profile: Mapped[dict] = mapped_column(JSONB, default_factory=dict)

    organization: Mapped["Organization"] = relationship(back_populates="users", init=False)


# src/registry/db/models/registered_model.py
class RegisteredModel(Base, TimestampMixin):
    __tablename__ = "registered_models"
    __table_args__ = (
        sa.UniqueConstraint("organization_id", "name"),
        sa.Index("idx_models_tags", "tags", postgresql_using="gin"),
        sa.Index("idx_models_governance", "governance", postgresql_using="gin"),
    )

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    organization_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("organizations.id"))
    name: Mapped[str] = mapped_column(String(255))
    description: Mapped[str | None] = mapped_column(sa.Text, default=None)
    owner_id: Mapped[uuid.UUID | None] = mapped_column(sa.ForeignKey("users.id"), default=None)
    risk_classification: Mapped[str | None] = mapped_column(String(50), default=None)
    ai_system_type: Mapped[str | None] = mapped_column(String(100), default=None)
    tags: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    governance: Mapped[dict] = mapped_column(JSONB, default_factory=dict)

    organization: Mapped["Organization"] = relationship(back_populates="models", init=False)
    versions: Mapped[list["ModelVersion"]] = relationship(back_populates="model", init=False)
    aliases: Mapped[list["ModelAlias"]] = relationship(back_populates="model", init=False)
```

**Testing**:
- `Unit: Organization model creates with valid name and slug`
- `Unit: Organization.settings defaults to empty dict`
- `Unit: User model enforces unique (organization_id, email) constraint`
- `Unit: RegisteredModel.tags stores and retrieves JSONB correctly`
- `Integration (testcontainers): create Organization, User, RegisteredModel in transaction, verify persisted`
- `Integration (testcontainers): unique constraint on (org_id, model_name) raises IntegrityError on duplicate`

#### 1.4 — Health Check and OpenAPI Skeleton

**What**: Implement the health check endpoint and confirm OpenAPI spec auto-generation with proper metadata.

**Design**:

```python
# Health check router
from fastapi import APIRouter, Depends

health_router = APIRouter(tags=["health"])

@health_router.get("/health")
async def health_check(db: AsyncSession = Depends(get_db_session)) -> dict:
    # Verify DB connectivity
    await db.execute(sa.text("SELECT 1"))
    return {"status": "ok", "version": settings.app_version}

@health_router.get("/readiness")
async def readiness_check(
    db: AsyncSession = Depends(get_db_session),
) -> dict:
    checks = {}
    # DB
    try:
        await db.execute(sa.text("SELECT 1"))
        checks["database"] = "ok"
    except Exception as e:
        checks["database"] = f"error: {e}"
    # Redis
    try:
        redis = await get_redis()
        await redis.ping()
        checks["redis"] = "ok"
    except Exception as e:
        checks["redis"] = f"error: {e}"
    
    all_ok = all(v == "ok" for v in checks.values())
    return {"status": "ok" if all_ok else "degraded", "checks": checks}
```

**Testing**:
- `Integration: GET /health returns 200 with status ok`
- `Integration: GET /readiness returns all checks ok when all services up`
- `Integration: GET /openapi.json returns valid OpenAPI 3.1 document`
- `Integration: OpenAPI spec includes correct title, version, and server URL`

---

## Phase 2: Registry Core — Models, Versions, Aliases, and Tags

### Purpose

Implement the primary registry CRUD operations: registering models, creating versions with artifact metadata, managing aliases (champion/challenger), and tagging. After this phase, users can register models, create versions, assign aliases, and query the registry through the REST API. This is the core value proposition of the product.

### Tasks

#### 2.1 — Model Version and Alias Database Models

**What**: Define SQLAlchemy models for model_versions, model_aliases, and supporting tables.

**Design**:

```python
# src/registry/db/models/model_version.py
class ModelVersion(Base, TimestampMixin):
    __tablename__ = "model_versions"
    __table_args__ = (
        sa.UniqueConstraint("model_id", "version"),
        sa.Index("idx_versions_stage", "stage"),
        sa.Index("idx_versions_framework", "framework"),
        sa.Index("idx_versions_tags", "tags", postgresql_using="gin"),
        sa.Index("idx_versions_metrics", "metrics", postgresql_using="gin"),
        sa.Index("idx_versions_framework_meta", "framework_metadata", postgresql_using="gin"),
    )

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    model_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("registered_models.id", ondelete="CASCADE"))
    version: Mapped[int]
    description: Mapped[str | None] = mapped_column(sa.Text, default=None)
    # Artifact
    artifact_uri: Mapped[str] = mapped_column(sa.Text)
    artifact_format: Mapped[str] = mapped_column(String(50), default="mlmodel")
    artifact_hash: Mapped[str | None] = mapped_column(String(128), default=None)
    artifact_size_bytes: Mapped[int | None] = mapped_column(sa.BigInteger, default=None)
    # Framework
    framework: Mapped[str] = mapped_column(String(100))
    framework_metadata: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    # Schema
    input_schema: Mapped[dict | None] = mapped_column(JSONB, default=None)
    output_schema: Mapped[dict | None] = mapped_column(JSONB, default=None)
    # Lifecycle
    stage: Mapped[str] = mapped_column(String(50), default="development")
    status: Mapped[str] = mapped_column(String(50), default="pending_review")
    # Lineage
    run_id: Mapped[str | None] = mapped_column(String(255), default=None)
    source_uri: Mapped[str | None] = mapped_column(sa.Text, default=None)
    created_by: Mapped[uuid.UUID | None] = mapped_column(sa.ForeignKey("users.id"), default=None)
    # Flexible metadata
    tags: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    metrics: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    parameters: Mapped[dict] = mapped_column(JSONB, default_factory=dict)

    model: Mapped["RegisteredModel"] = relationship(back_populates="versions", init=False)

# Valid stage transitions
STAGE_TRANSITIONS: dict[str, list[str]] = {
    "development": ["staging", "archived"],
    "staging": ["production", "development", "archived"],
    "production": ["staging", "archived"],
    "archived": ["development"],
}

# src/registry/db/models/model_alias.py
class ModelAlias(Base):
    __tablename__ = "model_aliases"

    model_id: Mapped[uuid.UUID] = mapped_column(
        sa.ForeignKey("registered_models.id", ondelete="CASCADE"), primary_key=True
    )
    alias: Mapped[str] = mapped_column(String(255), primary_key=True)
    version_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("model_versions.id"))
    updated_by: Mapped[uuid.UUID | None] = mapped_column(sa.ForeignKey("users.id"), default=None)
    updated_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )

    model: Mapped["RegisteredModel"] = relationship(back_populates="aliases", init=False)
    version: Mapped["ModelVersion"] = relationship(init=False)
```

**Testing**:
- `Unit: ModelVersion.stage defaults to "development"`
- `Unit: ModelVersion.framework_metadata stores PyTorch-specific JSONB`
- `Unit: ModelVersion.framework_metadata stores ONNX-specific JSONB`
- `Unit: STAGE_TRANSITIONS allows staging -> production`
- `Unit: STAGE_TRANSITIONS disallows development -> production`
- `Integration (testcontainers): unique constraint on (model_id, version) prevents duplicate versions`
- `Integration (testcontainers): ModelAlias composite primary key (model_id, alias) works correctly`

#### 2.2 — Registered Models API (CRUD)

**What**: Implement REST endpoints for creating, listing, getting, updating, and deleting registered models.

**Design**:

```python
# src/registry/api/schemas/models.py
from pydantic import BaseModel, Field

class RegisteredModelCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=255, pattern=r"^[a-zA-Z0-9_\-\.]+$")
    description: str | None = None
    risk_classification: str | None = Field(None, pattern=r"^(minimal|limited|high|unacceptable)$")
    ai_system_type: str | None = None
    tags: dict[str, str] = Field(default_factory=dict)
    governance: dict = Field(default_factory=dict)

class RegisteredModelUpdate(BaseModel):
    description: str | None = None
    risk_classification: str | None = None
    tags: dict[str, str] | None = None
    governance: dict | None = None

class RegisteredModelResponse(BaseModel):
    id: uuid.UUID
    organization_id: uuid.UUID
    name: str
    description: str | None
    owner_id: uuid.UUID | None
    risk_classification: str | None
    ai_system_type: str | None
    tags: dict[str, str]
    governance: dict
    latest_version: int | None
    created_at: datetime
    updated_at: datetime

class RegisteredModelList(BaseModel):
    items: list[RegisteredModelResponse]
    total: int
    page: int
    page_size: int

# src/registry/api/v1/models.py
router = APIRouter(prefix="/models", tags=["models"])

@router.post("/", status_code=201, response_model=RegisteredModelResponse)
async def create_model(body: RegisteredModelCreate, ...): ...

@router.get("/", response_model=RegisteredModelList)
async def list_models(
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    name: str | None = Query(None),
    tags: str | None = Query(None),  # JSON-encoded tag filter
    risk_classification: str | None = Query(None),
    ...
): ...

@router.get("/{model_id_or_name}", response_model=RegisteredModelResponse)
async def get_model(model_id_or_name: str, ...): ...

@router.patch("/{model_id_or_name}", response_model=RegisteredModelResponse)
async def update_model(model_id_or_name: str, body: RegisteredModelUpdate, ...): ...

@router.delete("/{model_id_or_name}", status_code=204)
async def delete_model(model_id_or_name: str, ...): ...
```

**Testing**:
- `Unit: RegisteredModelCreate validates name pattern rejects spaces and special chars`
- `Unit: RegisteredModelCreate validates risk_classification enum values`
- `Integration: POST /api/v1/models creates model and returns 201 with generated id`
- `Integration: POST /api/v1/models with duplicate name returns 409 Conflict`
- `Integration: GET /api/v1/models returns paginated list with correct total`
- `Integration: GET /api/v1/models?name=fraud returns filtered results`
- `Integration: GET /api/v1/models?tags={"team":"risk"} filters by JSONB containment`
- `Integration: GET /api/v1/models/{name} returns model by name`
- `Integration: GET /api/v1/models/{uuid} returns model by id`
- `Integration: GET /api/v1/models/nonexistent returns 404`
- `Integration: PATCH /api/v1/models/{name} updates description and tags`
- `Integration: DELETE /api/v1/models/{name} returns 204 and model is gone`
- `Integration: DELETE /api/v1/models/{name} cascades to versions and aliases`

#### 2.3 — Model Versions API (CRUD + Auto-Increment)

**What**: Implement REST endpoints for creating versions with automatic version increment, listing versions, transitioning stages, and managing version metadata.

**Design**:

```python
# src/registry/api/schemas/versions.py
class ModelVersionCreate(BaseModel):
    artifact_uri: str
    artifact_format: str = "mlmodel"
    artifact_hash: str | None = None
    artifact_size_bytes: int | None = None
    framework: str  # "pytorch", "tensorflow", "sklearn", "xgboost", "huggingface", "onnx", "custom"
    framework_metadata: dict = Field(default_factory=dict)
    input_schema: dict | None = None
    output_schema: dict | None = None
    run_id: str | None = None
    source_uri: str | None = None
    description: str | None = None
    tags: dict[str, str] = Field(default_factory=dict)
    metrics: dict[str, float] = Field(default_factory=dict)
    parameters: dict[str, str] = Field(default_factory=dict)

class StageTransitionRequest(BaseModel):
    stage: str = Field(..., pattern=r"^(development|staging|production|archived)$")
    comment: str | None = None

class ModelVersionResponse(BaseModel):
    id: uuid.UUID
    model_id: uuid.UUID
    version: int
    description: str | None
    artifact_uri: str
    artifact_format: str
    artifact_hash: str | None
    artifact_size_bytes: int | None
    framework: str
    framework_metadata: dict
    input_schema: dict | None
    output_schema: dict | None
    stage: str
    status: str
    run_id: str | None
    source_uri: str | None
    tags: dict[str, str]
    metrics: dict[str, float]
    parameters: dict[str, str]
    created_by: uuid.UUID | None
    created_at: datetime
    updated_at: datetime

# src/registry/services/version_service.py
class VersionService:
    async def create_version(
        self, model_id: uuid.UUID, data: ModelVersionCreate, user_id: uuid.UUID
    ) -> ModelVersion:
        # Auto-increment: SELECT COALESCE(MAX(version), 0) + 1 FROM model_versions WHERE model_id = ?
        next_version = await self._get_next_version(model_id)
        version = ModelVersion(
            model_id=model_id,
            version=next_version,
            created_by=user_id,
            **data.model_dump(),
        )
        self.session.add(version)
        await self.session.flush()
        return version

    async def transition_stage(
        self, version_id: uuid.UUID, to_stage: str, comment: str | None, user_id: uuid.UUID
    ) -> ModelVersion:
        version = await self._get_version(version_id)
        allowed = STAGE_TRANSITIONS.get(version.stage, [])
        if to_stage not in allowed:
            raise InvalidStageTransition(version.stage, to_stage)
        version.stage = to_stage
        # Record audit log entry
        await self.audit_service.log(
            entity_type="model_version", entity_id=version_id,
            action="stage_changed",
            changes={"stage": {"old": version.stage, "new": to_stage}},
            actor_id=user_id, metadata={"comment": comment},
        )
        return version
```

**Testing**:
- `Unit: create_version auto-increments from 0 to 1 for first version`
- `Unit: create_version auto-increments from 3 to 4 when 3 versions exist`
- `Unit: transition_stage allows staging -> production`
- `Unit: transition_stage raises InvalidStageTransition for development -> production`
- `Integration: POST /api/v1/models/{name}/versions creates version with auto-incremented number`
- `Integration: POST /api/v1/models/{name}/versions with framework_metadata stores JSONB correctly`
- `Integration: GET /api/v1/models/{name}/versions returns all versions sorted by version desc`
- `Integration: GET /api/v1/models/{name}/versions?stage=production filters by stage`
- `Integration: POST /api/v1/models/{name}/versions/{v}/stage transitions stage and records audit`
- `Integration: POST /api/v1/models/{name}/versions/{v}/stage with invalid transition returns 400`
- `Integration: concurrent version creation uses SELECT FOR UPDATE to prevent duplicate version numbers`

#### 2.4 — Model Aliases API

**What**: Implement endpoints for creating, listing, updating, and deleting named aliases that point to specific model versions.

**Design**:

```python
# src/registry/api/v1/aliases.py
router = APIRouter(prefix="/models/{model_name}/aliases", tags=["aliases"])

class AliasCreate(BaseModel):
    alias: str = Field(..., min_length=1, max_length=255, pattern=r"^[a-zA-Z0-9_\-]+$")
    version: int  # Version number (not UUID) for user-friendliness

class AliasResponse(BaseModel):
    model_id: uuid.UUID
    alias: str
    version_id: uuid.UUID
    version: int
    updated_by: uuid.UUID | None
    updated_at: datetime

@router.put("/{alias}", response_model=AliasResponse)
async def set_alias(model_name: str, alias: str, body: AliasCreate, ...):
    """Create or update an alias. Atomic swap if alias already exists."""
    ...

@router.get("/", response_model=list[AliasResponse])
async def list_aliases(model_name: str, ...): ...

@router.get("/{alias}", response_model=ModelVersionResponse)
async def get_version_by_alias(model_name: str, alias: str, ...):
    """Resolve alias to its current model version."""
    ...

@router.delete("/{alias}", status_code=204)
async def delete_alias(model_name: str, alias: str, ...): ...
```

**Testing**:
- `Unit: AliasCreate validates alias pattern rejects spaces`
- `Integration: PUT /api/v1/models/{name}/aliases/champion creates alias pointing to version 3`
- `Integration: PUT /api/v1/models/{name}/aliases/champion with different version atomically swaps`
- `Integration: GET /api/v1/models/{name}/aliases/champion resolves to the correct version`
- `Integration: GET /api/v1/models/{name}/aliases/nonexistent returns 404`
- `Integration: DELETE /api/v1/models/{name}/aliases/champion removes alias`
- `Integration: deleting a model cascades alias deletion`

#### 2.5 — Audit Log Foundation

**What**: Implement the audit_log table and service that records all state-changing operations for compliance.

**Design**:

```python
# src/registry/db/models/audit.py
class AuditLog(Base):
    __tablename__ = "audit_log"
    __table_args__ = (
        sa.Index("idx_audit_entity", "entity_type", "entity_id"),
        sa.Index("idx_audit_actor", "actor_id"),
        sa.Index("idx_audit_created", "created_at", postgresql_ops={"created_at": "DESC"}),
    )

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    entity_type: Mapped[str] = mapped_column(String(100))
    entity_id: Mapped[uuid.UUID]
    action: Mapped[str] = mapped_column(String(100))
    actor_id: Mapped[uuid.UUID | None] = mapped_column(sa.ForeignKey("users.id"), default=None)
    changes: Mapped[dict | None] = mapped_column(JSONB, default=None)
    metadata: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    created_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )

# src/registry/services/audit_service.py
class AuditService:
    async def log(
        self,
        entity_type: str,
        entity_id: uuid.UUID,
        action: str,
        actor_id: uuid.UUID | None = None,
        changes: dict | None = None,
        metadata: dict | None = None,
    ) -> AuditLog:
        entry = AuditLog(
            entity_type=entity_type,
            entity_id=entity_id,
            action=action,
            actor_id=actor_id,
            changes=changes,
            metadata=metadata or {},
        )
        self.session.add(entry)
        return entry

    async def get_history(
        self, entity_type: str, entity_id: uuid.UUID, limit: int = 50
    ) -> list[AuditLog]:
        stmt = (
            sa.select(AuditLog)
            .where(AuditLog.entity_type == entity_type, AuditLog.entity_id == entity_id)
            .order_by(AuditLog.created_at.desc())
            .limit(limit)
        )
        result = await self.session.execute(stmt)
        return list(result.scalars().all())
```

**Testing**:
- `Unit: AuditService.log creates entry with correct entity_type and action`
- `Unit: AuditService.get_history returns entries in reverse chronological order`
- `Integration: stage transition creates audit_log entry with changes JSONB showing old/new stage`
- `Integration: model creation creates audit_log entry with action "created"`
- `Integration: GET /api/v1/models/{name}/audit returns audit trail for model`

---

## Phase 3: Artifact Storage and Lineage

### Purpose

Add model artifact upload/download to S3-compatible storage with integrity verification, and implement dataset registration with lineage tracking. After this phase, users can upload actual model files, verify artifact integrity, and trace the lineage from model versions back to training datasets and code commits.

### Tasks

#### 3.1 — Artifact Storage Service (S3-Compatible)

**What**: Implement presigned URL generation for artifact upload/download, hash verification, and size tracking.

**Design**:

```python
# src/registry/integrations/storage/base.py
from abc import ABC, abstractmethod

class ArtifactStore(ABC):
    @abstractmethod
    async def generate_upload_url(self, key: str, content_type: str, expires_in: int = 3600) -> str: ...

    @abstractmethod
    async def generate_download_url(self, key: str, expires_in: int = 3600) -> str: ...

    @abstractmethod
    async def get_object_metadata(self, key: str) -> ObjectMetadata: ...

    @abstractmethod
    async def delete_object(self, key: str) -> None: ...

    @abstractmethod
    async def compute_hash(self, key: str) -> str: ...

class ObjectMetadata(BaseModel):
    key: str
    size_bytes: int
    content_type: str
    etag: str
    last_modified: datetime

# src/registry/integrations/storage/s3.py
class S3ArtifactStore(ArtifactStore):
    def __init__(self, settings: Settings):
        self.client = boto3.client(
            "s3",
            endpoint_url=settings.s3_endpoint_url,
            aws_access_key_id=settings.s3_access_key,
            aws_secret_access_key=settings.s3_secret_key,
            region_name=settings.s3_region,
        )
        self.bucket = settings.s3_bucket

    async def generate_upload_url(self, key: str, content_type: str, expires_in: int = 3600) -> str:
        return self.client.generate_presigned_url(
            "put_object",
            Params={"Bucket": self.bucket, "Key": key, "ContentType": content_type},
            ExpiresIn=expires_in,
        )

# src/registry/api/v1/versions.py (additions)
@router.post("/{model_name}/versions/{version}/artifact/upload-url")
async def get_upload_url(model_name: str, version: int, ...) -> dict:
    """Generate a presigned URL for uploading the model artifact."""
    artifact_key = f"models/{org_id}/{model_name}/v{version}/model.{format}"
    url = await artifact_store.generate_upload_url(artifact_key, content_type)
    return {"upload_url": url, "artifact_key": artifact_key, "expires_in": 3600}

@router.post("/{model_name}/versions/{version}/artifact/finalize")
async def finalize_artifact(model_name: str, version: int, ...) -> dict:
    """After upload, verify hash and update version metadata."""
    meta = await artifact_store.get_object_metadata(artifact_key)
    hash_val = await artifact_store.compute_hash(artifact_key)
    # Update model_version with size and hash
    ...
```

**Testing**:
- `Unit (mocked S3): generate_upload_url returns a presigned URL with correct bucket and key`
- `Unit (mocked S3): generate_download_url returns a presigned URL`
- `Unit (mocked S3): compute_hash returns SHA-256 of object contents`
- `Integration (MinIO): upload a 1KB file via presigned URL, finalize, verify hash matches`
- `Integration (MinIO): upload a 100MB file via presigned URL (chunked), verify size_bytes`
- `Integration: GET /api/v1/models/{name}/versions/{v}/artifact/download-url returns working presigned URL`
- `Integration: finalize_artifact with corrupted upload returns 422 (hash mismatch)`

#### 3.2 — Dataset Registration and Versioning

**What**: Implement dataset and dataset_versions tables with CRUD endpoints.

**Design**:

```python
# src/registry/db/models/dataset.py
class Dataset(Base, TimestampMixin):
    __tablename__ = "datasets"
    __table_args__ = (sa.UniqueConstraint("organization_id", "name"),)

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    organization_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("organizations.id"))
    name: Mapped[str] = mapped_column(String(255))
    source_type: Mapped[str | None] = mapped_column(String(50), default=None)
    source_uri: Mapped[str | None] = mapped_column(sa.Text, default=None)
    metadata: Mapped[dict] = mapped_column(JSONB, default_factory=dict)

class DatasetVersion(Base):
    __tablename__ = "dataset_versions"
    __table_args__ = (sa.UniqueConstraint("dataset_id", "version"),)

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    dataset_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("datasets.id", ondelete="CASCADE"))
    version: Mapped[int]
    uri: Mapped[str] = mapped_column(sa.Text)
    hash: Mapped[str | None] = mapped_column(String(128), default=None)
    metadata: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    created_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )

# API endpoints: /api/v1/datasets and /api/v1/datasets/{name}/versions
# Follow same patterns as models and model_versions
```

**Testing**:
- `Integration: POST /api/v1/datasets creates dataset with auto-generated id`
- `Integration: POST /api/v1/datasets/{name}/versions auto-increments version`
- `Integration: GET /api/v1/datasets/{name}/versions returns all versions`
- `Integration: dataset metadata JSONB stores num_rows, num_features, schema correctly`

#### 3.3 — Lineage Links

**What**: Implement polymorphic lineage tracking between models, datasets, code commits, and experiment runs.

**Design**:

```python
# src/registry/db/models/lineage.py
class LineageLink(Base):
    __tablename__ = "lineage_links"
    __table_args__ = (
        sa.Index("idx_lineage_source", "source_type", "source_id"),
        sa.Index("idx_lineage_target", "target_type", "target_id"),
    )

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    source_type: Mapped[str] = mapped_column(String(50))  # "dataset_version", "model_version", "code_commit"
    source_id: Mapped[uuid.UUID]
    target_type: Mapped[str] = mapped_column(String(50))  # "model_version", "serving_endpoint"
    target_id: Mapped[uuid.UUID]
    relationship: Mapped[str] = mapped_column(String(50))  # "trained_on", "evaluated_on", "derived_from"
    metadata: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    created_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )

# src/registry/services/lineage_service.py
class LineageService:
    async def add_link(
        self, source_type: str, source_id: uuid.UUID,
        target_type: str, target_id: uuid.UUID,
        relationship: str, metadata: dict | None = None,
    ) -> LineageLink: ...

    async def get_upstream(
        self, entity_type: str, entity_id: uuid.UUID, max_depth: int = 5
    ) -> list[LineageNode]:
        """Recursive upstream traversal using CTE."""
        ...

    async def get_downstream(
        self, entity_type: str, entity_id: uuid.UUID, max_depth: int = 5
    ) -> list[LineageNode]:
        """Recursive downstream traversal using CTE."""
        ...

# src/registry/api/v1/lineage.py
@router.post("/lineage", status_code=201)
async def create_lineage_link(body: LineageLinkCreate, ...): ...

@router.get("/lineage/upstream/{entity_type}/{entity_id}")
async def get_upstream_lineage(entity_type: str, entity_id: uuid.UUID, depth: int = 5, ...): ...

@router.get("/lineage/downstream/{entity_type}/{entity_id}")
async def get_downstream_lineage(entity_type: str, entity_id: uuid.UUID, depth: int = 5, ...): ...
```

**Testing**:
- `Unit: LineageService.add_link creates link with correct source and target`
- `Integration: create link from dataset_version to model_version with relationship "trained_on"`
- `Integration: get_upstream from model_version returns dataset_version at depth 1`
- `Integration: get_upstream with depth 3 traverses dataset -> model -> endpoint chain`
- `Integration: get_downstream from dataset_version returns all model_versions trained on it`
- `Integration: lineage API returns correct graph structure for multi-hop chain`
- `Integration: circular lineage reference handled without infinite recursion (max_depth limit)`

---

## Phase 4: Authentication, Authorization, and Multi-Tenancy

### Purpose

Add JWT-based authentication, API key support for service principals, OIDC integration for enterprise SSO, and role-based access control scoped to organizations. After this phase, the registry is secure and multi-tenant.

### Tasks

#### 4.1 — JWT Authentication and API Key Support

**What**: Implement JWT token issuance, validation, and API key authentication for service principals.

**Design**:

```python
# src/registry/auth/jwt.py
from jose import jwt, JWTError

class JWTService:
    def __init__(self, settings: Settings):
        self.secret_key = settings.jwt_secret_key
        self.algorithm = settings.jwt_algorithm
        self.expiry_minutes = settings.jwt_expiry_minutes

    def create_token(self, user_id: uuid.UUID, org_id: uuid.UUID, role: str) -> str:
        payload = {
            "sub": str(user_id),
            "org": str(org_id),
            "role": role,
            "exp": datetime.utcnow() + timedelta(minutes=self.expiry_minutes),
            "iat": datetime.utcnow(),
        }
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)

    def decode_token(self, token: str) -> TokenPayload:
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            return TokenPayload(**payload)
        except JWTError as e:
            raise AuthenticationError(f"Invalid token: {e}")

# src/registry/auth/api_key.py
class APIKeyService:
    async def create_key(self, user_id: uuid.UUID, name: str, scopes: list[str]) -> tuple[str, APIKey]:
        """Returns (raw_key, api_key_record). Raw key shown once."""
        raw_key = secrets.token_urlsafe(32)
        key_hash = bcrypt.hashpw(raw_key.encode(), bcrypt.gensalt()).decode()
        api_key = APIKey(user_id=user_id, key_hash=key_hash, name=name, scopes=scopes)
        ...
        return f"mreg_{raw_key}", api_key

    async def validate_key(self, raw_key: str) -> User:
        # Strip prefix, hash, look up
        ...

# src/registry/api/deps.py
async def get_current_user(
    authorization: str | None = Header(None),
    x_api_key: str | None = Header(None, alias="X-API-Key"),
    db: AsyncSession = Depends(get_db_session),
) -> User:
    if x_api_key:
        return await api_key_service.validate_key(x_api_key)
    if authorization and authorization.startswith("Bearer "):
        token = authorization[7:]
        payload = jwt_service.decode_token(token)
        return await get_user_by_id(db, payload.sub)
    raise AuthenticationError("No valid credentials provided")
```

**Testing**:
- `Unit: JWTService.create_token includes sub, org, role, exp claims`
- `Unit: JWTService.decode_token with valid token returns correct payload`
- `Unit: JWTService.decode_token with expired token raises AuthenticationError`
- `Unit: JWTService.decode_token with tampered token raises AuthenticationError`
- `Unit: APIKeyService.create_key returns key with mreg_ prefix`
- `Unit: APIKeyService.validate_key with correct key returns user`
- `Unit: APIKeyService.validate_key with wrong key raises AuthenticationError`
- `Integration: API request with valid Bearer token succeeds`
- `Integration: API request with valid X-API-Key header succeeds`
- `Integration: API request with no auth returns 401`
- `Integration: API request with expired token returns 401`

#### 4.2 — OIDC Integration for Enterprise SSO

**What**: Implement OIDC discovery, authorization code flow, and user provisioning from OIDC claims.

**Design**:

```python
# src/registry/auth/oidc.py
from authlib.integrations.httpx_client import AsyncOAuth2Client

class OIDCProvider:
    def __init__(self, settings: Settings):
        self.discovery_url = settings.oidc_discovery_url
        self.client_id = settings.oidc_client_id
        self.client_secret = settings.oidc_client_secret
        self._config: dict | None = None

    async def get_authorization_url(self, redirect_uri: str, state: str) -> str: ...

    async def exchange_code(self, code: str, redirect_uri: str) -> OIDCTokens: ...

    async def get_userinfo(self, access_token: str) -> OIDCUserInfo: ...

class OIDCUserInfo(BaseModel):
    sub: str
    email: str
    name: str | None
    groups: list[str] = []

# API routes: /api/v1/auth/oidc/authorize, /api/v1/auth/oidc/callback
```

**Testing**:
- `Unit (mocked): OIDCProvider.get_authorization_url constructs correct URL with client_id and scope`
- `Unit (mocked): OIDCProvider.exchange_code returns tokens from token endpoint`
- `Unit (mocked): OIDCProvider.get_userinfo returns user claims`
- `Integration (mocked): full OIDC flow: authorize -> callback -> JWT issued -> user created in DB`
- `Integration: OIDC callback with existing user updates display_name from claims`

#### 4.3 — Role-Based Access Control (RBAC)

**What**: Implement permission checking based on user roles (admin, member, viewer) scoped to organizations.

**Design**:

```python
# src/registry/auth/rbac.py
from enum import StrEnum
from functools import wraps

class Permission(StrEnum):
    MODEL_READ = "model:read"
    MODEL_WRITE = "model:write"
    MODEL_DELETE = "model:delete"
    MODEL_APPROVE = "model:approve"      # Stage transition to production
    SERVING_MANAGE = "serving:manage"
    COMPLIANCE_WRITE = "compliance:write"
    ADMIN = "admin"

ROLE_PERMISSIONS: dict[str, set[Permission]] = {
    "admin": {p for p in Permission},
    "member": {
        Permission.MODEL_READ, Permission.MODEL_WRITE,
        Permission.MODEL_APPROVE, Permission.SERVING_MANAGE,
    },
    "viewer": {Permission.MODEL_READ},
}

def require_permission(permission: Permission):
    """FastAPI dependency that checks user has the required permission."""
    async def check(user: User = Depends(get_current_user)):
        user_perms = ROLE_PERMISSIONS.get(user.role, set())
        if permission not in user_perms:
            raise HTTPException(403, f"Requires permission: {permission}")
        return user
    return Depends(check)
```

**Testing**:
- `Unit: admin role has all permissions`
- `Unit: viewer role has only MODEL_READ`
- `Unit: member role can MODEL_WRITE but not ADMIN`
- `Integration: viewer user GET /models succeeds`
- `Integration: viewer user POST /models returns 403`
- `Integration: member user POST /models succeeds`
- `Integration: member user DELETE /models returns 403`
- `Integration: admin user DELETE /models succeeds`
- `Integration: user from org A cannot access models from org B`

---

## Phase 5: Serving Orchestration and Traffic Management

### Purpose

Implement the serving endpoint management layer that orchestrates model deployment to KServe and Seldon Core, manages traffic splitting for A/B experiments, and supports canary rollouts. The registry does not serve models directly -- it configures and monitors serving infrastructure.

### Tasks

#### 5.1 — Serving Endpoint and Variant Database Models

**What**: Define database models for serving_endpoints, serving_variants, and ab_experiments.

**Design**:

```python
# src/registry/db/models/serving.py
class ServingEndpoint(Base, TimestampMixin):
    __tablename__ = "serving_endpoints"
    __table_args__ = (sa.UniqueConstraint("organization_id", "name"),)

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    organization_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("organizations.id"))
    name: Mapped[str] = mapped_column(String(255))
    protocol: Mapped[str] = mapped_column(String(50), default="v2")  # Open Inference Protocol
    status: Mapped[str] = mapped_column(String(50), default="creating")
    endpoint_url: Mapped[str | None] = mapped_column(sa.Text, default=None)
    config: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    traffic_split: Mapped[dict] = mapped_column(JSONB, default_factory=dict)

class ServingVariant(Base, TimestampMixin):
    __tablename__ = "serving_variants"
    __table_args__ = (sa.UniqueConstraint("endpoint_id", "variant_name"),)

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    endpoint_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("serving_endpoints.id", ondelete="CASCADE"))
    version_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("model_versions.id"))
    variant_name: Mapped[str] = mapped_column(String(255))
    traffic_percentage: Mapped[float] = mapped_column(sa.Numeric(5, 2), default=0)
    is_shadow: Mapped[bool] = mapped_column(default=False)
    status: Mapped[str] = mapped_column(String(50), default="creating")
    config: Mapped[dict] = mapped_column(JSONB, default_factory=dict)

class ABExperiment(Base):
    __tablename__ = "ab_experiments"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    endpoint_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("serving_endpoints.id", ondelete="CASCADE"))
    name: Mapped[str] = mapped_column(String(255))
    status: Mapped[str] = mapped_column(String(50), default="draft")
    config: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    results: Mapped[dict | None] = mapped_column(JSONB, default=None)
    started_at: Mapped[datetime | None] = mapped_column(sa.TIMESTAMP(timezone=True), default=None)
    concluded_at: Mapped[datetime | None] = mapped_column(sa.TIMESTAMP(timezone=True), default=None)
    created_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )
```

**Testing**:
- `Unit: ServingEndpoint status defaults to "creating"`
- `Unit: ServingVariant traffic_percentage stores decimal precision`
- `Integration (testcontainers): create endpoint with two variants, verify traffic_split JSONB`
- `Integration (testcontainers): unique constraint on (endpoint_id, variant_name) prevents duplicates`

#### 5.2 — Serving API and Traffic Management

**What**: Implement REST endpoints for creating endpoints, deploying variants, updating traffic splits, and managing A/B experiments.

**Design**:

```python
# src/registry/api/schemas/serving.py
class EndpointCreate(BaseModel):
    name: str = Field(..., pattern=r"^[a-zA-Z0-9_\-]+$")
    protocol: str = "v2"
    config: dict = Field(default_factory=dict)

class VariantDeploy(BaseModel):
    variant_name: str
    model_name: str
    version: int  # or alias name
    traffic_percentage: float = Field(ge=0, le=100)
    is_shadow: bool = False
    runtime: str = "kserve"  # "kserve", "seldon", "triton", "bentoml"
    config: dict = Field(default_factory=dict)

class TrafficUpdate(BaseModel):
    traffic_split: dict[str, float]  # {"control": 80.0, "canary": 20.0}

class ExperimentCreate(BaseModel):
    name: str
    control_variant: str
    treatment_variants: list[str]
    primary_metric: str
    significance_level: float = 0.05
    minimum_sample_size: int | None = None
    guardrail_metrics: list[str] = Field(default_factory=list)

# src/registry/services/traffic_service.py
class TrafficService:
    async def update_traffic_split(
        self, endpoint_id: uuid.UUID, split: dict[str, float]
    ) -> ServingEndpoint:
        # Validate percentages sum to 100 (excluding shadow variants)
        non_shadow = {k: v for k, v in split.items()
                      if not await self._is_shadow(endpoint_id, k)}
        if abs(sum(non_shadow.values()) - 100.0) > 0.01:
            raise ValidationError("Non-shadow traffic must sum to 100%")
        # Update each variant's traffic_percentage
        for variant_name, pct in split.items():
            await self._update_variant_traffic(endpoint_id, variant_name, pct)
        # Update denormalised traffic_split on endpoint
        ...

# src/registry/api/v1/serving.py
@router.post("/endpoints", status_code=201)
async def create_endpoint(body: EndpointCreate, ...): ...

@router.post("/endpoints/{name}/variants")
async def deploy_variant(name: str, body: VariantDeploy, ...): ...

@router.put("/endpoints/{name}/traffic")
async def update_traffic(name: str, body: TrafficUpdate, ...): ...

@router.post("/endpoints/{name}/experiments")
async def create_experiment(name: str, body: ExperimentCreate, ...): ...

@router.post("/endpoints/{name}/experiments/{exp_id}/start")
async def start_experiment(name: str, exp_id: uuid.UUID, ...): ...

@router.post("/endpoints/{name}/experiments/{exp_id}/conclude")
async def conclude_experiment(name: str, exp_id: uuid.UUID, ...): ...
```

**Testing**:
- `Unit: TrafficService validates traffic sum equals 100%`
- `Unit: TrafficService allows shadow variants to exceed 100% total`
- `Unit: TrafficService rejects negative percentages`
- `Integration: POST /endpoints creates endpoint with status "creating"`
- `Integration: POST /endpoints/{name}/variants deploys variant and updates traffic_split`
- `Integration: PUT /endpoints/{name}/traffic with 80/20 split updates both variants`
- `Integration: PUT /endpoints/{name}/traffic with 60/30 split (sum 90) returns 400`
- `Integration: POST /experiments creates draft experiment with control and treatment`
- `Integration: start experiment transitions status to "running" and records started_at`
- `Integration: conclude experiment transitions to "concluded" and stores results JSONB`

#### 5.3 — KServe Integration Client

**What**: Implement the Kubernetes client that creates and manages KServe InferenceService CRDs.

**Design**:

```python
# src/registry/integrations/kserve/client.py
from kubernetes_asyncio import client, config

class KServeClient:
    def __init__(self, namespace: str = "model-serving"):
        self.namespace = namespace
        self.api: client.CustomObjectsApi | None = None

    async def connect(self):
        await config.load_incluster_config()  # or load_kube_config for local
        self.api = client.CustomObjectsApi()

    async def create_inference_service(
        self, name: str, model_uri: str, framework: str,
        min_replicas: int = 0, max_replicas: int = 10,
    ) -> dict:
        body = {
            "apiVersion": "serving.kserve.io/v1beta1",
            "kind": "InferenceService",
            "metadata": {"name": name, "namespace": self.namespace},
            "spec": {
                "predictor": {
                    framework: {"storageUri": model_uri},
                    "minReplicas": min_replicas,
                    "maxReplicas": max_replicas,
                }
            },
        }
        return await self.api.create_namespaced_custom_object(
            group="serving.kserve.io", version="v1beta1",
            namespace=self.namespace, plural="inferenceservices", body=body,
        )

    async def update_canary_traffic(self, name: str, canary_percent: int) -> dict:
        patch = {
            "spec": {
                "predictor": {"canaryTrafficPercent": canary_percent}
            }
        }
        return await self.api.patch_namespaced_custom_object(
            group="serving.kserve.io", version="v1beta1",
            namespace=self.namespace, plural="inferenceservices",
            name=name, body=patch,
        )

    async def get_inference_service_status(self, name: str) -> dict: ...

    async def delete_inference_service(self, name: str) -> None: ...
```

**Testing**:
- `Unit (mocked K8s): create_inference_service generates correct CRD YAML`
- `Unit (mocked K8s): update_canary_traffic patches canaryTrafficPercent`
- `Unit (mocked K8s): delete_inference_service calls correct API`
- `Integration (mocked K8s): full deploy flow: create endpoint -> deploy variant -> verify CRD created`
- `Integration (mocked K8s): update traffic triggers CRD patch with new traffic split`

---

## Phase 6: Monitoring, Drift Detection, and Rollback

### Purpose

Implement serving metrics collection, drift detection monitors with configurable thresholds, automated alerts, and rollback capabilities. This phase delivers the intelligent rollback feature that is a key differentiator -- compound signal detection (prediction drift + business KPI degradation) triggering automatic rollback.

### Tasks

#### 6.1 — Serving Metrics Collection

**What**: Implement the serving_metrics table and ingestion endpoint for time-bucketed metrics from serving infrastructure.

**Design**:

```python
# src/registry/db/models/monitoring.py
class ServingMetric(Base):
    __tablename__ = "serving_metrics"

    endpoint_id: Mapped[uuid.UUID] = mapped_column(primary_key=True)
    variant_name: Mapped[str] = mapped_column(String(255), primary_key=True)
    bucket: Mapped[datetime] = mapped_column(sa.TIMESTAMP(timezone=True), primary_key=True)
    metrics: Mapped[dict] = mapped_column(JSONB)  # request_count, error_count, latency_p50/p95/p99, etc.

# src/registry/api/v1/monitoring.py
class MetricsBatch(BaseModel):
    endpoint_name: str
    variant_name: str
    metrics: list[MetricDatapoint]

class MetricDatapoint(BaseModel):
    timestamp: datetime
    request_count: int
    error_count: int = 0
    latency_p50_ms: float | None = None
    latency_p95_ms: float | None = None
    latency_p99_ms: float | None = None
    prediction_mean: float | None = None
    prediction_stddev: float | None = None
    custom: dict = Field(default_factory=dict)

@router.post("/metrics/ingest", status_code=202)
async def ingest_metrics(body: MetricsBatch, ...): ...

@router.get("/endpoints/{name}/metrics")
async def get_endpoint_metrics(
    name: str,
    variant: str | None = None,
    start: datetime = Query(...),
    end: datetime = Query(...),
    ...
) -> list[dict]: ...
```

**Testing**:
- `Integration: POST /metrics/ingest stores metrics in correct time buckets`
- `Integration: GET /endpoints/{name}/metrics returns time-range filtered metrics`
- `Integration: GET /endpoints/{name}/metrics?variant=control filters by variant`
- `Integration: metrics with overlapping timestamps upsert correctly (ON CONFLICT UPDATE)`
- `Unit: MetricDatapoint validates positive request_count`

#### 6.2 — Drift Detection Engine

**What**: Implement configurable drift monitors that periodically compare current prediction distributions against reference distributions.

**Design**:

```python
# src/registry/db/models/monitoring.py (additions)
class DriftMonitor(Base):
    __tablename__ = "drift_monitors"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    endpoint_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("serving_endpoints.id", ondelete="CASCADE"))
    monitor_type: Mapped[str] = mapped_column(String(50))  # "data_drift", "prediction_drift", "performance_drift"
    status: Mapped[str] = mapped_column(String(50), default="active")
    config: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    last_check_at: Mapped[datetime | None] = mapped_column(sa.TIMESTAMP(timezone=True), default=None)
    last_result: Mapped[dict | None] = mapped_column(JSONB, default=None)
    created_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )

class DriftAlert(Base):
    __tablename__ = "drift_alerts"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    monitor_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("drift_monitors.id", ondelete="CASCADE"))
    severity: Mapped[str] = mapped_column(String(50))
    drift_score: Mapped[float] = mapped_column(sa.Numeric(8, 4))
    details: Mapped[dict] = mapped_column(JSONB)
    acknowledged_by: Mapped[uuid.UUID | None] = mapped_column(sa.ForeignKey("users.id"), default=None)
    acknowledged_at: Mapped[datetime | None] = mapped_column(sa.TIMESTAMP(timezone=True), default=None)
    created_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )

# src/registry/services/drift_service.py
class DriftService:
    def compute_psi(self, reference: np.ndarray, current: np.ndarray, bins: int = 10) -> float:
        """Population Stability Index for detecting distribution shift."""
        ref_hist, bin_edges = np.histogram(reference, bins=bins)
        cur_hist, _ = np.histogram(current, bins=bin_edges)
        # Avoid division by zero
        ref_pct = np.clip(ref_hist / ref_hist.sum(), 1e-6, None)
        cur_pct = np.clip(cur_hist / cur_hist.sum(), 1e-6, None)
        psi = np.sum((cur_pct - ref_pct) * np.log(cur_pct / ref_pct))
        return float(psi)

    async def check_drift(self, monitor_id: uuid.UUID) -> DriftCheckResult:
        monitor = await self._get_monitor(monitor_id)
        config = monitor.config
        # Fetch reference and current prediction distributions from serving_metrics
        reference = await self._get_reference_distribution(monitor)
        current = await self._get_current_distribution(monitor)
        drift_score = self.compute_psi(reference, current)

        severity = None
        if drift_score >= config.get("critical_threshold", 0.35):
            severity = "critical"
        elif drift_score >= config.get("alert_threshold", 0.20):
            severity = "warning"

        if severity:
            alert = DriftAlert(
                monitor_id=monitor_id,
                severity=severity,
                drift_score=drift_score,
                details={"psi": drift_score, "threshold": config.get("alert_threshold")},
            )
            self.session.add(alert)
            # Trigger webhook if configured
            await self.webhook_service.dispatch("drift.detected", {...})

        return DriftCheckResult(score=drift_score, severity=severity)
```

**Testing**:
- `Unit: compute_psi returns 0.0 for identical distributions`
- `Unit: compute_psi returns >0.2 for significantly shifted distributions`
- `Unit: check_drift with PSI 0.25 creates warning alert`
- `Unit: check_drift with PSI 0.40 creates critical alert`
- `Unit: check_drift with PSI 0.10 creates no alert`
- `Integration: POST /endpoints/{name}/monitors creates drift monitor with config`
- `Integration: periodic drift check task runs and stores last_check_at and last_result`
- `Integration: GET /endpoints/{name}/monitors/{id}/alerts returns alerts sorted by created_at desc`
- `Integration: POST /alerts/{id}/acknowledge updates acknowledged_by and acknowledged_at`

#### 6.3 — Automated Rollback on Compound Degradation

**What**: Implement rollback triggers that activate when both prediction drift AND business KPI degradation exceed thresholds simultaneously.

**Design**:

```python
# src/registry/services/drift_service.py (additions)
class CompoundDegradationChecker:
    """
    Checks for compound degradation: prediction drift + business metric drop.
    This is the intelligent rollback feature differentiating us from all competitors.
    """
    async def evaluate(
        self, endpoint_id: uuid.UUID, monitor_config: dict
    ) -> RollbackDecision:
        rollback_config = monitor_config.get("rollback_conditions", {})
        if not rollback_config.get("compound"):
            return RollbackDecision(should_rollback=False)

        # Check drift threshold
        drift_score = await self._get_latest_drift_score(endpoint_id)
        drift_threshold = rollback_config.get("drift_threshold", 0.35)
        drift_exceeded = drift_score >= drift_threshold

        # Check business metric
        business_metric = rollback_config.get("business_metric")
        metric_drop_pct = rollback_config.get("business_metric_drop_pct", 5.0)
        window_minutes = rollback_config.get("evaluation_window_minutes", 120)

        current_metric = await self._get_current_metric(endpoint_id, business_metric, window_minutes)
        baseline_metric = await self._get_baseline_metric(endpoint_id, business_metric)
        metric_drop = ((baseline_metric - current_metric) / baseline_metric) * 100
        metric_exceeded = metric_drop >= metric_drop_pct

        if drift_exceeded and metric_exceeded:
            return RollbackDecision(
                should_rollback=True,
                reason=f"Compound degradation: PSI {drift_score:.3f} + "
                       f"{business_metric} drop {metric_drop:.1f}% over {window_minutes}min",
                rollback_to_version_id=await self._get_previous_champion(endpoint_id),
            )

        return RollbackDecision(should_rollback=False)

# src/registry/workers/rollback_tasks.py
@celery_app.task
async def execute_rollback(endpoint_id: str, to_version_id: str, reason: str):
    """Atomic rollback: swap champion alias and update serving traffic."""
    # 1. Update model alias to point to previous version
    # 2. Update serving variant traffic to 100% on rollback version
    # 3. Create audit log entry
    # 4. Dispatch webhook: "endpoint.rollback_triggered"
    ...
```

**Testing**:
- `Unit: CompoundDegradationChecker returns should_rollback=False when only drift exceeds`
- `Unit: CompoundDegradationChecker returns should_rollback=False when only metric drops`
- `Unit: CompoundDegradationChecker returns should_rollback=True when both exceed thresholds`
- `Unit: CompoundDegradationChecker respects evaluation_window_minutes`
- `Integration: rollback task swaps champion alias to previous version`
- `Integration: rollback task updates traffic to 100% on rollback version`
- `Integration: rollback creates audit_log entry with reason`
- `Integration: rollback dispatches webhook event`
- `Integration: manual rollback via POST /endpoints/{name}/rollback works`

#### 6.4 — Periodic Drift Check Worker

**What**: Implement the Celery beat schedule and worker task that periodically runs drift checks for all active monitors.

**Design**:

```python
# src/registry/workers/drift_tasks.py
@celery_app.task(bind=True, max_retries=3)
def run_drift_check(self, monitor_id: str):
    """Run drift detection for a single monitor."""
    async def _run():
        async with async_session_factory() as session:
            drift_service = DriftService(session)
            result = await drift_service.check_drift(uuid.UUID(monitor_id))
            if result.severity == "critical":
                # Evaluate compound degradation for potential rollback
                compound_checker = CompoundDegradationChecker(session)
                decision = await compound_checker.evaluate(...)
                if decision.should_rollback:
                    execute_rollback.delay(...)
    asyncio.run(_run())

@celery_app.task
def schedule_drift_checks():
    """Beat task: find all active monitors and enqueue individual checks."""
    async def _schedule():
        async with async_session_factory() as session:
            monitors = await session.execute(
                sa.select(DriftMonitor).where(DriftMonitor.status == "active")
            )
            for monitor in monitors.scalars():
                run_drift_check.delay(str(monitor.id))
    asyncio.run(_schedule())

# celery_app.conf.beat_schedule
beat_schedule = {
    "drift-checks": {
        "task": "registry.workers.drift_tasks.schedule_drift_checks",
        "schedule": crontab(minute="*/5"),  # Every 5 minutes
    },
}
```

**Testing**:
- `Unit: schedule_drift_checks enqueues tasks for all active monitors`
- `Unit: schedule_drift_checks skips paused monitors`
- `Integration: run_drift_check updates monitor.last_check_at`
- `Integration: run_drift_check with critical drift score triggers compound evaluation`
- `Integration: beat schedule runs at configured interval`

---

## Phase 7: Webhooks and Event System

### Purpose

Implement webhook registration, event dispatching with reliable delivery (retry, exponential backoff), and HMAC signature verification. After this phase, external systems (CI/CD pipelines, Slack, PagerDuty) can subscribe to registry events.

### Tasks

#### 7.1 — Webhook Registration and Management

**What**: Implement webhook CRUD endpoints and HMAC secret management.

**Design**:

```python
# src/registry/db/models/webhook.py
class Webhook(Base):
    __tablename__ = "webhooks"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    organization_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("organizations.id"))
    name: Mapped[str] = mapped_column(String(255))
    url: Mapped[str] = mapped_column(sa.Text)
    secret_hash: Mapped[str | None] = mapped_column(String(128), default=None)
    event_types: Mapped[list[str]] = mapped_column(sa.ARRAY(sa.Text))
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )

# Supported event types
EVENT_TYPES = [
    "model.registered", "model.updated", "model.deleted",
    "version.created", "version.stage_changed",
    "alias.updated", "alias.deleted",
    "endpoint.created", "endpoint.deployed", "endpoint.traffic_updated",
    "endpoint.rollback_triggered",
    "drift.detected", "drift.critical",
    "compliance.document_submitted", "compliance.status_changed",
]

# src/registry/api/v1/webhooks.py
class WebhookCreate(BaseModel):
    name: str
    url: HttpUrl
    event_types: list[str]
    secret: str | None = None  # If provided, HMAC signing enabled

@router.post("/webhooks", status_code=201)
async def create_webhook(body: WebhookCreate, ...): ...

@router.get("/webhooks")
async def list_webhooks(...): ...

@router.delete("/webhooks/{webhook_id}", status_code=204)
async def delete_webhook(webhook_id: uuid.UUID, ...): ...

@router.post("/webhooks/{webhook_id}/test")
async def test_webhook(webhook_id: uuid.UUID, ...):
    """Send a test ping event to verify webhook connectivity."""
    ...
```

**Testing**:
- `Unit: WebhookCreate validates url is a valid HTTP URL`
- `Unit: WebhookCreate validates event_types against supported list`
- `Integration: POST /webhooks creates webhook with hashed secret`
- `Integration: POST /webhooks/{id}/test delivers test ping and returns response status`
- `Integration: GET /webhooks returns only webhooks for the current organization`
- `Integration: DELETE /webhooks/{id} soft-deactivates webhook`

#### 7.2 — Event Dispatching with Reliable Delivery

**What**: Implement the webhook delivery worker with retry logic, exponential backoff, and HMAC signature generation.

**Design**:

```python
# src/registry/services/webhook_service.py
class WebhookService:
    async def dispatch(self, event_type: str, payload: dict) -> None:
        """Find all active webhooks subscribed to this event type and enqueue delivery."""
        webhooks = await self._get_subscribed_webhooks(event_type)
        for webhook in webhooks:
            deliver_webhook.delay(str(webhook.id), event_type, payload)

# src/registry/workers/webhook_tasks.py
@celery_app.task(bind=True, max_retries=5, default_retry_delay=60)
def deliver_webhook(self, webhook_id: str, event_type: str, payload: dict):
    webhook = get_webhook(webhook_id)
    headers = {
        "Content-Type": "application/json",
        "X-Registry-Event": event_type,
        "X-Registry-Delivery": str(uuid.uuid4()),
        "X-Registry-Timestamp": datetime.utcnow().isoformat(),
    }
    body = json.dumps(payload)

    # HMAC signature
    if webhook.secret_hash:
        signature = hmac.new(
            webhook.secret.encode(), body.encode(), hashlib.sha256
        ).hexdigest()
        headers["X-Registry-Signature"] = f"sha256={signature}"

    try:
        response = httpx.post(webhook.url, content=body, headers=headers, timeout=10)
        record_delivery(webhook_id, event_type, payload, response.status_code, response.text)
        if response.status_code >= 500:
            raise self.retry(exc=Exception(f"Server error: {response.status_code}"))
    except httpx.TimeoutException:
        raise self.retry(exc=Exception("Webhook delivery timeout"))
```

**Testing**:
- `Unit: HMAC signature matches expected sha256 hash`
- `Unit: dispatch enqueues tasks for all matching webhooks`
- `Unit: dispatch skips inactive webhooks`
- `Unit: deliver_webhook retries on 500 response`
- `Unit: deliver_webhook retries on timeout`
- `Unit: deliver_webhook does not retry on 4xx response`
- `Integration: model creation dispatches "model.registered" webhook`
- `Integration: stage transition dispatches "version.stage_changed" webhook`
- `Integration: webhook delivery records are stored with response status`
- `Integration: GET /webhooks/{id}/deliveries returns delivery history`

---

## Phase 8: EU AI Act Compliance and Governance

### Purpose

Implement EU AI Act Annex IV documentation management, compliance checklists, and auto-generation of compliance documents from registry metadata. This is a key differentiator -- no other open-source registry provides structured EU AI Act compliance support. The August 2, 2026 enforcement deadline makes this feature urgent.

### Tasks

#### 8.1 — Compliance Records Database and API

**What**: Implement the compliance_records table and CRUD endpoints for managing per-model regulatory documentation.

**Design**:

```python
# src/registry/db/models/compliance.py
class ComplianceRecord(Base, TimestampMixin):
    __tablename__ = "compliance_records"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    model_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("registered_models.id", ondelete="CASCADE"))
    version_id: Mapped[uuid.UUID | None] = mapped_column(sa.ForeignKey("model_versions.id"), default=None)
    framework: Mapped[str] = mapped_column(String(100))  # "eu_ai_act", "nist_ai_rmf", "iso_42001"
    documentation: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    overall_status: Mapped[str] = mapped_column(String(50), default="incomplete")

# EU AI Act Annex IV sections
ANNEX_IV_SECTIONS = {
    1: "General description of the AI system",
    2: "Detailed description of elements and development process",
    3: "Training, validation, and testing data",
    4: "Detailed information about monitoring, functioning, and control",
    5: "Description of appropriateness of performance metrics",
    6: "Risk management system",
    7: "Changes and modifications throughout lifecycle",
    8: "Human oversight measures",
    9: "Post-market monitoring system",
}

# src/registry/api/schemas/compliance.py
class ComplianceRecordCreate(BaseModel):
    framework: str = Field(..., pattern=r"^(eu_ai_act|nist_ai_rmf|iso_42001)$")
    version_id: uuid.UUID | None = None

class SectionUpdate(BaseModel):
    section_key: str
    status: str = Field(..., pattern=r"^(draft|in_review|approved|expired)$")
    document_uri: str | None = None
    content_summary: str | None = None
    reviewed_by: uuid.UUID | None = None

class ComplianceStatusResponse(BaseModel):
    model_id: uuid.UUID
    framework: str
    total_sections: int
    completed_sections: int
    approved_sections: int
    section_status: dict  # section_key -> {status, document_uri, reviewed_at, ...}
    overall_status: str

# src/registry/api/v1/compliance.py
@router.post("/models/{model_name}/compliance", status_code=201)
async def create_compliance_record(model_name: str, body: ComplianceRecordCreate, ...): ...

@router.get("/models/{model_name}/compliance/{framework}")
async def get_compliance_status(model_name: str, framework: str, ...): ...

@router.put("/models/{model_name}/compliance/{framework}/sections/{section_key}")
async def update_section(model_name: str, framework: str, section_key: str, body: SectionUpdate, ...): ...

@router.post("/models/{model_name}/compliance/{framework}/generate")
async def generate_compliance_docs(model_name: str, framework: str, ...):
    """Auto-generate compliance documentation from registry metadata."""
    ...
```

**Testing**:
- `Integration: POST /compliance creates record with all 9 Annex IV sections in "draft" status`
- `Integration: PUT /compliance/.../sections/section_3 updates section status to "in_review"`
- `Integration: GET /compliance/.../eu_ai_act returns correct completed/approved counts`
- `Integration: overall_status transitions to "compliant" when all sections are "approved"`
- `Integration: section update creates audit_log entry`
- `Unit: ANNEX_IV_SECTIONS contains all 9 mandatory sections`

#### 8.2 — Compliance Document Auto-Generation

**What**: Implement automated generation of Annex IV section content by synthesising registry metadata, model version details, dataset lineage, and evaluation metrics.

**Design**:

```python
# src/registry/services/compliance_service.py
class ComplianceService:
    async def generate_annex_iv(self, model_id: uuid.UUID, version_id: uuid.UUID | None = None) -> dict:
        """Generate EU AI Act Annex IV documentation from registry metadata."""
        model = await self._get_model(model_id)
        versions = await self._get_versions(model_id)
        lineage = await self._get_lineage(model_id)
        latest = versions[0] if versions else None

        sections = {}

        # Section 1: General description
        sections["section_1_general_description"] = {
            "content": {
                "system_name": model.name,
                "intended_purpose": model.governance.get("intended_purpose", ""),
                "risk_classification": model.risk_classification,
                "ai_system_type": model.ai_system_type,
                "provider": model.governance.get("provider", ""),
                "version_count": len(versions),
                "latest_version": latest.version if latest else None,
            },
            "status": "draft",
            "generated_at": datetime.utcnow().isoformat(),
        }

        # Section 3: Training, validation, testing data
        datasets = [l for l in lineage if l.source_type == "dataset_version"]
        sections["section_3_training_data"] = {
            "content": {
                "datasets": [{
                    "name": d.metadata.get("dataset_name"),
                    "version": d.metadata.get("version"),
                    "uri": d.metadata.get("uri"),
                    "num_rows": d.metadata.get("num_rows"),
                    "data_split": d.metadata.get("data_split"),
                    "hash": d.metadata.get("hash"),
                } for d in datasets],
                "training_parameters": latest.parameters if latest else {},
            },
            "status": "draft",
            "generated_at": datetime.utcnow().isoformat(),
        }

        # Section 5: Performance metrics
        sections["section_5_performance_metrics"] = {
            "content": {
                "evaluation_metrics": latest.metrics if latest else {},
                "framework": latest.framework if latest else None,
                "input_schema": latest.input_schema if latest else None,
                "output_schema": latest.output_schema if latest else None,
            },
            "status": "draft",
            "generated_at": datetime.utcnow().isoformat(),
        }

        # Sections 2, 4, 6, 7, 8, 9 scaffolded with available metadata
        ...

        return sections

# Worker task for async generation
@celery_app.task
def generate_compliance_docs_task(model_id: str, framework: str):
    """Async task for generating compliance documentation."""
    ...
```

**Testing**:
- `Unit: generate_annex_iv produces all 9 sections`
- `Unit: section_1 includes model name, intended_purpose, risk_classification`
- `Unit: section_3 includes linked dataset information from lineage`
- `Unit: section_5 includes evaluation metrics from latest model version`
- `Integration: POST /compliance/{framework}/generate creates compliance record with auto-populated sections`
- `Integration: generated sections have status "draft" requiring human review`
- `Integration: model with no lineage generates section_3 with empty datasets list`
- `Integration: model with multiple versions uses latest version for metrics`

---

## Phase 9: LLM Lifecycle Support

### Purpose

Add prompt version tracking, adapter weight management (LoRA/QLoRA), and quantisation variant registration. This extends the registry from traditional ML to cover the full LLM lifecycle, a capability that is nascent across all competitors.

### Tasks

#### 9.1 — Prompt Version Tracking

**What**: Implement prompt versioning with template storage, variable tracking, and configuration metadata.

**Design**:

```python
# src/registry/db/models/prompt_version.py
class PromptVersion(Base):
    __tablename__ = "prompt_versions"
    __table_args__ = (sa.UniqueConstraint("model_id", "version"),)

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default_factory=uuid.uuid4)
    model_id: Mapped[uuid.UUID] = mapped_column(sa.ForeignKey("registered_models.id", ondelete="CASCADE"))
    version: Mapped[int]
    name: Mapped[str | None] = mapped_column(String(255), default=None)
    template: Mapped[str] = mapped_column(sa.Text)
    system_prompt: Mapped[str | None] = mapped_column(sa.Text, default=None)
    config: Mapped[dict] = mapped_column(JSONB, default_factory=dict)
    # config example: {"variables": ["transaction"], "temperature": 0.1, "max_tokens": 256}
    created_by: Mapped[uuid.UUID | None] = mapped_column(sa.ForeignKey("users.id"), default=None)
    created_at: Mapped[datetime] = mapped_column(
        sa.TIMESTAMP(timezone=True), server_default=sa.func.now(), init=False
    )

# src/registry/api/v1/prompts.py
class PromptVersionCreate(BaseModel):
    template: str = Field(..., min_length=1)
    name: str | None = None
    system_prompt: str | None = None
    config: dict = Field(default_factory=dict)

@router.post("/models/{model_name}/prompts", status_code=201)
async def create_prompt_version(model_name: str, body: PromptVersionCreate, ...): ...

@router.get("/models/{model_name}/prompts")
async def list_prompt_versions(model_name: str, ...): ...

@router.get("/models/{model_name}/prompts/{version}")
async def get_prompt_version(model_name: str, version: int, ...): ...

@router.get("/models/{model_name}/prompts/latest")
async def get_latest_prompt(model_name: str, ...): ...
```

**Testing**:
- `Integration: POST /prompts creates prompt version with auto-incremented version`
- `Integration: GET /prompts/latest returns the highest version`
- `Integration: prompt template with variables stores correctly`
- `Integration: prompt config stores temperature, max_tokens, stop_sequences`
- `Unit: PromptVersionCreate validates non-empty template`

#### 9.2 — Adapter Weight and Quantisation Variant Management via framework_metadata

**What**: Document and validate the LLM-specific framework_metadata patterns for adapters and quantisation variants stored as model versions.

**Design**:

Adapters and quantisation variants are registered as model versions with specific `framework_metadata` patterns, following data-model-suggestion-3's approach of using JSONB rather than separate tables.

```python
# src/registry/api/schemas/versions.py (additions)
# JSON Schema validators for framework_metadata by framework type

ADAPTER_METADATA_SCHEMA = {
    "type": "object",
    "properties": {
        "adapter_type": {"type": "string", "enum": ["lora", "qlora", "prefix_tuning", "full_finetune"]},
        "base_model": {"type": "string"},
        "rank": {"type": "integer", "minimum": 1},
        "alpha": {"type": "integer"},
        "target_modules": {"type": "array", "items": {"type": "string"}},
        "trainable_parameters": {"type": "integer"},
        "merged": {"type": "boolean"},
    },
    "required": ["adapter_type", "base_model"],
}

QUANTISATION_METADATA_SCHEMA = {
    "type": "object",
    "properties": {
        "quantisation_method": {"type": "string", "enum": ["gptq", "awq", "gguf", "fp16", "int8", "int4"]},
        "bits": {"type": "integer"},
        "group_size": {"type": "integer"},
        "benchmark_throughput": {"type": "number"},
        "benchmark_latency_ms": {"type": "number"},
    },
    "required": ["quantisation_method", "bits"],
}

# src/registry/services/version_service.py (additions)
def validate_framework_metadata(framework: str, metadata: dict) -> None:
    """Validate framework_metadata against known schemas."""
    if framework == "adapter":
        jsonschema.validate(metadata, ADAPTER_METADATA_SCHEMA)
    elif framework == "quantisation":
        jsonschema.validate(metadata, QUANTISATION_METADATA_SCHEMA)
    # Other frameworks: no strict validation, JSONB stores anything
```

**Testing**:
- `Unit: adapter framework_metadata with type=lora, base_model, rank validates`
- `Unit: adapter framework_metadata missing base_model fails validation`
- `Unit: quantisation framework_metadata with method=awq, bits=4 validates`
- `Unit: quantisation framework_metadata missing bits fails validation`
- `Integration: POST /versions with framework=adapter stores LoRA metadata correctly`
- `Integration: GET /versions?framework=adapter returns only adapter versions`
- `Integration: POST /versions with framework=quantisation stores quantisation metadata`

---

## Phase 10: Python SDK and CLI

### Purpose

Build the Python SDK client library and CLI tool that ML engineers use to interact with the registry from training scripts, CI/CD pipelines, and the command line. This is the primary interface for most users.

### Tasks

#### 10.1 — Python SDK Client

**What**: Implement a typed Python client for all registry operations with automatic pagination and retry logic.

**Design**:

```python
# src/sdk/client.py
import httpx
from dataclasses import dataclass

@dataclass
class RegistryClient:
    base_url: str = "http://localhost:8000/api/v1"
    api_key: str | None = None
    token: str | None = None
    timeout: float = 30.0

    def __post_init__(self):
        headers = {}
        if self.api_key:
            headers["X-API-Key"] = self.api_key
        elif self.token:
            headers["Authorization"] = f"Bearer {self.token}"
        self._client = httpx.Client(
            base_url=self.base_url, headers=headers, timeout=self.timeout
        )

    # Model operations
    def create_model(self, name: str, **kwargs) -> RegisteredModel: ...
    def get_model(self, name: str) -> RegisteredModel: ...
    def list_models(self, **filters) -> list[RegisteredModel]: ...
    def delete_model(self, name: str) -> None: ...

    # Version operations
    def create_version(
        self, model_name: str, artifact_uri: str, framework: str, **kwargs
    ) -> ModelVersion: ...
    def get_version(self, model_name: str, version: int) -> ModelVersion: ...
    def list_versions(self, model_name: str, **filters) -> list[ModelVersion]: ...
    def transition_stage(self, model_name: str, version: int, stage: str, comment: str | None = None) -> ModelVersion: ...

    # Alias operations
    def set_alias(self, model_name: str, alias: str, version: int) -> ModelAlias: ...
    def get_by_alias(self, model_name: str, alias: str) -> ModelVersion: ...

    # Artifact operations
    def upload_artifact(self, model_name: str, version: int, file_path: str) -> None:
        """Upload a model artifact file via presigned URL."""
        # 1. Get presigned upload URL
        # 2. Upload file in chunks
        # 3. Finalize and verify hash
        ...

    def download_artifact(self, model_name: str, version: int, dest_path: str) -> str:
        """Download a model artifact to a local path."""
        ...

    # Convenience: MLflow-compatible log_model
    def log_model(
        self, model_name: str, artifact_path: str, framework: str,
        metrics: dict | None = None, parameters: dict | None = None,
        tags: dict | None = None, run_id: str | None = None,
    ) -> ModelVersion:
        """Register a model in one call: upload artifact, create version, log metadata."""
        ...

# src/sdk/models.py
@dataclass
class RegisteredModel:
    id: str
    name: str
    description: str | None
    risk_classification: str | None
    tags: dict[str, str]
    created_at: str
    updated_at: str

@dataclass
class ModelVersion:
    id: str
    model_id: str
    version: int
    stage: str
    framework: str
    artifact_uri: str
    metrics: dict[str, float]
    created_at: str
```

**Testing**:
- `Unit (mocked API): create_model sends correct POST request`
- `Unit (mocked API): get_model sends GET and deserialises response`
- `Unit (mocked API): list_models with filters passes query parameters`
- `Unit (mocked API): transition_stage sends correct stage transition request`
- `Unit (mocked API): upload_artifact gets presigned URL, uploads, and finalizes`
- `Unit (mocked API): log_model combines upload + create_version in one call`
- `Integration: SDK create_model against running API creates model in DB`
- `Integration: SDK upload + download artifact preserves file integrity (hash match)`
- `E2E: full lifecycle: create_model -> create_version -> upload_artifact -> set_alias -> get_by_alias`

#### 10.2 — CLI Tool

**What**: Implement a CLI using `click` that wraps SDK operations for command-line use.

**Design**:

```python
# src/sdk/cli.py
import click
from registry_sdk import RegistryClient

@click.group()
@click.option("--url", envvar="REGISTRY_URL", default="http://localhost:8000/api/v1")
@click.option("--api-key", envvar="REGISTRY_API_KEY")
@click.pass_context
def cli(ctx, url, api_key):
    ctx.obj = RegistryClient(base_url=url, api_key=api_key)

@cli.group()
def models():
    """Manage registered models."""
    pass

@models.command("create")
@click.argument("name")
@click.option("--description", "-d")
@click.option("--risk", type=click.Choice(["minimal", "limited", "high", "unacceptable"]))
@click.option("--tag", "-t", multiple=True, help="key=value tag")
@click.pass_obj
def create_model(client, name, description, risk, tag): ...

@models.command("list")
@click.option("--format", "fmt", type=click.Choice(["table", "json"]), default="table")
@click.pass_obj
def list_models(client, fmt): ...

@cli.group()
def versions():
    """Manage model versions."""
    pass

@versions.command("create")
@click.argument("model_name")
@click.option("--artifact", "-a", required=True, type=click.Path(exists=True))
@click.option("--framework", "-f", required=True)
@click.option("--metric", "-m", multiple=True, help="key=value metric")
@click.pass_obj
def create_version(client, model_name, artifact, framework, metric): ...

@versions.command("promote")
@click.argument("model_name")
@click.argument("version", type=int)
@click.argument("stage", type=click.Choice(["staging", "production", "archived"]))
@click.option("--comment", "-c")
@click.pass_obj
def promote_version(client, model_name, version, stage, comment): ...

@cli.group()
def aliases():
    """Manage model aliases."""
    pass

# Entry point: `mreg models list`, `mreg versions create ...`, etc.
```

**Testing**:
- `E2E: mreg models create my-model --description "test" creates model`
- `E2E: mreg models list --format json outputs valid JSON`
- `E2E: mreg versions create my-model -a model.pkl -f sklearn creates version and uploads artifact`
- `E2E: mreg versions promote my-model 1 production --comment "approved" transitions stage`
- `E2E: mreg aliases set my-model champion 1 sets alias`
- `Unit: --url and --api-key read from env vars REGISTRY_URL and REGISTRY_API_KEY`
- `Unit: tag parsing splits "key=value" correctly`
- `Unit: metric parsing handles float values like "accuracy=0.95"`

---

## Phase 11: Web UI

### Purpose

Build the React frontend providing model browsing, version comparison, lineage graph visualisation, serving dashboard, and compliance status overview. The UI is an internal tool for ML engineers and MLOps teams.

### Tasks

#### 11.1 — Frontend Scaffolding and Model Registry Browser

**What**: Set up Vite + React + TypeScript project, API client layer, and the model list and detail pages.

**Design**:

```typescript
// frontend/src/api/client.ts
import axios from "axios";

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || "/api/v1",
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem("auth_token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Types
interface RegisteredModel {
  id: string;
  name: string;
  description: string | null;
  risk_classification: string | null;
  tags: Record<string, string>;
  governance: Record<string, unknown>;
  latest_version: number | null;
  created_at: string;
  updated_at: string;
}

interface ModelVersion {
  id: string;
  model_id: string;
  version: number;
  stage: string;
  framework: string;
  artifact_format: string;
  metrics: Record<string, number>;
  tags: Record<string, string>;
  created_at: string;
}

// API functions
export const modelsApi = {
  list: (params?: Record<string, string>) => api.get<PaginatedList<RegisteredModel>>("/models", { params }),
  get: (name: string) => api.get<RegisteredModel>(`/models/${name}`),
  create: (data: CreateModelRequest) => api.post<RegisteredModel>("/models", data),
  getVersions: (name: string) => api.get<ModelVersion[]>(`/models/${name}/versions`),
};

// Pages
// ModelListPage: TanStack Table with sorting, filtering by name/tag/risk, pagination
// ModelDetailPage: tabs for Versions, Aliases, Lineage, Compliance, Audit
```

**Testing**:
- `E2E (Playwright): model list page loads and displays registered models`
- `E2E (Playwright): clicking a model navigates to detail page with version list`
- `E2E (Playwright): filtering by risk classification updates table`
- `E2E (Playwright): pagination controls work correctly`
- `Unit (React Testing Library): ModelListPage renders table headers`
- `Unit (React Testing Library): ModelDetailPage shows correct tabs`

#### 11.2 — Lineage Graph Visualisation

**What**: Implement an interactive lineage graph using React Flow, showing the upstream/downstream dependency chain for a model version.

**Design**:

```typescript
// frontend/src/components/lineage/LineageGraph.tsx
import ReactFlow, { Node, Edge } from "reactflow";

interface LineageNode {
  id: string;
  type: string;  // "model_version", "dataset_version", "serving_endpoint", "code_commit"
  label: string;
  properties: Record<string, unknown>;
  depth: number;
}

// Fetch upstream and downstream lineage from API
// Transform into React Flow nodes and edges
// Color-code by node type:
//   model_version: blue, dataset_version: green, serving_endpoint: purple, code_commit: gray
// Layout: dagre (top-to-bottom hierarchical)
```

**Testing**:
- `E2E (Playwright): lineage tab renders graph with correct node count`
- `E2E (Playwright): clicking a graph node shows node detail popover`
- `E2E (Playwright): graph renders with no lineage shows "No lineage recorded" message`
- `Unit: lineage data transforms correctly into React Flow nodes and edges`

#### 11.3 — Serving Dashboard and Metrics Charts

**What**: Implement the serving endpoint dashboard with traffic split visualisation, metrics charts, and drift alert timeline.

**Design**:

```typescript
// Pages: /serving/endpoints, /serving/endpoints/{name}
// EndpointDetailPage sections:
//   1. Traffic split bar chart (Recharts) showing variant percentages
//   2. Latency time series (p50, p95, p99) line chart
//   3. Request rate / error rate chart
//   4. Active drift monitors with alert badges
//   5. A/B experiment status cards
//   6. Rollback history timeline
```

**Testing**:
- `E2E (Playwright): serving dashboard lists all endpoints with status badges`
- `E2E (Playwright): endpoint detail shows traffic split chart`
- `E2E (Playwright): metrics charts render with time-range selector`
- `E2E (Playwright): drift alerts section shows alert cards with severity colours`

#### 11.4 — Compliance Dashboard

**What**: Implement the compliance overview showing per-model Annex IV section completion status.

**Design**:

```typescript
// ComplianceDashboard: grid of model cards showing completion percentage
// ComplianceDetailPage: 9-section checklist with status badges
//   - Each section: status (draft/in_review/approved), document link, reviewer, date
//   - Progress bar showing approved/total sections
//   - "Generate Documentation" button triggering auto-generation
```

**Testing**:
- `E2E (Playwright): compliance dashboard shows models with progress bars`
- `E2E (Playwright): compliance detail shows all 9 Annex IV sections`
- `E2E (Playwright): clicking "Generate Documentation" triggers generation and shows spinner`
- `E2E (Playwright): approved sections show green checkmark`

---

## Phase 12: MLflow Compatibility Layer and OpenTelemetry

### Purpose

Add an MLflow-compatible REST API surface so teams using MLflow can migrate to this registry with minimal code changes, and integrate OpenTelemetry for production observability. This phase completes the product for production deployment.

### Tasks

#### 12.1 — MLflow REST API Compatibility Layer

**What**: Implement a subset of the MLflow REST API 2.0 (`/api/2.0/mlflow/...`) that maps to the native registry API.

**Design**:

```python
# src/registry/integrations/mlflow/compat.py
from fastapi import APIRouter

mlflow_router = APIRouter(prefix="/api/2.0/mlflow", tags=["mlflow-compat"])

# MLflow Model Registry endpoints:
# POST /api/2.0/mlflow/registered-models/create
# GET  /api/2.0/mlflow/registered-models/get
# GET  /api/2.0/mlflow/registered-models/search
# PATCH /api/2.0/mlflow/registered-models/update
# DELETE /api/2.0/mlflow/registered-models/delete
# POST /api/2.0/mlflow/model-versions/create
# GET  /api/2.0/mlflow/model-versions/get
# GET  /api/2.0/mlflow/model-versions/search
# PATCH /api/2.0/mlflow/model-versions/update
# DELETE /api/2.0/mlflow/model-versions/delete
# POST /api/2.0/mlflow/model-versions/transition-stage
# POST /api/2.0/mlflow/registered-models/alias

@mlflow_router.post("/registered-models/create")
async def mlflow_create_model(body: dict, ...):
    """Map MLflow's create model to native API."""
    name = body.get("name")
    tags = [{"key": t["key"], "value": t["value"]} for t in body.get("tags", [])]
    # Convert to native format and delegate
    ...

@mlflow_router.get("/registered-models/search")
async def mlflow_search_models(filter: str = Query(""), ...):
    """Map MLflow's search syntax (filter_string) to native query."""
    ...
```

**Testing**:
- `Integration: MLflow client (mlflow.MlflowClient) creates model via compat API`
- `Integration: MLflow client searches models via compat API`
- `Integration: MLflow client creates model version via compat API`
- `Integration: MLflow client transitions stage via compat API`
- `Integration: MLflow client sets alias via compat API`
- `Integration: existing MLflow training script works with registry URL swap`
- `E2E: full MLflow workflow (create model -> log model -> register -> promote) via compat layer`

#### 12.2 — OpenTelemetry Integration

**What**: Instrument the FastAPI application with OpenTelemetry for distributed tracing, metrics, and log correlation.

**Design**:

```python
# src/registry/telemetry/otel.py
from opentelemetry import trace, metrics
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider

def init_telemetry(app: FastAPI, settings: Settings) -> None:
    if not settings.otel_enabled:
        return

    # Tracing
    tracer_provider = TracerProvider(resource=Resource.create({"service.name": "ml-model-registry"}))
    tracer_provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(endpoint=settings.otel_exporter_endpoint)))
    trace.set_tracer_provider(tracer_provider)

    # Metrics
    meter_provider = MeterProvider(resource=Resource.create({"service.name": "ml-model-registry"}))
    metrics.set_meter_provider(meter_provider)

    # Auto-instrument
    FastAPIInstrumentor.instrument_app(app)
    SQLAlchemyInstrumentor().instrument(engine=engine)
    HTTPXClientInstrumentor().instrument()

# src/registry/telemetry/metrics.py
# Custom metrics
meter = metrics.get_meter("ml-model-registry")

model_registrations = meter.create_counter(
    "registry.models.registered", description="Number of model registrations"
)
version_creations = meter.create_counter(
    "registry.versions.created", description="Number of model versions created"
)
stage_transitions = meter.create_counter(
    "registry.versions.stage_transitions", description="Stage transition count"
)
drift_alerts = meter.create_counter(
    "registry.drift.alerts", description="Drift alerts fired"
)
artifact_upload_duration = meter.create_histogram(
    "registry.artifacts.upload_duration_seconds", description="Artifact upload duration"
)
```

**Testing**:
- `Unit: init_telemetry with otel_enabled=False does not instrument`
- `Unit: init_telemetry with otel_enabled=True instruments FastAPI and SQLAlchemy`
- `Integration: API request generates trace spans visible in OTel collector`
- `Integration: model registration increments registry.models.registered counter`
- `Integration: stage transition increments registry.versions.stage_transitions counter`
- `Integration: artifact upload records histogram duration`

#### 12.3 — Prometheus Metrics Endpoint

**What**: Expose a `/metrics` endpoint in Prometheus exposition format for Kubernetes monitoring.

**Design**:

```python
# src/registry/telemetry/metrics.py (additions)
from prometheus_client import Counter, Histogram, Gauge, generate_latest

# Prometheus metrics (in addition to OTel)
PROM_REQUESTS = Counter("http_requests_total", "Total HTTP requests", ["method", "path", "status"])
PROM_LATENCY = Histogram("http_request_duration_seconds", "Request latency", ["method", "path"])
PROM_ACTIVE_MODELS = Gauge("registry_active_models", "Number of active registered models")
PROM_PRODUCTION_VERSIONS = Gauge("registry_production_versions", "Versions in production stage")

@health_router.get("/metrics")
async def prometheus_metrics():
    return Response(content=generate_latest(), media_type="text/plain; version=0.0.4")
```

**Testing**:
- `Integration: GET /metrics returns Prometheus-formatted text`
- `Integration: http_requests_total increments on each API call`
- `Integration: registry_active_models reflects actual model count`
- `Integration: Prometheus can scrape /metrics endpoint successfully`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                     ─── required by everything
    │
Phase 2: Registry Core                 ─── requires Phase 1
    │
Phase 3: Artifact Storage & Lineage    ─── requires Phase 2
    │
    ├── Phase 4: Auth & Multi-Tenancy  ─── requires Phase 2 (can parallel with Phase 3)
    │
Phase 5: Serving Orchestration         ─── requires Phase 3 + Phase 4
    │
Phase 6: Monitoring & Drift & Rollback ─── requires Phase 5
    │
Phase 7: Webhooks & Events             ─── requires Phase 2 (can parallel with Phases 5-6)
    │
Phase 8: EU AI Act Compliance          ─── requires Phase 3 (lineage) + Phase 7 (webhooks)
    │
Phase 9: LLM Lifecycle                 ─── requires Phase 2 (can parallel with Phases 5-8)
    │
Phase 10: Python SDK & CLI             ─── requires Phase 4 (auth) + Phase 3 (artifacts)
    │
Phase 11: Web UI                       ─── requires Phase 5 + Phase 6 + Phase 8
    │
Phase 12: MLflow Compat & OTel         ─── requires Phase 2 + Phase 4
```

**Parallelism opportunities:**
- Phases 3 and 4 can be developed concurrently after Phase 2
- Phase 7 (Webhooks) can be developed concurrently with Phases 5 and 6
- Phase 9 (LLM Lifecycle) can be developed concurrently with Phases 5-8
- Phase 10 (SDK/CLI) can begin after Phase 4, concurrently with Phases 5-8
- Phase 12 (MLflow Compat) can be developed concurrently with Phase 11

---

## Definition of Done (per phase)

1. All tasks implemented with complete functionality as specified in the Design section.
2. All unit tests pass (`pytest tests/unit/`).
3. All integration tests pass (`pytest tests/integration/`).
4. Ruff linting passes with zero errors (`ruff check src/ tests/`).
5. Ruff formatting passes (`ruff format --check src/ tests/`).
6. mypy type checking passes in strict mode (`mypy src/`).
7. Docker build succeeds (`docker build -t ml-model-registry .`).
8. `docker compose up` starts all services and health checks pass.
9. Alembic migrations apply cleanly to a fresh database.
10. New API endpoints appear in the auto-generated OpenAPI spec at `/openapi.json`.
11. New configuration options documented in `.env.example`.
12. Audit log entries created for all state-changing operations.
13. Test coverage for new code exceeds 80% (measured by `pytest --cov`).
