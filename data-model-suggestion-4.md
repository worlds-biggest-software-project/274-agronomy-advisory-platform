# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Agronomy Advisory Platform · Created: 2026-05-25

## Philosophy

This model combines conventional relational tables for operational CRUD with a property graph layer for modeling the dense, many-to-many relationships that are central to agronomy: crops host specific pests, pests are controlled by specific products, products interact with soil chemistry, fields are influenced by neighboring fields' pest pressure, and agronomists serve networks of growers with overlapping crop portfolios. These relationship networks are awkward in pure relational design (requiring dozens of junction tables and recursive CTEs) but are natural as graph traversals.

The design uses PostgreSQL's `ltree` extension for hierarchical paths (grower → farm → field → season), a dedicated `graph_edge` table for arbitrary typed relationships between entities, and relational tables for the time-series and transactional data where tabular storage excels. This is not a full graph database (Neo4j) — it's a pragmatic hybrid that adds graph-query capability to a PostgreSQL deployment without introducing a second database system.

The graph layer is particularly powerful for the AI-native features of the platform: "given this disease observation in field A, which neighboring fields with the same crop are at risk?" is a graph traversal. "Which input products are effective against this pest on this crop in this soil type?" is a multi-hop path query. "Which agronomists have experience managing this disease in this region?" traverses the advisor-grower-field-pest network.

**Best for:** Platforms where relationship discovery and network analysis are core features (e.g., pest spread modeling, advisor-grower matching, product-crop-pest knowledge graphs); teams that want graph-query power without operating a separate graph database; deployments where recommendation quality depends on traversing relationships across the crop-pest-product-soil-weather domain.

**Trade-offs:**
- (+) Multi-hop relationship queries are fast and natural (2-3 JOINs vs. 6-8 in pure relational)
- (+) The graph layer enables pest spread modeling, advisor expertise matching, and knowledge graph construction
- (+) `ltree` paths give fast subtree queries for the farm hierarchy without recursive CTEs
- (+) Single database (PostgreSQL + extensions) — no operational overhead of a graph database
- (+) New relationship types can be added without schema changes (new edge_type values)
- (-) Graph queries require familiarity with ltree and recursive patterns
- (-) The graph_edge table can grow large if not pruned; needs careful indexing
- (-) Not as performant as a dedicated graph database for very deep traversals (>5 hops)
- (-) Hybrid complexity: developers must decide which queries use the graph layer vs. relational JOINs
- (-) Less common pattern — harder to hire for than standard relational or event-sourced designs

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ADAPT Standard 1.0 | Grower/Farm/Field hierarchy modeled as ltree paths |
| ISO 28258:2013 | Soil observations linked to fields via graph edges for spatial soil variability analysis |
| EPPO Codes | Pest and crop nodes in the graph, with EPPO codes as node identifiers |
| fiboa Specification | Field boundary geometry stored in field table (PostGIS, WGS84) |
| OGC SensorThings API | Sensor devices linked to fields via graph edges with relationship metadata |
| 4R Nutrient Stewardship | Recommendation nodes linked to product/crop/field nodes with 4R attributes on edges |
| IPM Framework | Pest management graph: pest → treatment_option edges with IPM tier labels |
| RFC 7946 (GeoJSON) | All spatial data in PostGIS GEOGRAPHY type (GeoJSON-compatible) |
| ISO 3166-1/2 | Region nodes in the graph for jurisdiction-based knowledge lookup |

---

## Prerequisites

```sql
-- Required PostgreSQL extensions
CREATE EXTENSION IF NOT EXISTS "pgcrypto";      -- gen_random_uuid()
CREATE EXTENSION IF NOT EXISTS "postgis";       -- spatial types and functions
CREATE EXTENSION IF NOT EXISTS "ltree";         -- hierarchical path queries
```

---

## Graph Infrastructure

