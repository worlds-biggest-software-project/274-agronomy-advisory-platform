# Agronomy Advisory Platform — Phased Development Plan

> Project: 274-agronomy-advisory-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files into a concrete, phased implementation roadmap. The canonical database schema adopts **Data Model Suggestion 1** (entity-centric normalized relational, standards-aligned) as the backbone, selectively borrowing the JSONB-for-variable-data pattern from **Suggestion 2** (soil observations, sensor readings, recommendation detail) and the pest→product→crop knowledge graph from **Suggestion 4** (the recommendation/advisory engine). Event sourcing (Suggestion 3) is deliberately *not* adopted as the core pattern to keep MVP complexity low; auditability is met instead via a partitioned `audit_log` table and append-only recommendation/operation history.

---

## Core Requirements Synthesis

**What it does.** A vendor-neutral, AI-native agronomy advisory platform that ingests weather, soil, satellite, IoT sensor, and field-scouting data and turns it into actionable agronomic recommendations (irrigation, fertility, pest, disease, yield) for crop advisors, growers, cooperatives, and extension services. It is hardware-optional and reaches low-connectivity users via SMS/voice.

**Primary personas.** (1) Commercial crop advisor / agronomist managing many grower clients; (2) Grower / farm manager monitoring their own fields; (3) Cooperative / input-retailer agronomy team; (4) Extension / smallholder programme operator using SMS channels.

**Key differentiators.** Objectivity (not tied to an input/seed vendor); hardware-optional satellite + smartphone capture; AI-native disease/pest ID from phone photos; LLM advisory chatbot translating data to plain language; adaptive recommendations that learn from grower accept/reject; open standards interoperability (ADAPT, fiboa, ISOXML, OGC, EPPO) and a public MCP server (early-mover opportunity per standards.md).

**Deployment model.** Cloud-hosted multi-tenant SaaS, mobile-first PWA + web dashboard, with SMS/voice channels. Self-hostable via Docker Compose.

**Integration surface.** Weather (Tomorrow.io, OpenWeather Agro), satellite (Sentinel Hub / Copernicus, Planet), FMIS aggregation (Leaf, John Deere, Trimble), pest reference (EPPO), SMS/voice (Twilio), LLM providers (Anthropic / OpenAI via a provider abstraction), object storage (S3/GCS).

**Standards to implement.** GeoJSON (RFC 7946) and fiboa for field boundaries; OGC API Features for spatial endpoints; OGC SensorThings entity model for IoT; ISO 28258 vocabulary for soil; EPPO codes for crops/pests; 4R Nutrient Stewardship + IPM tiers for recommendations; ISOXML (ISO 11783-10) for prescription export; OAuth 2.0 / OIDC for auth; OpenAPI 3.1 for the API spec; OWASP API Security Top 10 for hardening; EU Data Act / GDPR data portability; MCP for the AI agent surface.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language (backend) | **Python 3.12** | The platform is ML/LLM-heavy (CV disease ID, yield models, LLM chatbot, geospatial analytics). Python has the deepest ecosystem for these: `sentinelhub-py`, `rasterio`, `shapely`, `scikit-learn`, vision frameworks, and the Anthropic/OpenAI SDKs. |
| API framework | **FastAPI** | Async (needed for fan-out to external weather/satellite APIs), first-class Pydantic v2 validation, and auto-generates the required OpenAPI 3.1 spec. Native dependency injection eases per-request tenant scoping and OWASP object-level auth. |
| Data validation | **Pydantic v2** | Single source of truth for request/response schemas and JSONB payload validation (the variable-data pattern from Suggestion 2 needs app-level structure enforcement). |
| Primary database | **PostgreSQL 16 + PostGIS 3.4** | Suggestion 1 schema is relational + geospatial. PostGIS gives `GEOGRAPHY(4326)` for fiboa/GeoJSON, GiST spatial indexes, area/distance. JSONB + GIN covers variable soil/sensor data. `ltree` (Suggestion 4) gives the farm hierarchy and knowledge-graph edges in one DB — no second datastore. |
| Time-series partitioning | **Native PG declarative partitioning** | `sensor_observation`, `weather_daily`, `audit_log` partitioned by time range (Suggestion 1 decision 3) to stay performant at scale without TimescaleDB ops overhead. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic + GeoAlchemy2** | Mature async ORM with PostGIS support via GeoAlchemy2; Alembic for versioned DDL migrations. |
| Task queue | **Celery + Redis** | Long-running async work (satellite tile fetch + index computation, CV inference, LLM calls, SMS dispatch, daily advisory recompute, webhook ingestion) must run off the request path. Redis doubles as cache + rate-limit store. |
| Scheduler | **Celery Beat** | Daily weather pull, scheduled satellite revisit checks, risk-model recompute, alert dispatch windows. |
| Frontend | **Next.js 16 (App Router) + TypeScript + React** | Web dashboard for advisors (map-heavy, data tables) and an installable mobile-first PWA for growers/scouts (camera capture, offline scouting forms). Server Components for fast initial map loads. |
| Map / GIS UI | **MapLibre GL JS + deck.gl** | Open-source (no Mapbox token lock-in), renders field boundaries, NDVI overlays, prescription zones, scouting pins. deck.gl for large zone/heatmap layers. |
| UI components | **shadcn/ui + Tailwind CSS** | Fast, accessible, themeable component layer for dashboard and forms. |
| LLM / AI gateway | **Provider abstraction over Anthropic + OpenAI** | Chatbot, recommendation drafting, image-to-text fallback. Abstraction allows self-hosters to swap providers; structured tool-calling drives the advisory engine. |
| Computer vision | **PyTorch + timm (EfficientNet/ConvNeXt) served via TorchServe** | Disease/pest image classification from scouting photos; fine-tunable on PlantVillage-style datasets; TorchServe gives a stable inference endpoint the Celery worker calls. |
| Satellite processing | **sentinelhub-py + rasterio + numpy** | Field-polygon statistics for NDVI/NDRE/EVI/NDWI/SAVI per Suggestion 1 satellite tables; OGC WMS/WCS compatible. |
| Weather client | **httpx async clients for Tomorrow.io + OpenWeather Agro** | ET₀, GDD, forecast layers driving irrigation/pest/disease risk models. |
| SMS / voice | **Twilio (Programmable SMS + Voice)** | Mature global reach for the feature-phone accessibility requirement; pluggable behind a `MessagingProvider` interface. |
| MCP server | **`mcp` Python SDK (FastMCP)** | Exposes advisory tools (crop health, field recommendations, pest lookup, weather, submit observation) to MCP-compatible AI assistants — the early-mover opportunity noted in standards.md. |
| Object storage | **S3-compatible (MinIO for self-host, AWS S3 for SaaS)** | Scouting images, satellite cache, prescription exports, report PDFs. |
| Auth | **Authlib (OAuth 2.0 / OIDC) + python-jose** | OIDC SSO for advisors managing multiple accounts; OAuth 2.0 client flows for John Deere/Trimble/Sentinel integrations; local password fallback. |
| Containerisation | **Docker + docker-compose** | One-command self-host; service split (api, worker, beat, torchserve, db, redis, minio, web). |
| Testing | **pytest + pytest-asyncio + testcontainers + Playwright** | Unit + integration (real Postgres/PostGIS via testcontainers), respx for mocking external HTTP APIs, Playwright for web E2E. |
| Code quality | **ruff (lint+format) + mypy (strict) + pre-commit** | Backend. **ESLint + Prettier + tsc** for frontend. |
| Package manager | **uv** (Python), **pnpm** (frontend) | Fast, reproducible installs. |
| CI/CD | **GitHub Actions** | Lint, type-check, test matrix, Docker build, OpenAPI spec diff check. |

