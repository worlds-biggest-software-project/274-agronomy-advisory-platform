# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Agronomy Advisory Platform · Created: 2026-05-25

## Philosophy

This model follows classical third-normal-form relational design, giving every domain concept its own table with foreign keys enforcing referential integrity. The schema mirrors the real-world hierarchy of growers, farms, fields, seasons, and operations that the ADAPT Standard 1.0 and Leaf Agriculture API both employ. Every measurement, observation, recommendation, and prescription is stored as a discrete, strongly-typed row with full audit metadata.

The design draws heavily from the ISO 28258 soil data model (site/profile/horizon pattern), the OGC SensorThings API entity model (Thing/Datastream/Observation), the EPPO pest code taxonomy, and the fiboa field boundary specification. By aligning table and column names to these published standards, the platform inherits interoperability with the precision agriculture ecosystem without custom translation layers.

This is the "correct by construction" approach: the database schema itself prevents most classes of data inconsistency. It trades simplicity and write throughput for query flexibility and long-term maintainability.

**Best for:** Teams that value data integrity above all else, need complex cross-entity analytics (e.g., "which fields with soil pH below 6.0 received nitrogen recommendations last season and had above-median yields?"), and operate in regulated environments requiring auditable records.

**Trade-offs:**
- (+) Maximum referential integrity — the database catches errors, not the application
- (+) Complex analytical queries are straightforward with JOINs
- (+) Standards-aligned column names ease integration with ADAPT, Leaf, Sentinel Hub
- (+) Schema is self-documenting; new developers can read the ERD
- (-) High table count (~45-55 tables) increases migration complexity
- (-) Schema changes for new crop types or jurisdictions require DDL migrations
- (-) Write-heavy IoT ingestion may bottleneck on foreign key checks
- (-) Multi-region variation (different soil tests, pest lists) requires many lookup tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ADAPT Standard 1.0 (AgGateway) | Grower/Farm/Field/Season hierarchy; field operation types (planting, application, harvest, tillage) |
| ISO 28258:2013 | Soil site/profile/horizon entity pattern; soil observation vocabulary |
| ISO 11783-10 (ISOBUS/ISOXML) | Prescription and task data structure for variable-rate application exchange |
| fiboa Specification | Field boundary geometry attributes (area in hectares, GeoJSON Polygon, WGS84 CRS) |
| OGC SensorThings API | Thing/Datastream/Observation pattern for IoT sensor data ingestion |
| RFC 7946 (GeoJSON) | All geometry columns stored as GeoJSON-compatible PostGIS types |
| ISO 19115-1:2014 | Metadata attributes for geospatial layers (satellite imagery, yield maps) |
| EPPO Codes | Pest/disease/host plant identification using 5-6 letter EPPO taxonomy codes |
| 4R Nutrient Stewardship | Recommendation structure: source, rate, time, place for fertiliser advice |
| IPM Framework | Pest management recommendation tiers: cultural, biological, chemical |
| OAuth 2.0 / OIDC | User authentication model supporting multi-provider SSO |
| ISO 3166-1/2 | Country and subdivision codes for jurisdiction and region modeling |

---

## Core Identity & Multi-Tenancy

```sql
-- Organisation: the top-level tenant (advisory firm, cooperative, extension service)
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,          -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),               -- ISO 3166-2
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    subscription_tier VARCHAR(30) NOT NULL DEFAULT 'free',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Users: agronomists, growers, administrators
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    phone           VARCHAR(30),
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local', -- local, google, microsoft
    auth_subject    VARCHAR(255),                          -- OIDC subject identifier
    role            VARCHAR(30) NOT NULL DEFAULT 'viewer', -- admin, agronomist, grower, viewer
    locale          VARCHAR(10) NOT NULL DEFAULT 'en',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_app_user_org ON app_user(organisation_id);
CREATE INDEX idx_app_user_email ON app_user(email);
```