```sql
-- Graph node: every entity in the system is a node
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(50) NOT NULL,           -- organisation, grower, farm, field, season, crop, pest, product, user, region, sensor, recommendation
    label           VARCHAR(255) NOT NULL,           -- human-readable label
    path            LTREE,                           -- hierarchical path (e.g., 'org1.grower5.farm3.field12')
    organisation_id UUID,                            -- tenant scope (NULL for global reference nodes)
    properties      JSONB NOT NULL DEFAULT '{}',     -- node-specific attributes
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_path ON graph_node USING GIST(path);
CREATE INDEX idx_gn_org ON graph_node(organisation_id);
CREATE INDEX idx_gn_props ON graph_node USING GIN(properties);
CREATE INDEX idx_gn_label ON graph_node(label);

-- Graph edge: typed, directed relationships between nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_node(id),
    target_id       UUID NOT NULL REFERENCES graph_node(id),
    edge_type       VARCHAR(50) NOT NULL,           -- owns, manages, grows, hosts, controls, recommends, observes, applies_to, adjacent_to, etc.
    weight          NUMERIC(8,4) DEFAULT 1.0,       -- relationship strength/confidence
    properties      JSONB NOT NULL DEFAULT '{}',     -- edge-specific attributes
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                     -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_source ON graph_edge(source_id, edge_type);
CREATE INDEX idx_ge_target ON graph_edge(target_id, edge_type);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_validity ON graph_edge(valid_from, valid_to);
CREATE INDEX idx_ge_props ON graph_edge USING GIN(properties);
```

### Node Type Examples

```sql
-- Organisation node
-- node_type: 'organisation'
-- label: 'Midwest Agronomy Services'
-- path: 'org_abc123'
-- properties: {"country_code": "US", "subdivision_code": "US-IA", "timezone": "America/Chicago"}

-- Grower node
-- node_type: 'grower'
-- label: 'Johnson Family Farms'
-- path: 'org_abc123.grw_def456'
-- properties: {"contact_email": "bjohnson@example.com", "farm_size_ha": 1200}

-- Farm node
-- node_type: 'farm'
-- label: 'North Farm'
-- path: 'org_abc123.grw_def456.frm_ghi789'
-- properties: {"elevation_m": 320}

-- Field node
-- node_type: 'field'
-- label: 'North 80'
-- path: 'org_abc123.grw_def456.frm_ghi789.fld_jkl012'
-- properties: {"area_ha": 32.4, "soil_type": "silt_loam", "irrigation_type": "center_pivot"}

-- Crop node (global reference)
-- node_type: 'crop'
-- label: 'Maize (Zea mays)'
-- path: NULL
-- organisation_id: NULL
-- properties: {"eppo_code": "ZEAMX", "family": "Poaceae", "crop_group": "cereals"}

-- Pest node (global reference)
-- node_type: 'pest'
-- label: 'Western corn rootworm (Diabrotica virgifera)'
-- path: NULL
-- organisation_id: NULL
-- properties: {"eppo_code": "DIABVI", "pest_type": "insect"}

-- Product node (global reference)
-- node_type: 'product'
-- label: 'Warrior II (lambda-cyhalothrin)'
-- path: NULL
-- organisation_id: NULL
-- properties: {"product_type": "insecticide", "active_ingredient": "lambda-cyhalothrin", "formulation": "CS"}

-- Region node (global reference)
-- node_type: 'region'
-- label: 'Iowa, United States'
-- path: 'rgn_us.rgn_us_ia'
-- organisation_id: NULL
-- properties: {"iso_3166_1": "US", "iso_3166_2": "US-IA", "climate_zone": "Dfa"}
```

### Edge Type Examples

```sql
-- Ownership/management edges:
-- grower --[owns]--> farm
-- farm --[contains]--> field
-- user --[manages]--> grower (agronomist-client relationship)
-- organisation --[employs]--> user

-- Agronomic knowledge graph edges:
-- pest --[hosts]--> crop            (pest X affects crop Y)
-- product --[controls]--> pest      (product X is effective against pest Y)
-- crop --[grows_in]--> region       (crop X is commonly grown in region Y)
-- pest --[prevalent_in]--> region   (pest X is common in region Y)

-- Temporal/seasonal edges:
-- season --[planted_on]--> field    (season X is on field Y)
-- season --[grows]--> crop          (season X grows crop Y)
-- recommendation --[targets]--> season
-- field_operation --[applies]--> product
-- field_operation --[executed_on]--> field

-- Spatial edges:
-- field --[adjacent_to]--> field    (spatial adjacency for pest spread)
-- field --[monitored_by]--> sensor

-- Observation edges:
-- scouting_visit --[observed]--> pest (with severity in edge properties)
-- ai_detection --[identified]--> pest
```

---

## Relational Tables for Operational Data

The graph layer handles relationships and knowledge. Operational, transactional, and time-series data lives in conventional relational tables that reference graph_node IDs.