### Project Structure

```
agronomy-advisory-platform/
├── README.md
├── docker-compose.yml
├── docker-compose.prod.yml
├── .github/workflows/ci.yml
├── backend/
│   ├── pyproject.toml                 # uv-managed; ruff + mypy config
│   ├── Dockerfile
│   ├── alembic.ini
│   ├── alembic/versions/              # DDL migrations
│   ├── app/
│   │   ├── main.py                    # FastAPI app factory, router registration
│   │   ├── config.py                  # Pydantic Settings (env-driven)
│   │   ├── db/
│   │   │   ├── session.py             # async engine + session
│   │   │   ├── base.py                # declarative base, mixins (TimestampMixin)
│   │   │   └── partitioning.py        # partition-management helpers
│   │   ├── models/                    # SQLAlchemy ORM models (one module per domain)
│   │   │   ├── identity.py organisation.py grower.py field.py season.py
│   │   │   ├── reference.py            # crop, pest, pest_host, input_product
│   │   │   ├── soil.py sensor.py weather.py satellite.py
│   │   │   ├── scouting.py recommendation.py operation.py
│   │   │   ├── ai.py chat.py alert.py financial.py audit.py
│   │   │   └── graph.py               # ltree paths + graph_edge (knowledge graph)
│   │   ├── schemas/                   # Pydantic request/response + JSONB payload models
│   │   ├── api/
│   │   │   ├── deps.py                # auth, tenant scoping, pagination deps
│   │   │   ├── v1/                    # routers: fields, seasons, soil, weather,
│   │   │   │                          #   satellite, scouting, recommendations,
│   │   │   │                          #   prescriptions, yield, chat, alerts, integrations
│   │   │   └── ogc/                   # OGC API Features endpoints (field collections)
│   │   ├── services/                  # business logic (advisory engine, risk models)
│   │   │   ├── advisory/              # 4R/IPM rule engine + LLM drafting
│   │   │   ├── risk/                  # pest/disease/frost/irrigation risk models
│   │   │   ├── yield_model/           # yield prediction
│   │   │   ├── knowledge_graph.py     # pest↔product↔crop traversals
│   │   │   └── learning.py            # adaptive accept/reject feedback loop
│   │   ├── integrations/              # external API clients (one module each)
│   │   │   ├── weather/ satellite/ fmis/ messaging/ eppo/ llm/ vision/ storage/
│   │   ├── exporters/                 # isoxml.py, geojson.py, sarif_unused, pdf_report.py
│   │   ├── workers/                   # celery app + tasks
│   │   ├── mcp/server.py              # FastMCP server exposing advisory tools
│   │   └── security/                  # oauth, rbac, rate_limit, owasp helpers
│   └── tests/
│       ├── unit/ integration/ e2e/ fixtures/
├── frontend/
│   ├── package.json                   # pnpm; Next.js 16
│   ├── Dockerfile
│   ├── app/                           # App Router routes
│   ├── components/                    # map, field-detail, scouting-form, chat, charts
│   ├── lib/                           # API client (generated from OpenAPI), offline store
│   └── tests/                         # Playwright E2E
└── ml/
    ├── disease_classifier/            # training scripts, model card, export to TorchServe
    └── yield_model/                   # feature engineering + training notebooks/scripts
```

---

## Phase 1: Foundation — Project Skeleton, Config, Auth, Multi-Tenancy

### Purpose
Establish the runnable backbone: containerised Postgres/PostGIS/Redis, FastAPI app factory, configuration, database session management, the identity/tenancy tables, and authentication with RBAC. After this phase the system boots, migrations run, a user can register/log in to an organisation, and every subsequent endpoint can be tenant-scoped and authorised. This is intentionally small but load-bearing.

### Tasks

#### 1.1 — Repository scaffold, tooling, and Docker Compose
**What**: Create the monorepo skeleton with `backend`, `frontend`, `ml`, CI, and a `docker-compose.yml` bringing up `db` (postgis/postgis:16-3.4), `redis`, `minio`, `api`, `worker`, `beat`.

**Design**:
- `backend/pyproject.toml` declares deps and tool config:
  ```toml
  [tool.ruff]
  line-length = 100
  target-version = "py312"
  [tool.mypy]
  strict = true
  plugins = ["pydantic.mypy"]
  ```
- `app/config.py` uses `pydantic_settings.BaseSettings`:
  ```python
  class Settings(BaseSettings):
      database_url: PostgresDsn
      redis_url: RedisDsn
      jwt_secret: SecretStr
      jwt_alg: str = "HS256"
      access_token_ttl_min: int = 30
      s3_endpoint: str; s3_bucket: str = "agronomy"
      llm_provider: Literal["anthropic", "openai"] = "anthropic"
      environment: Literal["dev", "staging", "prod"] = "dev"
      model_config = SettingsConfigDict(env_file=".env")
  ```