---

## Grower / Farm / Field Hierarchy (ADAPT-aligned)

```sql
-- Grower: the farmer or farm business entity
CREATE TABLE grower (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    contact_user_id UUID REFERENCES app_user(id),
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_grower_org ON grower(organisation_id);

-- Farm: a named collection of fields belonging to a grower
CREATE TABLE farm (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grower_id       UUID NOT NULL REFERENCES grower(id),
    name            VARCHAR(255) NOT NULL,
    location        GEOGRAPHY(POINT, 4326),    -- centroid for map display
    elevation_m     NUMERIC(7,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_farm_grower ON farm(grower_id);

-- Field: the fundamental agronomic unit
CREATE TABLE field (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    farm_id         UUID NOT NULL REFERENCES farm(id),
    name            VARCHAR(255) NOT NULL,
    boundary        GEOGRAPHY(POLYGON, 4326) NOT NULL, -- fiboa: GeoJSON Polygon, WGS84
    area_ha         NUMERIC(10,4) NOT NULL,             -- fiboa: area in hectares
    soil_type       VARCHAR(100),
    irrigation_type VARCHAR(50),                         -- rainfed, drip, pivot, flood
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_field_farm ON field(farm_id);
CREATE INDEX idx_field_boundary ON field USING GIST(boundary);

-- Season: a crop cycle on a specific field
CREATE TABLE season (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id),
    crop_code       VARCHAR(20) NOT NULL,               -- EPPO code for the crop
    variety         VARCHAR(100),
    planting_date   DATE,
    harvest_date    DATE,
    season_year     INTEGER NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'planned', -- planned, active, harvested
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_season_field ON season(field_id);
CREATE INDEX idx_season_year ON season(season_year);
```

---

## Crop & Pest Reference Data (EPPO-aligned)

```sql
-- Crop reference table (EPPO taxonomy)
CREATE TABLE crop (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    eppo_code       VARCHAR(6) NOT NULL UNIQUE,  -- e.g., ZEAMX for maize
    scientific_name VARCHAR(255) NOT NULL,
    common_name     VARCHAR(255) NOT NULL,
    family          VARCHAR(100),
    crop_group      VARCHAR(50),                  -- cereals, oilseeds, vegetables, fruits
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_crop_eppo ON crop(eppo_code);

-- Pest/disease reference table (EPPO taxonomy)
CREATE TABLE pest (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    eppo_code       VARCHAR(6) NOT NULL UNIQUE,  -- e.g., PHYTIF for Phytophthora infestans
    scientific_name VARCHAR(255) NOT NULL,
    common_name     VARCHAR(255),
    pest_type       VARCHAR(30) NOT NULL,         -- insect, fungus, bacteria, virus, nematode, weed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_pest_eppo ON pest(eppo_code);

-- Pest-host associations (which pests affect which crops)
CREATE TABLE pest_host (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pest_id         UUID NOT NULL REFERENCES pest(id),
    crop_id         UUID NOT NULL REFERENCES crop(id),
    severity_class  VARCHAR(20),                  -- minor, moderate, major, quarantine
    UNIQUE (pest_id, crop_id)
);
CREATE INDEX idx_pest_host_pest ON pest_host(pest_id);
CREATE INDEX idx_pest_host_crop ON pest_host(crop_id);

-- Product reference table (fertilisers, pesticides, biologicals)
CREATE TABLE input_product (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    product_type    VARCHAR(30) NOT NULL,          -- fertiliser, herbicide, fungicide, insecticide, biological, seed_treatment
    active_ingredient VARCHAR(255),
    formulation     VARCHAR(100),
    manufacturer    VARCHAR(255),
    registration_number VARCHAR(100),
    unit_of_measure VARCHAR(20) NOT NULL,          -- kg, L, mL, g
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Soil Data (ISO 28258-aligned)

```sql
-- Soil site: a location where soil data is collected (ISO 28258 Site)
CREATE TABLE soil_site (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id),
    location        GEOGRAPHY(POINT, 4326) NOT NULL,
    site_name       VARCHAR(100),
    sampling_date   DATE NOT NULL,
    sampled_by_id   UUID REFERENCES app_user(id),
    lab_reference   VARCHAR(100),                  -- external lab sample ID
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_soil_site_field ON soil_site(field_id);