```sql
-- Field boundary (spatial data too complex for JSONB properties)
CREATE TABLE field_boundary (
    field_node_id   UUID PRIMARY KEY REFERENCES graph_node(id),
    boundary        GEOGRAPHY(POLYGON, 4326) NOT NULL,
    area_ha         NUMERIC(10,4) NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_field_boundary_geom ON field_boundary USING GIST(boundary);

-- Season detail
CREATE TABLE season (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_node_id   UUID NOT NULL REFERENCES graph_node(id),
    crop_eppo_code  VARCHAR(20) NOT NULL,
    variety         VARCHAR(100),
    season_year     INTEGER NOT NULL,
    planting_date   DATE,
    harvest_date    DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'planned',
    properties      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_season_field ON season(field_node_id);

-- Soil sample
CREATE TABLE soil_sample (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_node_id   UUID NOT NULL REFERENCES graph_node(id),
    location        GEOGRAPHY(POINT, 4326),
    sampling_date   DATE NOT NULL,
    sampled_by      UUID REFERENCES graph_node(id),  -- user node
    depth_range_cm  VARCHAR(20) NOT NULL DEFAULT '0-15',
    lab_reference   VARCHAR(100),
    results         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_soil_field ON soil_sample(field_node_id);

-- Sensor reading (time-series)
CREATE TABLE sensor_reading (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_node_id  UUID NOT NULL REFERENCES graph_node(id),
    recorded_at     TIMESTAMPTZ NOT NULL,
    readings        JSONB NOT NULL,
    quality_flag    VARCHAR(20) DEFAULT 'good'
) PARTITION BY RANGE (recorded_at);
CREATE INDEX idx_sensor_reading_device ON sensor_reading(sensor_node_id, recorded_at);

-- Weather daily
CREATE TABLE weather_daily (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_node_id   UUID NOT NULL REFERENCES graph_node(id),
    observation_date DATE NOT NULL,
    source          VARCHAR(50) NOT NULL,
    data            JSONB NOT NULL,
    UNIQUE (field_node_id, observation_date, source)
) PARTITION BY RANGE (observation_date);

-- Satellite observation
CREATE TABLE satellite_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_node_id   UUID NOT NULL REFERENCES graph_node(id),
    observation_date DATE NOT NULL,
    source          VARCHAR(50) NOT NULL,
    cloud_cover_pct NUMERIC(5,2),
    indices         JSONB NOT NULL,
    UNIQUE (field_node_id, observation_date, source)
);
CREATE INDEX idx_satellite_field ON satellite_observation(field_node_id, observation_date);

-- Scouting visit
CREATE TABLE scouting_visit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    scout_node_id   UUID NOT NULL REFERENCES graph_node(id),  -- user node
    visit_date      TIMESTAMPTZ NOT NULL,
    location        GEOGRAPHY(POINT, 4326),
    growth_stage    VARCHAR(30),
    observations    JSONB NOT NULL DEFAULT '[]',
    general_notes   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scouting_season ON scouting_visit(season_id);

-- Scouting image
CREATE TABLE scouting_image (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scouting_visit_id UUID NOT NULL REFERENCES scouting_visit(id),
    storage_key     VARCHAR(500) NOT NULL,
    content_type    VARCHAR(50) NOT NULL DEFAULT 'image/jpeg',
    ai_results      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scouting_img_visit ON scouting_image(scouting_visit_id);

-- Recommendation
CREATE TABLE recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    created_by      UUID NOT NULL REFERENCES graph_node(id),  -- user node
    recommendation_type VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    issued_at       TIMESTAMPTZ,
    grower_response VARCHAR(20),
    grower_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rec_season ON recommendation(season_id);

-- Prescription
CREATE TABLE prescription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recommendation_id UUID REFERENCES recommendation(id),
    field_node_id   UUID NOT NULL REFERENCES graph_node(id),
    season_id       UUID NOT NULL REFERENCES season(id),
    prescription_type VARCHAR(30) NOT NULL,
    product_node_id UUID REFERENCES graph_node(id),
    rate_unit       VARCHAR(20) NOT NULL,
    zones           JSONB NOT NULL,
    summary         JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_presc_field ON prescription(field_node_id);

-- Field operation
CREATE TABLE field_operation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    operation_type  VARCHAR(30) NOT NULL,
    operation_date  DATE NOT NULL,
    product_node_id UUID REFERENCES graph_node(id),
    prescription_id UUID REFERENCES prescription(id),
    operator_node_id UUID REFERENCES graph_node(id),
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_field_op_season ON field_operation(season_id);

-- Yield prediction
CREATE TABLE yield_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    prediction_date DATE NOT NULL,
    predicted_yield NUMERIC(10,2) NOT NULL,
    yield_unit      VARCHAR(20) NOT NULL,
    model_info      JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_yield_season ON yield_prediction(season_id);

-- Chat conversation
CREATE TABLE chat_conversation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_node_id    UUID NOT NULL REFERENCES graph_node(id),
    season_id       UUID REFERENCES season(id),
    title           VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE chat_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES chat_conversation(id),
    role            VARCHAR(20) NOT NULL,
    content         TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Alert
CREATE TABLE alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_node_id    UUID NOT NULL REFERENCES graph_node(id),
    season_id       UUID REFERENCES season(id),
    alert_type      VARCHAR(30) NOT NULL,
    channel         VARCHAR(20) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,
    delivery        JSONB NOT NULL DEFAULT '{}',
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL,
    actor_node_id   UUID REFERENCES graph_node(id),
    changes         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);
CREATE INDEX idx_audit_table ON audit_log(table_name, record_id);
```