- `app/main.py` exposes `create_app() -> FastAPI`, mounts `/api/v1`, `/ogc`, `/health`, registers exception handlers, and emits the OpenAPI 3.1 doc at `/openapi.json`.
- `/health` returns `{"status":"ok","db":<bool>,"redis":<bool>}`.

**Testing**:
- `Integration: docker compose up` → `/health` returns 200 with `db=true, redis=true` (testcontainers-backed).
- `Unit: Settings loads from env` → missing `database_url` raises `ValidationError` naming the field.
- `CI: ruff + mypy strict + pytest` all pass on empty skeleton.

#### 1.2 — Database session, base models, migration harness
**What**: Async SQLAlchemy engine/session, declarative base with `TimestampMixin` and `UUIDPrimaryKeyMixin`, PostGIS/`ltree`/`pgcrypto` extension enablement, and Alembic configured for autogenerate + PostGIS types.

**Design**:
- First migration enables extensions:
  ```sql
  CREATE EXTENSION IF NOT EXISTS postgis;
  CREATE EXTENSION IF NOT EXISTS pgcrypto;
  CREATE EXTENSION IF NOT EXISTS ltree;
  ```
- Mixins:
  ```python
  class UUIDPrimaryKeyMixin:
      id: Mapped[UUID] = mapped_column(primary_key=True, server_default=text("gen_random_uuid()"))
  class TimestampMixin:
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())
  ```
- `partitioning.py` exposes `ensure_month_partition(table, month)` for time-series tables.

**Testing**:
- `Integration (real PG): run alembic upgrade head` → extensions present, `alembic current` == head.
- `Unit: TimestampMixin` → `updated_at` changes on update, `created_at` stable.

#### 1.3 — Identity & multi-tenancy models
**What**: Implement `organisation` and `app_user` tables exactly per Suggestion 1 (ISO 3166 codes, roles, OIDC subject), plus repository helpers.

**Design**: Use the DDL from Suggestion 1 §"Core Identity & Multi-Tenancy". Roles enum: `admin | agronomist | grower | viewer`. Every tenant-owned table later carries an `organisation_id` for row-level scoping; a `TenantScopedQuery` helper injects `WHERE organisation_id = :ctx_org`.

**Testing**:
- `Unit: create org + user` → `UNIQUE(organisation_id, email)` violation on duplicate email within org.
- `Integration: two orgs, same email` → both succeed (email unique only within org).

#### 1.4 — Authentication, RBAC, and OWASP baseline
**What**: Local register/login issuing JWT access + refresh tokens, OIDC login via Authlib, a `require_role(...)` dependency, per-IP and per-token rate limiting (Redis), and OWASP API Security controls.

**Design**:
- Endpoints: `POST /api/v1/auth/register`, `POST /api/v1/auth/login`, `POST /api/v1/auth/refresh`, `GET /api/v1/auth/me`, `GET /api/v1/auth/oidc/{provider}` + callback.
- `deps.get_current_user()` decodes JWT, loads user, sets request-scoped tenant context (`organisation_id`). `require_role("admin","agronomist")` raises 403 otherwise.
- Object-level authorisation helper `assert_owned(resource, ctx)` enforces OWASP API1 (BOLA): every resource fetch checks `resource.organisation_id == ctx.org_id` before returning.
- Rate limit middleware: token-bucket in Redis, default 120 req/min/token, 30 req/min/IP on auth routes.
- Passwords hashed with `argon2`. Mass-assignment prevented by explicit Pydantic input schemas (no `**model_dump()` into ORM).

**Testing**:
- `Integration (mocked OIDC): valid login` → 200 + access/refresh tokens; `Unit: expired token` → 401.
- `Integration: user A requests user B's org resource` → 403 (BOLA blocked), no data leaked.
- `Integration: exceed auth rate limit` → 429 with `Retry-After` header.
- `Unit: register with weak/duplicate email` → 422 / 409 respectively.

---

## Phase 2: Spatial Core — Growers, Farms, Fields, Seasons, GeoJSON/fiboa

### Purpose
Build the agronomic backbone every other feature attaches to: the ADAPT-aligned grower→farm→field→season hierarchy with PostGIS field boundaries, GeoJSON/fiboa import-export, area computation, and the OGC API Features endpoint for spatial discovery. After this phase, an advisor can model a real farm operation and the platform can answer "what is at this field, this season".

### Tasks

#### 2.1 — Grower/Farm/Field/Season models and CRUD
**What**: Implement the four hierarchy tables (Suggestion 1 §"Grower/Farm/Field Hierarchy") plus REST CRUD, with field boundary as `GEOGRAPHY(POLYGON,4326)` and `area_ha` auto-computed.

**Design**:
- Endpoints (all tenant-scoped, `require_role` ≥ agronomist for writes):
  - `POST/GET/PATCH/DELETE /api/v1/growers`, `/farms`, `/fields`, `/seasons`
  - `GET /api/v1/fields/{id}/seasons`, `GET /api/v1/farms/{id}/fields`
- Field create accepts GeoJSON Polygon; server computes `area_ha = ST_Area(boundary::geography)/10000` and rejects self-intersecting polygons (`ST_IsValid`).
- Season `crop_code` validated against the `crop` reference table (FK by EPPO code), `status` enum `planned|active|harvested`.

**Testing**:
- `Unit: GeoJSON polygon → area_ha` within 0.5% of known value for a 10 ha square.
- `Integration: invalid (self-intersecting) polygon` → 422 with geometry error.
- `Integration: create field under another org's farm` → 403.

#### 2.2 — fiboa / GeoJSON import & export
**What**: Bulk import field boundaries from a fiboa-compliant GeoJSON FeatureCollection and export the same.

**Design**:
- `POST /api/v1/fields/import` (multipart GeoJSON) → maps fiboa attributes (`id`, `area`, `crop:code`, `determination_datetime`) to `field`/`season`; returns per-feature success/error report.
- `GET /api/v1/fields/export?farm_id=...&format=geojson|fiboa` → RFC 7946 FeatureCollection; fiboa adds the standardised attribute schema.

**Testing**:
- `Fixture: sample fiboa FeatureCollection (5 fields)` import → 5 fields created, areas match `area` attribute within tolerance.
- `Unit: export round-trip` → import then export yields geometrically-equal polygons (`ST_Equals`).
- `Integration: malformed feature` → that feature reported as error, valid ones still imported (partial success).