-- Soil profile: vertical section at a site (ISO 28258 Profile)
CREATE TABLE soil_profile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    soil_site_id    UUID NOT NULL REFERENCES soil_site(id),
    profile_type    VARCHAR(30) NOT NULL DEFAULT 'standard', -- standard, composite, transect
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Soil horizon: a layer within a profile (ISO 28258 Horizon)
CREATE TABLE soil_horizon (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    soil_profile_id UUID NOT NULL REFERENCES soil_profile(id),
    designation     VARCHAR(10) NOT NULL,           -- A, B, C, O, etc.
    upper_depth_cm  NUMERIC(6,1) NOT NULL,
    lower_depth_cm  NUMERIC(6,1) NOT NULL,
    texture_class   VARCHAR(30),                    -- sand, loam, clay, silt_loam, etc.
    colour_munsell  VARCHAR(20),                    -- Munsell notation e.g., 10YR 4/3
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Soil observation: individual measurements (ISO 28258 Observation)
CREATE TABLE soil_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    soil_horizon_id UUID REFERENCES soil_horizon(id),
    soil_site_id    UUID NOT NULL REFERENCES soil_site(id), -- direct link when no profile
    parameter       VARCHAR(50) NOT NULL,           -- pH, organic_matter_pct, nitrogen_ppm, phosphorus_ppm, potassium_ppm, CEC, etc.
    value           NUMERIC(12,4) NOT NULL,
    unit            VARCHAR(20) NOT NULL,            -- pH_units, pct, ppm, meq_100g, etc.
    method          VARCHAR(100),                    -- Mehlich-3, Bray-1, Olsen, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_soil_obs_site ON soil_observation(soil_site_id);
CREATE INDEX idx_soil_obs_param ON soil_observation(parameter);
```

---

## IoT Sensor Data (OGC SensorThings-aligned)

```sql
-- Sensor thing: a physical device deployed in a field (SensorThings Thing)
CREATE TABLE sensor_thing (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    device_type     VARCHAR(50) NOT NULL,           -- soil_probe, weather_station, camera_trap, rain_gauge
    manufacturer    VARCHAR(100),
    model           VARCHAR(100),
    location        GEOGRAPHY(POINT, 4326),
    installation_date DATE,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensor_thing_field ON sensor_thing(field_id);

-- Datastream: a typed stream of observations from a sensor (SensorThings Datastream)
CREATE TABLE sensor_datastream (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sensor_thing_id UUID NOT NULL REFERENCES sensor_thing(id),
    observed_property VARCHAR(50) NOT NULL,         -- soil_moisture_pct, soil_temp_c, air_temp_c, humidity_pct, rainfall_mm
    unit_of_measure VARCHAR(20) NOT NULL,
    observation_type VARCHAR(30) NOT NULL DEFAULT 'measurement', -- measurement, count, category
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_datastream_sensor ON sensor_datastream(sensor_thing_id);

-- Sensor observation: individual readings (SensorThings Observation)
CREATE TABLE sensor_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    datastream_id   UUID NOT NULL REFERENCES sensor_datastream(id),
    phenomenon_time TIMESTAMPTZ NOT NULL,           -- when the measurement was taken
    result_value    NUMERIC(12,4) NOT NULL,
    quality_flag    VARCHAR(20) DEFAULT 'good',     -- good, suspect, bad
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (phenomenon_time);
CREATE INDEX idx_sensor_obs_ds_time ON sensor_observation(datastream_id, phenomenon_time);

-- Monthly partitions for sensor observations (example)
-- CREATE TABLE sensor_observation_2026_01 PARTITION OF sensor_observation
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

---

## Weather Data

```sql
-- Weather station: source of weather data for a location
CREATE TABLE weather_station (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    location        GEOGRAPHY(POINT, 4326) NOT NULL,
    source          VARCHAR(50) NOT NULL,           -- tomorrow_io, openweather, noaa, on_farm
    external_id     VARCHAR(100),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_weather_station_loc ON weather_station USING GIST(location);

-- Field-station association
CREATE TABLE field_weather_station (
    field_id        UUID NOT NULL REFERENCES field(id),
    station_id      UUID NOT NULL REFERENCES weather_station(id),
    distance_km     NUMERIC(7,2),
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    PRIMARY KEY (field_id, station_id)
);

-- Daily weather observations
CREATE TABLE weather_daily (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    station_id      UUID NOT NULL REFERENCES weather_station(id),
    observation_date DATE NOT NULL,
    temp_max_c      NUMERIC(5,1),
    temp_min_c      NUMERIC(5,1),
    temp_mean_c     NUMERIC(5,1),
    precipitation_mm NUMERIC(7,2),
    humidity_pct    NUMERIC(5,1),
    wind_speed_ms   NUMERIC(5,1),
    solar_radiation_mj NUMERIC(7,2),              -- MJ/m2/day
    et0_mm          NUMERIC(7,2),                  -- reference evapotranspiration (Penman-Monteith)
    gdd_base10      NUMERIC(6,1),                  -- growing degree days (base 10C)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (station_id, observation_date)
) PARTITION BY RANGE (observation_date);
CREATE INDEX idx_weather_daily_station ON weather_daily(station_id, observation_date);
```

---

## Satellite & Remote Sensing

```sql
-- Satellite observation: field-level vegetation indices from satellite imagery
CREATE TABLE satellite_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id),
    observation_date DATE NOT NULL,
    source          VARCHAR(50) NOT NULL,           -- sentinel2, planet, landsat
    cloud_cover_pct NUMERIC(5,2),
    ndvi_mean       NUMERIC(6,4),                   -- Normalized Difference Vegetation Index
    ndvi_min        NUMERIC(6,4),
    ndvi_max        NUMERIC(6,4),
    ndvi_stddev     NUMERIC(6,4),
    ndre_mean       NUMERIC(6,4),                   -- Normalized Difference Red Edge
    evi_mean        NUMERIC(6,4),                   -- Enhanced Vegetation Index
    ndwi_mean       NUMERIC(6,4),                   -- Normalized Difference Water Index
    savi_mean       NUMERIC(6,4),                   -- Soil Adjusted Vegetation Index
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (field_id, observation_date, source)
);
CREATE INDEX idx_satellite_obs_field ON satellite_observation(field_id, observation_date);

-- Satellite zone map: spatial variability within a field
CREATE TABLE satellite_zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    satellite_observation_id UUID NOT NULL REFERENCES satellite_observation(id),
    zone_geometry   GEOGRAPHY(POLYGON, 4326) NOT NULL,
    zone_label      VARCHAR(20) NOT NULL,           -- low, medium, high, very_high
    ndvi_value      NUMERIC(6,4),
    area_ha         NUMERIC(10,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_satellite_zone_obs ON satellite_zone(satellite_observation_id);
```

---

## Scouting & Field Observations

```sql
-- Scouting visit: a field inspection event
CREATE TABLE scouting_visit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    scout_user_id   UUID NOT NULL REFERENCES app_user(id),
    visit_date      TIMESTAMPTZ NOT NULL,
    location        GEOGRAPHY(POINT, 4326),
    growth_stage    VARCHAR(30),                    -- e.g., V6, R1, heading, flowering
    general_notes   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scouting_season ON scouting_visit(season_id);

-- Scouting observation: individual findings during a visit
CREATE TABLE scouting_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scouting_visit_id UUID NOT NULL REFERENCES scouting_visit(id),
    observation_type VARCHAR(30) NOT NULL,           -- pest, disease, weed, nutrient_deficiency, crop_damage, general
    pest_id         UUID REFERENCES pest(id),        -- if pest/disease observation
    severity        VARCHAR(20),                     -- none, low, moderate, high, severe
    incidence_pct   NUMERIC(5,2),                    -- percentage of plants affected
    location        GEOGRAPHY(POINT, 4326),          -- specific observation location
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scouting_obs_visit ON scouting_observation(scouting_visit_id);
CREATE INDEX idx_scouting_obs_pest ON scouting_observation(pest_id);

-- Scouting image: photos attached to observations
CREATE TABLE scouting_image (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scouting_observation_id UUID NOT NULL REFERENCES scouting_observation(id),
    storage_key     VARCHAR(500) NOT NULL,           -- S3/GCS object key
    content_type    VARCHAR(50) NOT NULL DEFAULT 'image/jpeg',
    file_size_bytes BIGINT,
    ai_classification VARCHAR(100),                  -- AI-detected pest/disease label
    ai_confidence   NUMERIC(5,4),                    -- model confidence score 0.0-1.0
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scouting_img_obs ON scouting_image(scouting_observation_id);
```

---

## Recommendations & Prescriptions (4R / IPM-aligned)

```sql
-- Recommendation: an advisory issued by the platform or an agronomist
CREATE TABLE recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    created_by_id   UUID NOT NULL REFERENCES app_user(id),
    recommendation_type VARCHAR(30) NOT NULL,        -- fertiliser, irrigation, pest_control, disease_control, planting, harvest
    category        VARCHAR(30) NOT NULL,             -- IPM: cultural, biological, chemical, mechanical
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal', -- urgent, high, normal, low
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    -- 4R Nutrient Stewardship fields (for fertiliser recommendations)
    right_source    VARCHAR(100),                     -- product name or nutrient source
    right_rate      NUMERIC(10,2),                    -- application rate
    right_rate_unit VARCHAR(20),                      -- kg/ha, L/ha, units/acre
    right_time      VARCHAR(100),                     -- timing guidance
    right_place     VARCHAR(100),                     -- placement method (broadcast, banded, foliar)
    -- Product reference
    product_id      UUID REFERENCES input_product(id),
    -- Status tracking
    status          VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, issued, accepted, rejected, applied, expired
    issued_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    grower_response VARCHAR(20),                      -- accepted, rejected, modified
    grower_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recommendation_season ON recommendation(season_id);
CREATE INDEX idx_recommendation_type ON recommendation(recommendation_type);
CREATE INDEX idx_recommendation_status ON recommendation(status);

-- Prescription: a spatially-variable application map (ISOXML-compatible)
CREATE TABLE prescription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recommendation_id UUID REFERENCES recommendation(id),
    field_id        UUID NOT NULL REFERENCES field(id),
    season_id       UUID NOT NULL REFERENCES season(id),
    prescription_type VARCHAR(30) NOT NULL,           -- variable_rate_fertiliser, variable_rate_seeding, variable_rate_spray
    product_id      UUID REFERENCES input_product(id),
    total_area_ha   NUMERIC(10,4),
    mean_rate       NUMERIC(10,2),
    rate_unit       VARCHAR(20) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, approved, exported, applied
    isoxml_exported BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prescription_field ON prescription(field_id);

-- Prescription zone: individual application zones within a prescription
CREATE TABLE prescription_zone (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    prescription_id UUID NOT NULL REFERENCES prescription(id),
    zone_geometry   GEOGRAPHY(POLYGON, 4326) NOT NULL,
    rate            NUMERIC(10,2) NOT NULL,
    rate_unit       VARCHAR(20) NOT NULL,
    area_ha         NUMERIC(10,4),
    zone_label      VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_presc_zone_presc ON prescription_zone(prescription_id);
```

---

## Field Operations (ADAPT-aligned)

```sql
-- Field operation: a recorded agronomic activity on a field
CREATE TABLE field_operation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    operation_type  VARCHAR(30) NOT NULL,             -- planting, fertiliser_application, spray_application, irrigation, tillage, harvest
    operation_date  DATE NOT NULL,
    product_id      UUID REFERENCES input_product(id),
    rate_applied    NUMERIC(10,2),
    rate_unit       VARCHAR(20),
    area_treated_ha NUMERIC(10,4),
    cost_per_unit   NUMERIC(10,2),
    cost_currency   CHAR(3),                          -- ISO 4217
    total_cost      NUMERIC(12,2),
    prescription_id UUID REFERENCES prescription(id), -- link to the prescription that guided this operation
    operator_user_id UUID REFERENCES app_user(id),
    equipment       VARCHAR(255),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_field_op_season ON field_operation(season_id);
CREATE INDEX idx_field_op_type ON field_operation(operation_type);
CREATE INDEX idx_field_op_date ON field_operation(operation_date);

-- Harvest result: yield data from a harvest operation
CREATE TABLE harvest_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_operation_id UUID NOT NULL REFERENCES field_operation(id),
    yield_value     NUMERIC(10,2) NOT NULL,
    yield_unit      VARCHAR(20) NOT NULL,             -- t/ha, bu/acre, kg/ha
    moisture_pct    NUMERIC(5,2),
    quality_grade   VARCHAR(30),
    protein_pct     NUMERIC(5,2),
    test_weight     NUMERIC(7,2),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_harvest_result_op ON harvest_result(field_operation_id);
```

---

## Yield Prediction & AI Models

```sql
-- Yield prediction: AI-generated yield forecast for a season
CREATE TABLE yield_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    prediction_date DATE NOT NULL,
    model_name      VARCHAR(100) NOT NULL,
    model_version   VARCHAR(30) NOT NULL,
    predicted_yield NUMERIC(10,2) NOT NULL,
    yield_unit      VARCHAR(20) NOT NULL,
    confidence_low  NUMERIC(10,2),
    confidence_high NUMERIC(10,2),
    confidence_pct  NUMERIC(5,2),                     -- e.g., 90 for 90% confidence interval
    input_features  TEXT[],                            -- list of features used: weather, soil, ndvi, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_yield_pred_season ON yield_prediction(season_id);

-- AI disease detection: results from image-based disease/pest identification
CREATE TABLE ai_detection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scouting_image_id UUID NOT NULL REFERENCES scouting_image(id),
    model_name      VARCHAR(100) NOT NULL,
    model_version   VARCHAR(30) NOT NULL,
    detected_pest_id UUID REFERENCES pest(id),
    label           VARCHAR(100) NOT NULL,
    confidence      NUMERIC(5,4) NOT NULL,
    bounding_box    JSONB,                            -- {"x": 0.1, "y": 0.2, "w": 0.3, "h": 0.4}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ai_detection_image ON ai_detection(scouting_image_id);
```

---

## Advisory Chatbot & Communication

```sql
-- Chat conversation: an LLM advisory session
CREATE TABLE chat_conversation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    season_id       UUID REFERENCES season(id),
    title           VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active', -- active, archived
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_chat_conv_user ON chat_conversation(user_id);

-- Chat message: individual messages in a conversation
CREATE TABLE chat_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES chat_conversation(id),
    role            VARCHAR(20) NOT NULL,              -- user, assistant, system
    content         TEXT NOT NULL,
    token_count     INTEGER,
    model_name      VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_chat_msg_conv ON chat_message(conversation_id);

-- SMS/voice alert: notifications sent to growers
CREATE TABLE alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    season_id       UUID REFERENCES season(id),
    alert_type      VARCHAR(30) NOT NULL,              -- pest_risk, disease_risk, irrigation, frost, harvest_window
    channel         VARCHAR(20) NOT NULL,              -- sms, voice, push, email
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,
    phone_number    VARCHAR(30),
    sent_at         TIMESTAMPTZ,
    delivered_at    TIMESTAMPTZ,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending', -- pending, sent, delivered, failed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_user ON alert(user_id);
CREATE INDEX idx_alert_status ON alert(status);
```

---

## Financial & ROI Tracking

```sql
-- Season financials: cost and revenue summary per season
CREATE TABLE season_financial (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    input_cost      NUMERIC(12,2),                     -- total input costs
    labour_cost     NUMERIC(12,2),
    equipment_cost  NUMERIC(12,2),
    revenue         NUMERIC(12,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',    -- ISO 4217
    roi_pct         NUMERIC(7,2),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (season_id)
);
```

---

## Audit Log

```sql
-- Audit log: tracks all data changes for compliance
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL,              -- INSERT, UPDATE, DELETE
    user_id         UUID REFERENCES app_user(id),
    old_values      JSONB,
    new_values      JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);
CREATE INDEX idx_audit_log_table ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_time ON audit_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | organisation, app_user |
| Grower/Farm/Field/Season | 4 | ADAPT-aligned hierarchy |
| Crop & Pest Reference | 4 | EPPO-aligned taxonomy + products |
| Soil Data | 4 | ISO 28258 site/profile/horizon/observation |
| IoT Sensors | 3 | OGC SensorThings Thing/Datastream/Observation |
| Weather | 3 | station, field association, daily observations |
| Satellite & Remote Sensing | 2 | field-level indices + zone maps |
| Scouting & Observations | 3 | visit, observation, image |
| Recommendations & Prescriptions | 3 | 4R/IPM advisory + variable-rate zones |
| Field Operations | 2 | ADAPT operations + harvest results |
| Yield Prediction & AI | 2 | forecasts + image detection |
| Chat & Alerts | 3 | LLM conversations + SMS/push |
| Financial | 1 | season ROI tracking |
| Audit | 1 | change log |
| **Total** | **37** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — supports distributed ID generation, multi-region deployment, and API URL stability without sequential ID enumeration attacks.

2. **PostGIS GEOGRAPHY type for all geometries** — uses geodetic coordinates (WGS84/EPSG:4326) matching fiboa and GeoJSON standards, enabling accurate distance and area calculations across latitudes without reprojection.

3. **Partitioned time-series tables** — `sensor_observation`, `weather_daily`, and `audit_log` are partitioned by time range to maintain query performance as data grows to billions of rows.

4. **EPPO codes as the canonical pest/crop identifier** — avoids ambiguity of common names across languages and regions; the 5-6 character codes are compact, standardised, and join-friendly.

5. **4R Nutrient Stewardship columns on recommendations** — the right_source/right_rate/right_time/right_place fields structure fertiliser advice according to the industry-standard framework, enabling compliance reporting and advisory quality auditing.

6. **Separate prescription and prescription_zone tables** — models variable-rate application maps as a parent-child hierarchy, supporting ISOXML export for direct upload to farm equipment Task Controllers.

7. **ISO 28258 soil hierarchy** — the site → profile → horizon → observation chain preserves full spatial and depth context for soil data, supporting multi-depth analysis and longitudinal soil health tracking.

8. **Organisation-scoped multi-tenancy** — row-level security at the organisation level rather than schema-per-tenant, keeping operational complexity low while supporting advisory firms managing hundreds of grower clients.

9. **Explicit field_operation to prescription link** — closing the loop between what was recommended and what was actually applied, enabling recommendation accuracy analysis over time.

10. **AI model versioning on predictions** — every yield prediction and disease detection records the model name and version, supporting A/B testing, model comparison, and auditability of AI-driven advice.