---

## Graph Query Examples

### 1. Pest spread risk: which nearby fields grow the same crop and are at risk?

```sql
-- Find fields adjacent to a field where a pest was observed,
-- that grow a crop the pest is known to host
WITH target_field AS (
    SELECT id FROM graph_node WHERE id = 'field-uuid-with-pest'
),
observed_pest AS (
    SELECT target_id AS pest_node_id
    FROM graph_edge
    WHERE source_id = (SELECT id FROM target_field)
      AND edge_type = 'observed'
      AND valid_to IS NULL
    ORDER BY created_at DESC
    LIMIT 1
),
adjacent_fields AS (
    SELECT target_id AS field_id
    FROM graph_edge
    WHERE source_id = (SELECT id FROM target_field)
      AND edge_type = 'adjacent_to'
      AND valid_to IS NULL
),
susceptible_crops AS (
    SELECT target_id AS crop_node_id
    FROM graph_edge
    WHERE source_id = (SELECT pest_node_id FROM observed_pest)
      AND edge_type = 'hosts'
)
SELECT
    f.label AS field_name,
    f.properties->>'area_ha' AS area_ha,
    c.label AS crop_name,
    p.label AS pest_name
FROM adjacent_fields af
JOIN graph_node f ON f.id = af.field_id
JOIN graph_edge grows ON grows.source_id = af.field_id AND grows.edge_type = 'grows' AND grows.valid_to IS NULL
JOIN graph_node c ON c.id = grows.target_id
JOIN susceptible_crops sc ON sc.crop_node_id = c.id
JOIN graph_node p ON p.id = (SELECT pest_node_id FROM observed_pest);
```

### 2. Product knowledge graph: which products control a specific pest on a specific crop?

```sql
SELECT
    prod.label AS product_name,
    prod.properties->>'active_ingredient' AS active_ingredient,
    e.properties->>'efficacy_rating' AS efficacy,
    e.properties->>'ipm_tier' AS ipm_tier
FROM graph_edge e
JOIN graph_node prod ON prod.id = e.source_id
WHERE e.target_id = (SELECT id FROM graph_node WHERE node_type = 'pest' AND properties->>'eppo_code' = 'DIABVI')
  AND e.edge_type = 'controls'
  AND e.valid_to IS NULL
ORDER BY (e.properties->>'efficacy_rating')::numeric DESC;
```

### 3. Agronomist expertise: which advisors have managed this pest in this region?

```sql
WITH pest_node AS (
    SELECT id FROM graph_node WHERE node_type = 'pest' AND properties->>'eppo_code' = 'CERCZE'
),
relevant_visits AS (
    SELECT DISTINCT scout_node_id
    FROM scouting_visit sv
    WHERE sv.observations @> '[{"pest_code": "CERCZE"}]'
),
region_agronomists AS (
    SELECT source_id AS user_id
    FROM graph_edge
    WHERE edge_type = 'manages'
      AND target_id IN (
          SELECT id FROM graph_node
          WHERE node_type = 'grower'
            AND path <@ 'org_abc123'    -- within this organisation
      )
      AND valid_to IS NULL
)
SELECT DISTINCT
    u.label AS agronomist_name,
    u.properties->>'email' AS email,
    COUNT(*) AS times_managed
FROM relevant_visits rv
JOIN graph_node u ON u.id = rv.scout_node_id
JOIN region_agronomists ra ON ra.user_id = u.id
GROUP BY u.id, u.label, u.properties->>'email'
ORDER BY times_managed DESC;
```

### 4. Farm hierarchy traversal using ltree