#### 2.3 — OGC API Features endpoint
**What**: Expose fields as an OGC API Features collection per OGC 17-069r4.

**Design**:
- `GET /ogc/collections` → lists `fields` collection.
- `GET /ogc/collections/fields/items?bbox=&limit=&datetime=` → GeoJSON FeatureCollection, tenant-scoped, with `bbox` spatial filter (`ST_Intersects` on GiST index) and link-based pagination.
- `GET /ogc/collections/fields/items/{id}` → single GeoJSON Feature.
- Conforms to required OGC conformance classes; declared in `GET /ogc/conformance`.

**Testing**:
- `Integration: bbox query` → returns only fields intersecting the box.
- `Unit: conformance doc` → includes core + geojson conformance URIs.
- `Integration: pagination` → `next` link returns the subsequent page with no overlap.

---

## Phase 3: Reference Data — Crops, Pests, Products, and the Knowledge Graph

### Purpose
Load and serve the agronomic reference layer (EPPO crops/pests, input products, pest-host associations) and build the pest↔crop↔product knowledge graph (Suggestion 4) that the recommendation engine in Phase 6 traverses. After this phase the platform can answer "which products control this pest on this crop" — the factual substrate for objective advice.

### Tasks

#### 3.1 — EPPO crop & pest reference ingestion
**What**: Implement `crop`, `pest`, `pest_host` tables (Suggestion 1 §"Crop & Pest Reference Data") and a loader that ingests EPPO Global Database exports.

**Design**:
- `integrations/eppo/loader.py` parses EPPO file-based exports (codes, scientific/common names, pest type, host lists) into the tables; idempotent upsert keyed on `eppo_code`.
- Read endpoints: `GET /api/v1/crops?q=`, `GET /api/v1/pests?q=&crop_code=`, `GET /api/v1/pests/{eppo_code}` (includes host crops).
- Pest type enum: `insect|fungus|bacteria|virus|nematode|weed`.

**Testing**:
- `Fixture: EPPO sample export` → crops/pests/pest_host populated; `ZEAMX` (maize) resolvable.
- `Unit: re-run loader` → no duplicate rows (idempotent upsert).
- `Integration: GET /pests?crop_code=ZEAMX` → returns maize-affecting pests only.

#### 3.2 — Input product catalogue
**What**: Implement `input_product` (fertilisers, pesticides, biologicals) with type, active ingredient, unit of measure, and registration metadata.

**Design**: Suggestion 1 §"input_product". Product type enum `fertiliser|herbicide|fungicide|insecticide|biological|seed_treatment`. Admin-scoped CRUD; org-scoped overlay table `org_product_pricing(product_id, organisation_id, cost_per_unit, currency)` for ROI.

**Testing**:
- `Unit: create product missing unit_of_measure` → 422.
- `Integration: org pricing` → product visible globally, pricing only to owning org.

#### 3.3 — Knowledge graph (pest ↔ product ↔ crop, IPM tiers)
**What**: Implement `graph_edge` (Suggestion 4) layering typed relationships over reference entities and a traversal service.

**Design**:
  ```sql
  CREATE TABLE graph_edge (
      id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      src_type    VARCHAR(30) NOT NULL,   -- pest, crop, product, soil_condition, region
      src_id      UUID NOT NULL,
      edge_type   VARCHAR(40) NOT NULL,   -- controls, hosts, suited_to, antagonises, registered_in
      dst_type    VARCHAR(30) NOT NULL,
      dst_id      UUID NOT NULL,
      ipm_tier    VARCHAR(20),            -- cultural | biological | chemical | mechanical
      attrs       JSONB NOT NULL DEFAULT '{}',
      created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
  );
  CREATE INDEX idx_edge_src ON graph_edge(src_type, src_id, edge_type);
  CREATE INDEX idx_edge_dst ON graph_edge(dst_type, dst_id, edge_type);
  ```
- `services/knowledge_graph.py`:
  ```python
  def control_options(pest_id: UUID, crop_id: UUID, region: str | None) -> list[ControlOption]
  # traverses pest --controls--> product, filtered by crop suitability + regional registration,
  # ordered by IPM tier (cultural→biological→chemical) per IPM framework.
  ```

**Testing**:
- `Fixture: pest P controlled by product F1 (biological) and F2 (chemical)` → `control_options` returns F1 before F2 (IPM ordering).
- `Unit: product not registered in region` → excluded from results.
- `Integration: unknown pest_id` → empty list, no error.

---

## Phase 4: Data Ingestion — Weather & Satellite

### Purpose
Wire the two highest-value external data sources. Weather drives every risk/irrigation model; satellite gives hardware-free crop-health monitoring (the core "accessible to smallholders" differentiator). After this phase, fields automatically accumulate daily weather (ET₀, GDD) and periodic NDVI/NDRE/EVI/NDWI/SAVI statistics. Can be developed in parallel with Phase 5.

### Tasks

#### 4.1 — Weather model and ingestion pipeline
**What**: Implement `weather_station`, `field_weather_station`, `weather_daily` (partitioned) and a Celery task pulling daily forecast + observed weather per field from Tomorrow.io (primary) / OpenWeather Agro (fallback).

**Design**:
- `integrations/weather/base.py`:
  ```python
  class WeatherProvider(Protocol):
      async def daily(self, lat: float, lon: float, start: date, end: date) -> list[DailyWeather]
  class DailyWeather(BaseModel):
      date: date; temp_max_c: float; temp_min_c: float; precipitation_mm: float
      humidity_pct: float; wind_speed_ms: float; solar_radiation_mj: float | None
      et0_mm: float | None
  ```
- Worker computes `et0_mm` (Penman-Monteith FAO-56 if provider omits it) and `gdd_base10 = max(0, (tmax+tmin)/2 - 10)`; upserts `weather_daily` keyed `(station_id, observation_date)`.
- Celery Beat: `pull_weather_daily` at 04:00 per-org timezone; `ensure_month_partition` runs monthly.
- Endpoint: `GET /api/v1/fields/{id}/weather?from=&to=` → daily series.

