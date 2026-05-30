# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Agronomy Advisory Platform · Created: 2026-05-25

## Philosophy

This model treats every agronomic action as an immutable event appended to a central event store. The current state of any entity — a field's soil health, a season's recommendation history, a grower's lifetime yield trend — is derived by replaying or projecting events, not by reading a mutable row. Materialized views (read models) are rebuilt from the event stream to serve different query patterns (CQRS: Command Query Responsibility Segregation).

The motivation for event sourcing in agronomy is compelling: agronomic advice is inherently temporal. "What was the soil potassium level when we issued that fertiliser recommendation last March?" is a question that mutable UPDATE-based systems cannot answer without a separate audit table. In an event-sourced system, this is a trivial replay. Regulators, crop insurance adjusters, and carbon credit auditors all need the ability to reconstruct the exact state of a field at any point in the past — event sourcing provides this by default.

This design is inspired by financial ledger systems, electronic health records (which use similar event-based audit patterns), and modern observability platforms (OpenTelemetry's span/event model). The event store uses a single partitioned table with a JSONB payload, while materialized read models are maintained by PostgreSQL triggers or an application-level projection engine.

**Best for:** Platforms where regulatory compliance and full auditability are paramount (EU Data Act, carbon credit verification, crop insurance claims); organisations that need temporal queries ("what was true on date X?"); teams planning to build ML/AI models on the event stream; platforms that may need to rebuild read models as requirements evolve.

**Trade-offs:**
- (+) Complete audit trail by default — no separate audit_log table needed
- (+) Temporal queries ("state at time T") are native, not retrofitted
- (+) Read models can be rebuilt from scratch when requirements change
- (+) Event stream is a natural input for ML pipelines and analytics
- (+) Enables undo/rollback by appending compensating events
- (-) Higher storage requirements — events are never deleted, read models duplicate data
- (-) Eventual consistency between event store and read models (milliseconds in practice, but architecturally significant)
- (-) More complex application code: commands produce events, projections consume events
- (-) Simple queries like "get field by ID" require a read model rather than a direct table lookup
- (-) Development team must understand event sourcing patterns — steeper learning curve
- (-) Schema evolution of event payloads requires careful versioning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ADAPT Standard 1.0 | Event types mirror ADAPT field operation categories (planting, application, harvest, tillage) |
| ISO 28258:2013 | Soil observation events carry ISO 28258-aligned property names in JSONB payloads |
| OGC SensorThings API | Sensor readings modeled as observation events with SensorThings property vocabulary |
| EPPO Codes | Pest/disease identification events reference EPPO taxonomy codes |
| 4R Nutrient Stewardship | Recommendation events include right_source/rate/time/place in payload |
| IPM Framework | Pest management events tagged with IPM tier (cultural, biological, chemical) |
| EU Data Act (2023/2854) | Event immutability and temporal query support directly address data portability requirements |
| fiboa Specification | Field boundary change events store GeoJSON geometry conforming to fiboa |
| RFC 7946 (GeoJSON) | All geometry in event payloads uses GeoJSON format |

---

## Event Store (Core)

```sql
-- The single source of truth: an append-only event log
CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                     -- the aggregate/entity this event belongs to
    stream_type     VARCHAR(50) NOT NULL,               -- field, season, grower, recommendation, etc.
    event_type      VARCHAR(100) NOT NULL,              -- e.g., FieldCreated, SoilSampleRecorded, RecommendationIssued
    event_version   INTEGER NOT NULL,                   -- monotonically increasing per stream_id
    occurred_at     TIMESTAMPTZ NOT NULL,               -- when the real-world event happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(), -- when we recorded it (always >= occurred_at)
    actor_id        UUID,                               -- user or system that caused the event
    organisation_id UUID NOT NULL,                      -- tenant partition key
    payload         JSONB NOT NULL,                     -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',        -- correlation IDs, source system, IP, etc.
    UNIQUE (stream_id, event_version)
) PARTITION BY RANGE (recorded_at);

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_event_org ON event_store(organisation_id, occurred_at);
CREATE INDEX idx_event_payload ON event_store USING GIN(payload);

-- Example partitions (monthly)
-- CREATE TABLE event_store_2026_01 PARTITION OF event_store
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

### Event Type Catalog

Below are the core event types with example payloads. Each event is self-contained — it carries all the data needed to understand what happened without reading other events.

```sql
-- FieldCreated
-- {
--   "event_type": "FieldCreated",
--   "payload": {
--     "field_id": "uuid...",
--     "farm_id": "uuid...",
--     "grower_id": "uuid...",
--     "name": "North 80",
--     "boundary": {"type": "Polygon", "coordinates": [[[...]], ...]},
--     "area_ha": 32.4,
--     "soil_type": "silt_loam",
--     "irrigation_type": "center_pivot"
--   }
-- }

-- FieldBoundaryUpdated
-- {
--   "event_type": "FieldBoundaryUpdated",
--   "payload": {
--     "field_id": "uuid...",
--     "old_boundary": {"type": "Polygon", "coordinates": [[[...]], ...]},
--     "new_boundary": {"type": "Polygon", "coordinates": [[[...]], ...]},
--     "old_area_ha": 32.4,
--     "new_area_ha": 33.1,
--     "reason": "GPS resurvey corrected southern boundary"
--   }
-- }

-- SeasonStarted
-- {
--   "event_type": "SeasonStarted",
--   "payload": {
--     "season_id": "uuid...",
--     "field_id": "uuid...",
--     "crop_code": "ZEAMX",
--     "variety": "DKC64-69",
--     "season_year": 2026,
--     "target_yield_t_ha": 12.5,
--     "tillage_system": "no_till"
--   }
-- }

-- SoilSampleRecorded
-- {
--   "event_type": "SoilSampleRecorded",
--   "payload": {
--     "sample_id": "uuid...",
--     "field_id": "uuid...",
--     "location": {"lat": 41.8827, "lng": -93.0977},
--     "sampling_date": "2026-03-15",
--     "depth_range_cm": "0-15",
--     "lab_reference": "AGL-2026-44210",
--     "results": {
--       "pH": 6.4,
--       "organic_matter_pct": 3.8,
--       "nitrogen_ppm": 22,
--       "phosphorus_ppm": 35,
--       "potassium_ppm": 180,
--       "cec_meq_100g": 18.5,
--       "method": "Mehlich-3"
--     }
--   }
-- }

-- SensorReadingReceived
-- {
--   "event_type": "SensorReadingReceived",
--   "payload": {
--     "device_id": "uuid...",
--     "field_id": "uuid...",
--     "readings": {
--       "soil_moisture_pct_10cm": 32.5,
--       "soil_moisture_pct_30cm": 41.2,
--       "soil_temp_c_10cm": 18.3
--     }
--   }
-- }

-- SatelliteObservationProcessed
-- {
--   "event_type": "SatelliteObservationProcessed",
--   "payload": {
--     "field_id": "uuid...",
--     "source": "sentinel2",
--     "observation_date": "2026-07-10",
--     "cloud_cover_pct": 5.2,
--     "ndvi": {"mean": 0.72, "min": 0.45, "max": 0.88, "stddev": 0.08},
--     "ndre": {"mean": 0.38},
--     "ndwi": {"mean": 0.18}
--   }
-- }

-- ScoutingVisitCompleted
-- {
--   "event_type": "ScoutingVisitCompleted",
--   "payload": {
--     "visit_id": "uuid...",
--     "season_id": "uuid...",
--     "growth_stage": "V12",
--     "observations": [
--       {
--         "type": "pest",
--         "pest_code": "DIABVI",
--         "severity": "moderate",
--         "incidence_pct": 15
--       }
--     ],
--     "images": ["storage-key-1.jpg", "storage-key-2.jpg"]
--   }
-- }

-- DiseaseDetectedByAI
-- {
--   "event_type": "DiseaseDetectedByAI",
--   "payload": {
--     "image_storage_key": "scouting/2026/07/15/abc123.jpg",
--     "field_id": "uuid...",
--     "model": "agro-vision-v3",
--     "model_version": "3.2.1",
--     "detections": [
--       {"label": "gray_leaf_spot", "pest_code": "CERCZE", "confidence": 0.92}
--     ]
--   }
-- }

-- RecommendationIssued
-- {
--   "event_type": "RecommendationIssued",
--   "payload": {
--     "recommendation_id": "uuid...",
--     "season_id": "uuid...",
--     "type": "fertiliser",
--     "category": "chemical",
--     "priority": "high",
--     "title": "Sidedress nitrogen application needed",
--     "right_source": "UAN 28-0-0",
--     "right_rate": 140,
--     "right_rate_unit": "kg_N/ha",
--     "right_time": "V6 sidedress, within 5 days",
--     "right_place": "banded 15cm from row",
--     "evidence": ["soil_test_N_low", "ndvi_below_average"]
--   }
-- }

-- RecommendationAcceptedByGrower
-- {
--   "event_type": "RecommendationAcceptedByGrower",
--   "payload": {
--     "recommendation_id": "uuid...",
--     "grower_notes": "Will apply Thursday if weather holds"
--   }
-- }

-- RecommendationRejectedByGrower
-- {
--   "event_type": "RecommendationRejectedByGrower",
--   "payload": {
--     "recommendation_id": "uuid...",
--     "reason": "Cost too high this month, will defer to next season"
--   }
-- }

-- PrescriptionGenerated
-- {
--   "event_type": "PrescriptionGenerated",
--   "payload": {
--     "prescription_id": "uuid...",
--     "recommendation_id": "uuid...",
--     "field_id": "uuid...",
--     "type": "variable_rate_fertiliser",
--     "zones": [
--       {"rate": 160, "area_ha": 12.3, "label": "high_yield_zone"},
--       {"rate": 120, "area_ha": 8.7, "label": "low_yield_zone"}
--     ],
--     "mean_rate": 142,
--     "total_product_kg": 12780
--   }
-- }

-- FieldOperationRecorded
-- {
--   "event_type": "FieldOperationRecorded",
--   "payload": {
--     "operation_id": "uuid...",
--     "season_id": "uuid...",
--     "operation_type": "fertiliser_application",
--     "operation_date": "2026-06-20",
--     "product": "UAN 28-0-0",
--     "rate_applied": 140,
--     "rate_unit": "kg_N/ha",
--     "area_treated_ha": 45.2,
--     "prescription_id": "uuid...",
--     "cost": {"value": 3864.60, "currency": "USD"}
--   }
-- }

-- HarvestRecorded
-- {
--   "event_type": "HarvestRecorded",
--   "payload": {
--     "season_id": "uuid...",
--     "yield_t_ha": 12.8,
--     "moisture_pct": 15.5,
--     "quality_grade": "US No. 2 Yellow",
--     "total_production_t": 578.6
--   }
-- }

-- YieldPredicted
-- {
--   "event_type": "YieldPredicted",
--   "payload": {
--     "season_id": "uuid...",
--     "model": "yield-xgboost-v4",
--     "model_version": "4.1.2",
--     "predicted_yield_t_ha": 12.5,
--     "confidence_interval": {"low": 11.2, "high": 14.1, "pct": 90},
--     "input_features": {"gdd_cumulative": 1245, "ndvi_mean": 0.72}
--   }
-- }

-- AlertSent
-- {
--   "event_type": "AlertSent",
--   "payload": {
--     "user_id": "uuid...",
--     "alert_type": "pest_risk",
--     "channel": "sms",
--     "phone": "+1-515-555-0123",
--     "title": "Western corn rootworm risk elevated",
--     "body": "Scout your corn fields this week..."
--   }
-- }
```

---

## Reference Data (Relational — not event-sourced)

Reference/lookup data that changes infrequently remains in conventional relational tables. These are not event-sourced because they represent global knowledge, not tenant-specific state.

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    phone           VARCHAR(30),
    role            VARCHAR(30) NOT NULL DEFAULT 'viewer',
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',
    auth_subject    VARCHAR(255),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_app_user_org ON app_user(organisation_id);

-- Crop reference (EPPO taxonomy)
CREATE TABLE crop_ref (
    eppo_code       VARCHAR(6) PRIMARY KEY,
    scientific_name VARCHAR(255) NOT NULL,
    common_name     VARCHAR(255) NOT NULL,
    crop_group      VARCHAR(50)
);

-- Pest/disease reference (EPPO taxonomy)
CREATE TABLE pest_ref (
    eppo_code       VARCHAR(6) PRIMARY KEY,
    scientific_name VARCHAR(255) NOT NULL,
    common_name     VARCHAR(255),
    pest_type       VARCHAR(30) NOT NULL
);

-- Input product reference
CREATE TABLE product_ref (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    product_type    VARCHAR(30) NOT NULL,
    active_ingredient VARCHAR(255),
    unit_of_measure VARCHAR(20) NOT NULL
);
```

---

## Materialized Read Models (Projections)

These tables are derived from the event store and can be rebuilt at any time by replaying events. They serve the application's read paths.

```sql
-- Current field state (projected from FieldCreated, FieldBoundaryUpdated, etc.)
CREATE MATERIALIZED VIEW mv_field_current AS
SELECT DISTINCT ON (stream_id)
    stream_id AS field_id,
    organisation_id,
    (payload->>'farm_id')::UUID AS farm_id,
    (payload->>'grower_id')::UUID AS grower_id,
    payload->>'name' AS name,
    ST_GeomFromGeoJSON(payload->'boundary') AS boundary,
    (payload->>'area_ha')::NUMERIC AS area_ha,
    payload->>'soil_type' AS soil_type,
    payload->>'irrigation_type' AS irrigation_type,
    occurred_at AS last_updated
FROM event_store
WHERE stream_type = 'field'
  AND event_type IN ('FieldCreated', 'FieldBoundaryUpdated', 'FieldUpdated')
ORDER BY stream_id, event_version DESC;

CREATE UNIQUE INDEX idx_mv_field_id ON mv_field_current(field_id);

-- Current season state
CREATE TABLE read_season (
    season_id       UUID PRIMARY KEY,
    field_id        UUID NOT NULL,
    organisation_id UUID NOT NULL,
    crop_code       VARCHAR(20) NOT NULL,
    variety         VARCHAR(100),
    season_year     INTEGER NOT NULL,
    planting_date   DATE,
    harvest_date    DATE,
    status          VARCHAR(20) NOT NULL,
    yield_t_ha      NUMERIC(10,2),
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_season_field ON read_season(field_id);
CREATE INDEX idx_read_season_year ON read_season(season_year);

-- Latest soil results per field
CREATE TABLE read_soil_latest (
    field_id        UUID NOT NULL,
    depth_range_cm  VARCHAR(20) NOT NULL,
    sampling_date   DATE NOT NULL,
    results         JSONB NOT NULL,
    lab_reference   VARCHAR(100),
    source_event_id UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (field_id, depth_range_cm)
);

-- Recommendation timeline per season
CREATE TABLE read_recommendation (
    recommendation_id UUID PRIMARY KEY,
    season_id       UUID NOT NULL,
    organisation_id UUID NOT NULL,
    type            VARCHAR(30) NOT NULL,
    category        VARCHAR(30),
    priority        VARCHAR(20) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    details         JSONB NOT NULL,
    status          VARCHAR(20) NOT NULL,    -- draft, issued, accepted, rejected, applied
    issued_at       TIMESTAMPTZ,
    grower_response VARCHAR(20),
    grower_notes    TEXT,
    last_event_version INTEGER NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_rec_season ON read_recommendation(season_id);
CREATE INDEX idx_read_rec_status ON read_recommendation(status);

-- Sensor reading time-series (projected from SensorReadingReceived events)
CREATE TABLE read_sensor_timeseries (
    device_id       UUID NOT NULL,
    field_id        UUID NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL,
    readings        JSONB NOT NULL,
    source_event_id UUID NOT NULL
) PARTITION BY RANGE (recorded_at);
CREATE INDEX idx_read_sensor_device ON read_sensor_timeseries(device_id, recorded_at);

-- Weather daily (projected from WeatherObservationReceived events)
CREATE TABLE read_weather_daily (
    field_id        UUID NOT NULL,
    observation_date DATE NOT NULL,
    source          VARCHAR(50) NOT NULL,
    data            JSONB NOT NULL,
    source_event_id UUID NOT NULL,
    PRIMARY KEY (field_id, observation_date, source)
) PARTITION BY RANGE (observation_date);

-- Satellite observation history
CREATE TABLE read_satellite (
    field_id        UUID NOT NULL,
    observation_date DATE NOT NULL,
    source          VARCHAR(50) NOT NULL,
    indices         JSONB NOT NULL,
    source_event_id UUID NOT NULL,
    PRIMARY KEY (field_id, observation_date, source)
);
CREATE INDEX idx_read_satellite_field ON read_satellite(field_id, observation_date);

-- Field operation log per season
CREATE TABLE read_field_operation (
    operation_id    UUID PRIMARY KEY,
    season_id       UUID NOT NULL,
    operation_type  VARCHAR(30) NOT NULL,
    operation_date  DATE NOT NULL,
    details         JSONB NOT NULL,
    source_event_id UUID NOT NULL
);
CREATE INDEX idx_read_field_op_season ON read_field_operation(season_id);

-- Yield prediction history
CREATE TABLE read_yield_prediction (
    prediction_id   UUID PRIMARY KEY,
    season_id       UUID NOT NULL,
    prediction_date DATE NOT NULL,
    predicted_yield NUMERIC(10,2) NOT NULL,
    yield_unit      VARCHAR(20) NOT NULL,
    model_info      JSONB NOT NULL,
    source_event_id UUID NOT NULL
);
CREATE INDEX idx_read_yield_season ON read_yield_prediction(season_id);
```

---

## Projection Engine (Trigger-based)

```sql
-- Example: projection function for recommendation events
CREATE OR REPLACE FUNCTION project_recommendation_event()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.event_type = 'RecommendationIssued' THEN
        INSERT INTO read_recommendation (
            recommendation_id, season_id, organisation_id,
            type, category, priority, title, details,
            status, issued_at, last_event_version, updated_at
        ) VALUES (
            (NEW.payload->>'recommendation_id')::UUID,
            (NEW.payload->>'season_id')::UUID,
            NEW.organisation_id,
            NEW.payload->>'type',
            NEW.payload->>'category',
            NEW.payload->>'priority',
            NEW.payload->>'title',
            NEW.payload,
            'issued',
            NEW.occurred_at,
            NEW.event_version,
            NEW.recorded_at
        )
        ON CONFLICT (recommendation_id) DO UPDATE SET
            status = 'issued',
            issued_at = NEW.occurred_at,
            details = NEW.payload,
            last_event_version = NEW.event_version,
            updated_at = NEW.recorded_at;

    ELSIF NEW.event_type = 'RecommendationAcceptedByGrower' THEN
        UPDATE read_recommendation SET
            status = 'accepted',
            grower_response = 'accepted',
            grower_notes = NEW.payload->>'grower_notes',
            last_event_version = NEW.event_version,
            updated_at = NEW.recorded_at
        WHERE recommendation_id = (NEW.payload->>'recommendation_id')::UUID;

    ELSIF NEW.event_type = 'RecommendationRejectedByGrower' THEN
        UPDATE read_recommendation SET
            status = 'rejected',
            grower_response = 'rejected',
            grower_notes = NEW.payload->>'reason',
            last_event_version = NEW.event_version,
            updated_at = NEW.recorded_at
        WHERE recommendation_id = (NEW.payload->>'recommendation_id')::UUID;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_project_recommendation
    AFTER INSERT ON event_store
    FOR EACH ROW
    WHEN (NEW.event_type LIKE 'Recommendation%')
    EXECUTE FUNCTION project_recommendation_event();
```

---

## Temporal Query Examples

```sql
-- What was the soil state of field X on March 1, 2026?
SELECT payload->'results'
FROM event_store
WHERE stream_type = 'field'
  AND stream_id = 'field-uuid-here'
  AND event_type = 'SoilSampleRecorded'
  AND occurred_at <= '2026-03-01'
ORDER BY occurred_at DESC
LIMIT 1;

-- Full recommendation history for a season (every state change)
SELECT event_type, occurred_at, payload
FROM event_store
WHERE stream_id IN (
    SELECT (payload->>'recommendation_id')::UUID
    FROM event_store
    WHERE event_type = 'RecommendationIssued'
      AND (payload->>'season_id')::UUID = 'season-uuid-here'
)
ORDER BY occurred_at;

-- Rebuild the complete state of a field at any point in time
SELECT event_type, occurred_at, payload
FROM event_store
WHERE stream_id = 'field-uuid-here'
  AND occurred_at <= '2026-06-15T00:00:00Z'
ORDER BY event_version;

-- All pest observations across the organisation in the last 30 days
SELECT
    payload->>'field_id' AS field_id,
    occurred_at,
    obs->>'pest_code' AS pest_code,
    obs->>'severity' AS severity
FROM event_store,
     jsonb_array_elements(payload->'observations') AS obs
WHERE event_type = 'ScoutingVisitCompleted'
  AND organisation_id = 'org-uuid-here'
  AND occurred_at > now() - interval '30 days'
  AND obs->>'type' = 'pest';

-- Recommendation acceptance rate over time
SELECT
    date_trunc('month', occurred_at) AS month,
    COUNT(*) FILTER (WHERE event_type = 'RecommendationAcceptedByGrower') AS accepted,
    COUNT(*) FILTER (WHERE event_type = 'RecommendationRejectedByGrower') AS rejected
FROM event_store
WHERE event_type IN ('RecommendationAcceptedByGrower', 'RecommendationRejectedByGrower')
  AND organisation_id = 'org-uuid-here'
GROUP BY month
ORDER BY month;
```

---

## Chat & Communication (Event-sourced)

Chat conversations are also event-sourced, making the full advisory transcript auditable:

```sql
-- ChatMessageSent
-- {
--   "event_type": "ChatMessageSent",
--   "stream_type": "conversation",
--   "payload": {
--     "conversation_id": "uuid...",
--     "user_id": "uuid...",
--     "role": "user",
--     "content": "My soil test shows potassium at 120 ppm. What should I do?",
--     "season_id": "uuid..."
--   }
-- }

-- ChatResponseGenerated
-- {
--   "event_type": "ChatResponseGenerated",
--   "stream_type": "conversation",
--   "payload": {
--     "conversation_id": "uuid...",
--     "role": "assistant",
--     "content": "A potassium level of 120 ppm is below the critical threshold of 150 ppm for corn. I recommend...",
--     "model": "claude-4-sonnet",
--     "token_count": 342,
--     "tools_used": ["soil_lookup", "recommendation_generator"],
--     "sources_cited": ["soil_sample:uuid...", "crop_ref:ZEAMX"]
--   }
-- }

-- Read model for conversations
CREATE TABLE read_conversation (
    conversation_id UUID PRIMARY KEY,
    user_id         UUID NOT NULL,
    organisation_id UUID NOT NULL,
    season_id       UUID,
    title           VARCHAR(255),
    message_count   INTEGER NOT NULL DEFAULT 0,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    last_message_at TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_conv_user ON read_conversation(user_id);

CREATE TABLE read_chat_message (
    message_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES read_conversation(conversation_id),
    role            VARCHAR(20) NOT NULL,
    content         TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_read_chat_conv ON read_chat_message(conversation_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single partitioned append-only table |
| Reference Data | 5 | organisation, app_user, crop_ref, pest_ref, product_ref |
| Read Models — Entity State | 3 | mv_field_current, read_season, read_soil_latest |
| Read Models — Operations | 3 | read_recommendation, read_field_operation, read_yield_prediction |
| Read Models — Time Series | 3 | read_sensor_timeseries, read_weather_daily, read_satellite |
| Read Models — Communication | 2 | read_conversation, read_chat_message |
| **Total** | **17** | Plus 1 materialized view; read models are rebuildable |

---

## Key Design Decisions

1. **Single event_store table as the sole source of truth** — all state changes across the entire platform flow through one table. This radically simplifies backup, replication, and compliance: to export a grower's complete data history (EU Data Act), you simply `SELECT * FROM event_store WHERE organisation_id = X`.

2. **stream_id + stream_type + event_version for aggregate consistency** — each entity (field, season, recommendation) gets a unique stream_id, and events within a stream are versioned monotonically. The UNIQUE constraint on (stream_id, event_version) prevents concurrent conflicting writes via optimistic concurrency.

3. **occurred_at vs. recorded_at separation** — `occurred_at` is when the real-world event happened (e.g., the soil was sampled on March 15); `recorded_at` is when the system received the event (maybe March 18 when the lab results arrived). This distinction is critical for temporal queries.

4. **Partitioning by recorded_at** — the event store is partitioned by the immutable recorded_at timestamp, enabling efficient time-range queries and old-partition archival without affecting current operations.

5. **Read models are disposable and rebuildable** — every read_* table can be dropped and rebuilt by replaying events from event_store. This means the read schema can evolve freely without data migration — just rebuild the projection.

6. **Trigger-based projections for simplicity** — rather than a separate projection service (Kafka Streams, etc.), PostgreSQL AFTER INSERT triggers on event_store maintain read models synchronously. This trades write throughput for operational simplicity. For high-volume sensor data, an async projection via LISTEN/NOTIFY or a change-data-capture pipeline could replace triggers.

7. **Reference data remains relational** — EPPO crop/pest codes, input products, and user accounts are not event-sourced because they represent slowly-changing global reference data, not tenant-specific agronomic state. Mixing them into the event store would add complexity without benefit.

8. **Event payload is self-contained** — each event carries enough context to be understood independently. A `RecommendationIssued` event includes the recommendation details, not just a recommendation_id reference. This enables event stream consumers (ML pipelines, analytics) to work without joining to read models.

9. **Grower recommendation response as separate events** — rather than updating a recommendation record, acceptance/rejection are separate events. This captures the response timestamp, the grower's reasoning, and supports analytics on response patterns (e.g., which recommendation types are most often rejected?).

10. **Natural ML pipeline input** — the event stream can be exported directly as training data for yield prediction models, recommendation effectiveness analysis, and adaptive prescription learning. Each event is a labeled, timestamped, structured data point — exactly what ML systems need.