```sql
-- All fields belonging to a specific grower (subtree query)
SELECT label, properties
FROM graph_node
WHERE node_type = 'field'
  AND path <@ 'org_abc123.grw_def456';  -- all descendants of this grower

-- All entities under an organisation
SELECT node_type, COUNT(*)
FROM graph_node
WHERE path <@ 'org_abc123'
GROUP BY node_type;

-- Find the grower for a given field (ancestor query)
SELECT label, properties
FROM graph_node
WHERE node_type = 'grower'
  AND 'org_abc123.grw_def456.frm_ghi789.fld_jkl012' <@ path;
```

### 5. Building spatial adjacency edges

```sql
-- Automatically create adjacent_to edges for fields within 500m of each other
INSERT INTO graph_edge (source_id, target_id, edge_type, properties)
SELECT
    a.field_node_id AS source_id,
    b.field_node_id AS target_id,
    'adjacent_to',
    jsonb_build_object('distance_m', ST_Distance(a.boundary::geography, b.boundary::geography))
FROM field_boundary a
CROSS JOIN field_boundary b
WHERE a.field_node_id != b.field_node_id
  AND ST_DWithin(a.boundary::geography, b.boundary::geography, 500)
ON CONFLICT DO NOTHING;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Infrastructure | 2 | graph_node, graph_edge |
| Spatial | 1 | field_boundary |
| Seasonal | 1 | season |
| Soil | 1 | soil_sample |
| Sensors & Weather | 2 | sensor_reading, weather_daily |
| Satellite | 1 | satellite_observation |
| Scouting | 2 | scouting_visit, scouting_image |
| Recommendations & Prescriptions | 2 | recommendation, prescription |
| Field Operations | 1 | field_operation |
| Yield Prediction | 1 | yield_prediction |
| Chat & Alerts | 3 | conversation, message, alert |
| Audit | 1 | audit_log |
| **Total** | **18** | Plus graph_node/graph_edge encode all entities and relationships |

---

## Key Design Decisions

1. **graph_node as the universal entity registry** — every concept in the system (fields, growers, crops, pests, products, users, regions) is a row in graph_node. This means any entity can participate in graph queries, and new entity types can be added without schema changes. The `node_type` field discriminates, and the `properties` JSONB holds type-specific attributes.

2. **graph_edge with temporal validity** — edges have `valid_from` and `valid_to` timestamps, enabling historical queries ("who was managing this grower in 2024?") and soft-delete of relationships. Current relationships have `valid_to IS NULL`.

3. **ltree paths for hierarchy, graph_edge for everything else** — the grower → farm → field hierarchy is modeled as ltree paths on graph_node because subtree queries (`<@`, `@>`) are the dominant access pattern and ltree is optimized for this. Non-hierarchical relationships (pest-hosts-crop, product-controls-pest, field-adjacent_to-field) use graph_edge because they don't fit a tree structure.

4. **Relational tables reference graph_node IDs** — operational tables (season, soil_sample, sensor_reading) have foreign keys to graph_node rather than to separate grower/farm/field tables. This unified reference model means a single JOIN to graph_node resolves the entity label, type, and properties regardless of what kind of entity it is.

5. **Agronomic knowledge graph as first-class data** — the crop-pest-product-region relationships form a domain knowledge graph that can be queried independently of operational farm data. This knowledge graph can be seeded from EPPO data, enriched by agronomist observations, and used by AI models to generate recommendations.

6. **Spatial adjacency as graph edges** — rather than computing field adjacency at query time (expensive), the system pre-computes `adjacent_to` edges using PostGIS distance functions. These edges enable fast pest-spread risk queries without runtime spatial computation.

7. **Weight on edges for confidence scoring** — the `weight` column on graph_edge allows relationship strength to be quantified. A product-controls-pest edge with weight 0.95 is a well-established treatment; weight 0.6 is marginal. AI recommendation engines can traverse weighted paths to rank treatment options.

8. **No separate user/grower/farm/field tables** — these are all graph_node rows distinguished by node_type. This is the most significant departure from the other models. The advantage is uniform graph traversal; the cost is that type-specific constraints (e.g., "all fields must have a boundary") must be enforced in application code or via CHECK constraints on the properties JSONB.

9. **Graph supports recommendation explanation** — when the AI generates a recommendation, the reasoning path through the knowledge graph (field → observed pest → pest hosts crop → product controls pest) can be stored as a sequence of edge IDs, providing explainable AI with full provenance.

10. **Extensibility without DDL changes** — new node types (e.g., "carbon_credit_project", "insurance_policy") and edge types (e.g., "insured_by", "contributes_to_carbon_credit") can be added as data, not schema. This makes the platform highly adaptable to emerging requirements like carbon credit tracking and crop insurance integration.