**Testing**:
- `Integration (respx-mocked Tomorrow.io): pull for field` → `weather_daily` rows created, ET₀/GDD computed.
- `Unit: Penman-Monteith ET₀` → matches FAO-56 worked example within 0.1 mm.
- `Integration: primary provider 500 → fallback used`.
- `Unit: idempotent upsert` → second pull for same date updates, not duplicates.

#### 4.2 — Satellite index ingestion
**What**: Implement `satellite_observation` and `satellite_zone` (Suggestion 1) and a Celery task computing field-polygon index statistics via Sentinel Hub.

**Design**:
- `integrations/satellite/sentinel.py` uses `sentinelhub-py` Statistical API: given field boundary + date range, returns per-date `{ndvi,ndre,evi,ndwi,savi}` mean/min/max/stddev + `cloud_cover_pct`. Observations with cloud_cover > 60% flagged low-quality.
- `satellite_zone` produced by k-means clustering NDVI into low/medium/high/very_high zones (used later for variable-rate prescriptions).
- Beat task `pull_satellite` checks revisit availability per field weekly.
- Endpoints: `GET /api/v1/fields/{id}/satellite?index=ndvi&from=&to=`, `GET /api/v1/fields/{id}/satellite/{obs_id}/zones`.

**Testing**:
- `Integration (mocked Sentinel Statistical API): compute for field` → `satellite_observation` rows with NDVI in [-1,1].
- `Unit: cloud_cover 75%` → observation marked low-quality.
- `Unit: zone clustering` → zones partition field area; summed zone area ≈ field area_ha.

---

## Phase 5: Field Workflow — Soil, Scouting, Operations (Mobile Capture)

### Purpose
Enable the human-in-field workflows: recording soil tests, capturing scouting observations with photos (the input to AI disease ID), and logging field operations/harvest (the input to ROI and yield models). After this phase the platform holds ground-truth agronomic data, not just remote-sensed data. Can be developed in parallel with Phase 4.

### Tasks

#### 5.1 — Soil data model and entry
**What**: Implement the ISO 28258 soil hierarchy (`soil_site` → `soil_profile` → `soil_horizon` → `soil_observation`) from Suggestion 1, with observations stored as typed rows but accepting a flexible parameter set (Suggestion 2 influence).

**Design**:
- `soil_observation.parameter` is a controlled vocabulary (pH, organic_matter_pct, nitrogen_ppm, phosphorus_ppm, potassium_ppm, CEC, ...) validated against a `soil_parameter` lookup; `method` records Mehlich-3/Bray-1/Olsen.
- Endpoints: `POST /api/v1/fields/{id}/soil-sites`, `POST /api/v1/soil-sites/{id}/observations` (batch), `GET /api/v1/fields/{id}/soil` (latest per parameter).
- Lab-import endpoint `POST /api/v1/soil-sites/{id}/import` accepts CSV mapped to parameters.

**Testing**:
- `Unit: unknown parameter` → 422 listing allowed parameters.
- `Integration: CSV lab import` → observations created and linked to correct horizon by depth.
- `Integration: GET field soil` → returns latest value per parameter across sites.

#### 5.2 — Scouting visits, observations, and image upload
**What**: Implement `scouting_visit`, `scouting_observation`, `scouting_image` (Suggestion 1) with S3 image upload via presigned URLs.

**Design**:
- `POST /api/v1/seasons/{id}/scouting-visits` (growth stage, location, notes).
- `POST /api/v1/scouting-visits/{id}/observations` (type, optional `pest_id`, severity, incidence_pct, location).
- `POST /api/v1/scouting-observations/{id}/images/presign` → returns presigned PUT URL + storage_key; client uploads directly to S3; `POST .../images/confirm` records the row. Sets `ai_classification`/`ai_confidence` later (Phase 8).
- Observation type enum `pest|disease|weed|nutrient_deficiency|crop_damage|general`.

**Testing**:
- `Integration (mocked S3): presign → confirm` → `scouting_image` row created with storage_key.
- `Unit: incidence_pct > 100` → 422.
- `Integration: observation referencing pest from another taxonomy` → resolves by pest_id; invalid id → 422.

#### 5.3 — Field operations and harvest results
**What**: Implement `field_operation` and `harvest_result` (Suggestion 1, ADAPT-aligned), capturing planting/application/spray/irrigation/tillage/harvest with cost and optional prescription link.

**Design**:
- `POST /api/v1/seasons/{id}/operations`; `total_cost` auto-computed from `rate_applied * area_treated_ha * cost_per_unit` when product pricing known.
- Harvest operations attach `harvest_result` (yield value/unit, moisture, grade).
- Endpoint `GET /api/v1/seasons/{id}/operations` chronological log.

**Testing**:
- `Unit: cost auto-compute` → correct `total_cost` for known rate/area/price.
- `Integration: harvest op + result` → yield retrievable; non-harvest op rejects harvest_result.

---

## Phase 6: Advisory Engine — Risk Models + 4R/IPM Recommendations + LLM Drafting

### Purpose
The heart of the product. Combine weather, satellite, soil, scouting, and the knowledge graph into objective recommendations structured by 4R Nutrient Stewardship and IPM tiers, with an LLM turning the structured advisory into plain-language guidance. Requires Phases 2–5. This is where the value proposition ships.

### Tasks

#### 6.1 — Risk models (irrigation, pest, disease, frost)
**What**: Deterministic, explainable agronomic risk models producing scored risk signals from weather/satellite/soil/scouting inputs.

**Design**:
- `services/risk/` modules return a common `RiskSignal`:
  ```python
  class RiskSignal(BaseModel):
      kind: Literal["irrigation","pest","disease","frost"]
      score: float            # 0.0–1.0
      level: Literal["low","moderate","high","severe"]
      drivers: list[str]      # human-readable explanation factors
      window: tuple[date, date] | None
  ```
- Irrigation: soil-water-balance from ET₀ minus precipitation/irrigation vs. crop coefficient (Kc by growth stage) → deficit triggers irrigation signal.
- Disease: weather-driven models (e.g., degree-day + leaf-wetness/humidity thresholds per pathogen from knowledge graph attrs) → infection risk windows.
- Pest: GDD accumulation vs. pest development thresholds + scouting incidence.
- Frost: forecast min temp below crop-stage threshold within 72 h.
- Explicit `drivers` make every signal auditable (no black box for the deterministic layer).

**Testing**:
- `Unit: irrigation deficit` → high signal when cumulative ET₀ − rain exceeds threshold.
- `Unit: disease model` → high risk only when humidity + degree-day thresholds both met; `drivers` lists both.
- `Unit: frost` → severe when forecast min < threshold; none otherwise.

#### 6.2 — Recommendation model and 4R/IPM generation
**What**: Implement `recommendation` and `prescription`/`prescription_zone` (Suggestion 1) and a generator turning risk signals + knowledge-graph control options into structured recommendations.

**Design**:
- `recommendation` carries 4R fields (`right_source/right_rate/right_time/right_place`) for fertility and `category` (IPM tier) for protection, plus `status` state machine: `draft → issued → accepted | rejected | applied | expired`.
- `services/advisory/generator.py`:
  ```python
  def generate(season_id: UUID) -> list[Recommendation]
  # 1. gather active RiskSignals; 2. for protection signals, query knowledge_graph.control_options
  #    (IPM-ordered); 3. for fertility, derive rate from soil_observation deficits + 4R rules;
  #    4. produce draft recommendations with drivers carried into `description`.
  ```
- Status transitions enforced by `transition(rec, new_status, actor)` guarding illegal jumps.

**Testing**:
- `Integration: season with high disease risk + known control` → draft pest_control recommendation, IPM-ordered options, drivers populated.
- `Unit: illegal status transition (draft→applied)` → raises; `draft→issued→accepted` allowed.
- `Unit: fertility 4R fields` → potassium deficit yields right_source/rate/time/place populated.

#### 6.3 — LLM advisory drafting and chatbot
**What**: Implement `chat_conversation`/`chat_message` and an LLM layer that (a) renders structured recommendations into plain language and (b) answers grower questions grounded in their field data via tool calling.

**Design**:
- `integrations/llm/base.py` provider abstraction: `complete(messages, tools) -> LLMResult` with `prompt_caching` on the system prompt + reference context.
- System prompt template (stored in `services/advisory/prompts.py`):
  ```
  You are an objective, vendor-neutral agronomy advisor. Use ONLY the provided field
  context and tool results. Cite the driver factors behind each recommendation. Never
  invent product names; only reference products returned by control_options. Express
  uncertainty when data is missing. Follow 4R Nutrient Stewardship and IPM principles.
  ```
- Tools exposed to the model: `get_field_context`, `get_recommendations`, `get_weather`, `get_satellite_health`, `lookup_pest`, `control_options`. Chatbot is retrieval-grounded — no free-form agronomic claims without tool data.
- `POST /api/v1/chat/conversations`, `POST /api/v1/chat/conversations/{id}/messages` (streaming SSE).

**Testing**:
- `Integration (mocked LLM): render recommendation` → plain-language text references the structured drivers and 4R fields.
- `Integration (mocked LLM tool-calls): "what's wrong with field X?"` → model calls `get_satellite_health` + `get_recommendations`; response grounded in returned data.
- `Unit: prompt assembly` → system prompt + field context within token budget; caching markers set.

---

## Phase 7: Decision Output — Prescriptions (ISOXML), Reports, Alerts (SMS/Voice)

### Purpose
Turn recommendations into things growers and machines can act on: ISOXML variable-rate prescriptions for farm equipment, PDF/CSV decision reports for documentation, and SMS/voice alerts for low-connectivity reach (the accessibility differentiator). Requires Phase 6.

### Tasks

#### 7.1 — Variable-rate prescription generation and ISOXML export
**What**: Generate prescription zones from satellite/soil variability and export ISOXML (ISO 11783-10) for upload to Task Controllers.

**Design**:
- `services/advisory/prescription.py` builds `prescription` + `prescription_zone` rows by assigning rates per `satellite_zone` (e.g., higher N where NDVI/yield-potential lower, bounded by agronomic min/max).
- `exporters/isoxml.py` produces a `TASKDATA.XML` (TSK/TZN/PDT/PFD elements) conforming to ISO 11783-10; geometries from PostGIS to ISOXML polygon format.
- `GET /api/v1/prescriptions/{id}/export?format=isoxml|geojson` → ISOXML zip or GeoJSON; sets `isoxml_exported=true`.

**Testing**:
- `Unit: zone rate assignment` → rates within configured min/max; total within agronomic budget.
- `Integration: ISOXML export` → output validates against ISO 11783-10 schema (XSD) and re-imports with equal zone geometries.
- `Unit: GeoJSON export` → valid RFC 7946 FeatureCollection with rate properties.

#### 7.2 — Decision report generation
**What**: Generate per-season PDF/CSV reports documenting recommendations, operations, and outcomes for compliance/record-keeping.

**Design**:
- `exporters/pdf_report.py` renders a season report (field summary, soil snapshot, weather summary, recommendations + grower responses, operations, yield). `GET /api/v1/seasons/{id}/report?format=pdf|csv`.

**Testing**:
- `Integration: generate report` → PDF produced, contains all issued recommendations and their statuses.
- `Unit: CSV report` → one row per recommendation with 4R/IPM columns.

#### 7.3 — Alerts via SMS/voice/push
**What**: Implement `alert` (Suggestion 1) and a dispatch worker delivering risk alerts over Twilio SMS/voice, plus push/email, with delivery tracking.

**Design**:
- `integrations/messaging/base.py`: `MessagingProvider.send_sms(to, body)`, `.place_voice(to, twiml)`.
- Worker `dispatch_alerts` runs on Beat windows; renders short localised SMS (≤160 chars) from high/severe `RiskSignal`s and harvest/frost windows. Alert `status` state machine `pending→sent→delivered|failed`; Twilio status webhook updates `delivered_at`.
- Per-user channel preference + quiet hours respected.

**Testing**:
- `Integration (mocked Twilio): severe frost signal` → SMS queued, status `sent`; webhook → `delivered`.
- `Unit: SMS body localisation/length` → respects locale and ≤160 chars.
- `Integration: quiet hours` → alert deferred, not dropped.

---

## Phase 8: AI Vision — Disease/Pest Identification from Photos

### Purpose
Deliver the headline AI-native capability: instant in-field disease/pest identification from a smartphone photo, replacing scheduled scouting visits. Requires Phase 5 (image upload) and Phase 3 (pest reference for mapping labels to EPPO codes). Can run in parallel with Phase 7.

### Tasks

#### 8.1 — Disease/pest classifier service
**What**: Train/serve a CV classifier mapping scouting images to pest/disease labels with confidence, served via TorchServe.

**Design**:
- `ml/disease_classifier/` trains an EfficientNet/ConvNeXt (timm) on a PlantVillage-style + field-image dataset; outputs a label→EPPO-code map and a model card (classes, accuracy, known limitations).
- TorchServe endpoint `POST /predictions/disease_classifier` returns `[{label, eppo_code, confidence, bbox?}]`.

**Testing**:
- `Unit: label→EPPO map` → every model class resolves to a `pest.eppo_code`.
- `Integration (served model): known sample image` → top label matches expected class with confidence > 0.5.
- `Model: held-out test set` → top-1 accuracy ≥ documented threshold (gate in CI on a small fixture set).

#### 8.2 — Inference pipeline and `ai_detection` integration
**What**: Implement `ai_detection` (Suggestion 1) and a Celery task that runs on image confirm, writes detections, and back-fills `scouting_image.ai_classification`/`ai_confidence`, optionally pre-filling the scouting observation's `pest_id`.

**Design**:
- On `images/confirm`, enqueue `classify_image(scouting_image_id)`; worker calls TorchServe, writes `ai_detection` with `model_name/model_version/confidence/bounding_box`, and if confidence ≥ 0.7 suggests the `pest_id` on the observation (advisory, not authoritative).
- Endpoint `GET /api/v1/scouting-observations/{id}/detections`.

**Testing**:
- `Integration (mocked TorchServe): confirm image` → `ai_detection` row written, `scouting_image` back-filled.
- `Unit: low confidence (<0.7)` → detection stored but `pest_id` not auto-suggested.

---

## Phase 9: Yield Prediction & Adaptive Learning

### Purpose
Deliver field-by-field yield forecasts weeks ahead and close the AI loop by learning from grower accept/reject feedback to personalise future recommendations — the adaptive differentiator from research.md. Requires Phases 4–6 and accumulated history.

### Tasks

#### 9.1 — Yield prediction model
**What**: Implement `yield_prediction` (Suggestion 1) and a model combining weather (GDD/ET₀), satellite (NDVI integral), soil, and management history into a forecast with confidence interval.

**Design**:
- `ml/yield_model/` trains a gradient-boosted regressor (and a baseline crop-stage growth model for cold-start) per crop group; features logged in `yield_prediction.input_features`.
- Beat task `recompute_yield` updates predictions weekly per active season; `GET /api/v1/seasons/{id}/yield`.
- Cold-start: when insufficient history, fall back to regional benchmark × NDVI-derived vigour factor and widen the confidence interval.

**Testing**:
- `Integration: active season with weather+satellite history` → prediction with `confidence_low < predicted < confidence_high`.
- `Unit: cold-start fallback` → produces benchmark-based estimate with wide CI and records the fallback in `input_features`.
- `Model: back-test on harvested seasons` → MAPE ≤ documented threshold on fixture set.

#### 9.2 — Adaptive recommendation learning loop
**What**: Implement `services/learning.py` that mines `recommendation.grower_response` + applied operations + outcomes to bias future recommendation generation per org/region/crop.

**Design**:
- Periodic job aggregates accept/reject rates and post-application outcomes (yield/ROI delta) by recommendation type/region/crop into a `recommendation_preference` table:
  ```sql
  CREATE TABLE recommendation_preference (
      id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      organisation_id UUID NOT NULL REFERENCES organisation(id),
      scope_key VARCHAR(120) NOT NULL,   -- e.g. "pest_control:ZEAMX:US-IA"
      accept_rate NUMERIC(5,4), reject_rate NUMERIC(5,4),
      mean_outcome_delta NUMERIC(10,4),
      sample_size INTEGER NOT NULL,
      updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
      UNIQUE(organisation_id, scope_key)
  );
  ```
- `generator.generate()` consults preferences to re-rank/suppress historically-rejected option types (only when `sample_size` is significant) and surfaces the bias in `drivers` for transparency.

**Testing**:
- `Integration: repeatedly rejected chemical option` → re-ranked below biological for that scope after threshold sample size.
- `Unit: small sample size` → no bias applied (avoids overfitting to noise).
- `Unit: transparency` → applied bias appears in recommendation `drivers`.

---

## Phase 10: Integrations, MCP Server, Compliance, and Hardening

### Purpose
Connect the platform to the wider precision-ag ecosystem (FMIS via Leaf/John Deere/Trimble), publish the early-mover MCP server, and meet EU Data Act / GDPR data-portability and OWASP hardening requirements before launch. Requires the core platform (Phases 1–7).

### Tasks

#### 10.1 — FMIS integration via Leaf unified API
**What**: OAuth 2.0 integration importing field boundaries and field operations from John Deere/Trimble/Climate FieldView through Leaf's unified API.

**Design**:
- `integrations/fmis/leaf.py`: OAuth connect flow per provider; sync pulls GeoJSON boundaries → `field`, operations → `field_operation`, mapped through ADAPT operation types. `POST /api/v1/integrations/leaf/connect`, callback, and `POST /api/v1/integrations/leaf/sync`.

**Testing**:
- `Integration (mocked Leaf): sync` → boundaries imported as fields, planting/application/harvest as operations.
- `Unit: ADAPT operation-type mapping` → each Leaf type maps to the correct `operation_type`.

#### 10.2 — MCP advisory server
**What**: Publish a FastMCP server exposing agronomy tools to MCP-compatible AI assistants (no domain MCP server exists publicly — early-mover per standards.md).

**Design**:
- `mcp/server.py` registers tools: `query_crop_health(field_id)`, `get_field_recommendations(season_id)`, `submit_observation(...)`, `get_weather_forecast(field_id)`, `lookup_pest(query)`, `control_options(pest, crop, region)`. Auth via per-org API token; all tools tenant-scoped and reuse Phase 6 services.

**Testing**:
- `Integration: MCP client lists tools` → all six tools advertised with schemas.
- `Integration: query_crop_health` → returns latest NDVI + risk signals for an authorised field; unauthorised field → error.

#### 10.3 — Data portability, GDPR, and audit
**What**: Implement `audit_log` (partitioned, Suggestion 1), EU Data Act export, and GDPR delete/anonymise.

**Design**:
- `GET /api/v1/growers/{id}/export` → full machine-readable archive (GeoJSON + JSON) of a grower's data, satisfying EU Data Act portability.
- `DELETE /api/v1/users/{id}` → GDPR erase/anonymise PII while preserving anonymised agronomic records.
- DB triggers (or ORM event hooks) write `audit_log` on insert/update/delete of tenant tables.

**Testing**:
- `Integration: grower export` → archive contains fields/seasons/soil/operations/recommendations; re-importable.
- `Integration: GDPR delete` → PII removed, agronomic rows retained but anonymised; audit entry written.
- `Unit: audit on update` → `old_values`/`new_values` captured.

#### 10.4 — OWASP hardening and OpenAPI publication
**What**: Final security pass against OWASP API Security Top 10 and publication of the OpenAPI 3.1 spec + generated TS client.

**Design**:
- Verify BOLA on every resource route, rate limiting on all endpoints, no mass-assignment, security headers, and authenticated `/openapi.json`. Generate the frontend client from the spec in CI; fail the build on spec drift.

**Testing**:
- `Integration: BOLA sweep` — parametrised test attempts cross-org access on every resource route → all 403/404.
- `CI: OpenAPI drift` → generated client matches committed client or build fails.
- `Integration: security headers present` (HSTS, X-Content-Type-Options, etc.).

---

## Phase 11: Frontend — Advisor Dashboard & Grower/Scout PWA

### Purpose
Deliver the human interfaces: a map-centric advisor dashboard and a mobile-first, offline-capable PWA for growers and scouts. Requires a stable API (Phases 2–9). UI work for read-only views can begin earlier against mocked endpoints.

### Tasks

#### 11.1 — App shell, auth, generated API client
**What**: Next.js 16 App Router shell, login/OIDC, role-aware navigation, and the OpenAPI-generated typed API client.

**Design**: Server Components for initial loads; client store for auth/session; `lib/api` generated from `/openapi.json`. Routes: `/login`, `/dashboard`, `/fields/[id]`, `/scouting`, `/chat`, `/admin`.

**Testing**:
- `E2E (Playwright): login → dashboard` → authenticated, role-appropriate nav rendered.
- `E2E: viewer role` → write actions hidden/disabled.

#### 11.2 — Map dashboard, field detail, NDVI/zone overlays
**What**: MapLibre + deck.gl map of fields with NDVI/zone/satellite overlays, field-detail page (weather chart, soil snapshot, recommendations, yield forecast).

**Design**: Field boundaries from OGC items endpoint; NDVI time-series and zone layers from satellite endpoints; recommendation cards with accept/reject actions calling the status-transition API.

**Testing**:
- `E2E: open field` → boundary + latest NDVI overlay render; weather chart loads.
- `E2E: accept recommendation` → status → accepted, reflected on reload.

#### 11.3 — Scouting PWA with camera capture and offline queue
**What**: Installable PWA for in-field scouting: capture photo, record observation, queue offline, sync when connected; show AI detection result.

**Design**: Service worker caches shell + queues unsynced observations in IndexedDB; on reconnect, replays presign→upload→confirm; displays returned `ai_detection` label/confidence.

**Testing**:
- `E2E: offline scouting` → observation captured offline, queued; on reconnect, synced and visible server-side.
- `E2E: AI result display` → after upload, detection label/confidence shown on the observation.

#### 11.4 — Advisory chatbot UI
**What**: Streaming chat interface over the Phase 6 SSE endpoint, scoped to a selected field/season.

**Design**: Conversation list + streaming message view; renders tool-grounded responses; context selector binds a season to the conversation.

**Testing**:
- `E2E (mocked SSE): ask question` → streamed tokens render; conversation persists across reload.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (auth, tenancy)        ─── required by everything
    │
Phase 2: Spatial Core (fields/seasons)     ─── requires 1
    │
Phase 3: Reference Data + Knowledge Graph  ─── requires 1
    │
    ├── Phase 4: Weather & Satellite Ingest ─── requires 2  ┐ (parallel)
    └── Phase 5: Soil/Scouting/Operations   ─── requires 2,3 ┘ (parallel)
              │
Phase 6: Advisory Engine (risk, 4R/IPM, LLM) ─── requires 2,3,4,5  ← core value ships here
              │
    ├── Phase 7: Prescriptions/Reports/Alerts ─── requires 6   ┐ (parallel)
    ├── Phase 8: AI Vision disease ID         ─── requires 3,5  ┤ (parallel)
    └── Phase 9: Yield + Adaptive Learning    ─── requires 4,5,6 ┘ (parallel)
              │
Phase 10: Integrations / MCP / Compliance / Hardening ─── requires 1–7
              │
Phase 11: Frontend (dashboard + PWA + chat)  ─── requires 2–9 (read-only views can start vs mocks)
```

**Parallelism opportunities:**
- Phases **4 and 5** run concurrently once Phases 2–3 are done.
- Phases **7, 8, and 9** run concurrently once Phase 6 is done.
- Phase **11** read-only views can be built against mocked/OpenAPI stubs while backend phases finish.
- The **ML training work** in `ml/` (Phases 8.1, 9.1) is independent of API work and can proceed in parallel from the start, gated only by dataset availability.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass (`pytest`), including mocked-external and real-Postgres (testcontainers) tiers.
3. `ruff` lint + format and `mypy --strict` pass on backend; `eslint` + `tsc` pass on frontend.
4. `docker compose up` builds and starts all services healthy; `/health` green.
5. The phase's feature works end-to-end (demonstrable via API call, E2E test, or worker run).
6. New configuration options are documented in `.env.example` and `README.md`.
7. New API endpoints appear in the auto-generated OpenAPI 3.1 spec and pass the CI spec-drift check.
8. Alembic migration(s) created, reversible, and applied cleanly on a fresh database; time-series tables have partition management.
9. New external integrations have a mocked-provider test and a documented credential/setup path.
10. OWASP checks relevant to new endpoints (BOLA, rate limiting, input validation) are covered by tests.
11. Any ML model ships with a model card (classes, metrics, limitations) and a CI accuracy gate on a fixture set.
```
