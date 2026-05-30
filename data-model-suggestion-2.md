# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Agronomy Advisory Platform · Created: 2026-05-25

## Philosophy

This model keeps a relational backbone for the core entities (organisations, growers, farms, fields, seasons) where referential integrity matters, but pushes domain-variable data into JSONB columns. Soil test parameters, pest observation attributes, weather data layers, and recommendation details all live in typed JSONB fields rather than dedicated columns or lookup tables.

The insight driving this design is that agronomy is inherently variable: a soil test in Iowa measures different parameters than one in Maharashtra; a pest scouting form for vineyards captures different severity metrics than one for rice paddies; fertiliser recommendations in the EU must include regulatory fields that don't exist in other jurisdictions. A fully normalized schema either proliferates nullable columns or requires dozens of lookup tables. JSONB absorbs this variability while keeping the structural skeleton relational.

This is the "flexible core" approach used by platforms like Airtable, Notion, and many multi-tenant SaaS products. PostgreSQL's JSONB operators, GIN indexes, and generated columns give near-relational query performance on the semi-structured data, while the relational tables enforce the invariants that truly must hold everywhere.

**Best for:** Rapid MVP development with small teams; platforms serving diverse geographies, crop types, and regulatory environments where the "shape" of agronomic data varies significantly; teams that want schema evolution without DDL migrations for every new crop parameter.

**Trade-offs:**
- (+) Far fewer tables (~20-25 vs. ~37-55) — simpler migrations, less ORM boilerplate
- (+) Adding a new soil parameter or pest attribute requires no schema change
- (+) Region-specific and crop-specific fields coexist without NULLable column sprawl
- (+) JSONB containment queries (@>) and GIN indexes provide good read performance
- (-) No database-level enforcement of JSONB structure — validation shifts to application code
- (-) JSONB fields are opaque to standard BI/reporting tools that expect flat columns
- (-) Complex cross-field queries (e.g., "all soil observations where potassium > 200 ppm") require JSONB extraction syntax
- (-) Storage overhead: JSONB stores key names with every row, unlike columnar storage
- (-) Schema documentation must include JSONB "shape" examples since the DDL alone is incomplete

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ADAPT Standard 1.0 | Grower/Farm/Field/Season relational hierarchy preserved as core tables |
| ISO 28258:2013 | Soil observation parameters stored as JSONB key-value pairs within soil_sample |
| fiboa Specification | Field boundary stored as PostGIS GEOGRAPHY; area_ha as relational column |
| OGC SensorThings API | Sensor readings stored as JSONB observations with typed property names |
| EPPO Codes | Pest/crop identification uses EPPO code as a relational VARCHAR field |
| 4R Nutrient Stewardship | Recommendation JSONB includes right_source, right_rate, right_time, right_place keys |
| RFC 7946 (GeoJSON) | Geometry columns use PostGIS types compatible with GeoJSON export |
| ISO 3166-1/2 | Country/subdivision codes in relational columns on organisation and grower |
| IPM Framework | Recommendation category in JSONB supports cultural/biological/chemical classification |

---

## Core Identity & Hierarchy (Relational)

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    country_code    CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "subscription_tier": "professional",
    --   "default_units": "metric",       -- metric | imperial
    --   "soil_test_parameters": ["pH", "organic_matter_pct", "N", "P", "K", "CEC", "Mg", "Ca", "S"],
    --   "enabled_modules": ["scouting", "prescriptions", "yield_prediction", "chatbot"],
    --   "regulatory_region": "eu"        -- eu, us, br, in, au
    -- }
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
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "locale": "en-US",
    --   "preferred_units": "imperial",
    --   "notification_prefs": {"sms": true, "email": true, "push": false},
    --   "certifications": ["CCA", "4R_NMS"],
    --   "specialties": ["corn", "soybean", "irrigation"]
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE INDEX idx_app_user_org ON app_user(organisation_id);
CREATE INDEX idx_app_user_email ON app_user(email);

CREATE TABLE grower (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    contact_user_id UUID REFERENCES app_user(id),
    country_code    CHAR(2) NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "farm_size_ha": 1200,
    --   "primary_crops": ["ZEAMX", "GLXMA"],
    --   "irrigation_available": true,
    --   "organic_certified": false,
    --   "cooperative_member": "Midwest Grain Co-op"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_grower_org ON grower(organisation_id);

CREATE TABLE farm (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    grower_id       UUID NOT NULL REFERENCES grower(id),
    name            VARCHAR(255) NOT NULL,
    location        GEOGRAPHY(POINT, 4326),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_farm_grower ON farm(grower_id);

CREATE TABLE field (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    farm_id         UUID NOT NULL REFERENCES farm(id),
    name            VARCHAR(255) NOT NULL,
    boundary        GEOGRAPHY(POLYGON, 4326) NOT NULL,
    area_ha         NUMERIC(10,4) NOT NULL,
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties:
    -- {
    --   "soil_type": "silt_loam",
    --   "irrigation_type": "center_pivot",
    --   "drainage_class": "well_drained",
    --   "slope_pct": 2.5,
    --   "elevation_m": 320,
    --   "land_capability_class": "II",
    --   "previous_crops": ["GLXMA", "ZEAMX", "GLXMA"]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_field_farm ON field(farm_id);
CREATE INDEX idx_field_boundary ON field USING GIST(boundary);
CREATE INDEX idx_field_props ON field USING GIN(properties);

CREATE TABLE season (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id),
    crop_code       VARCHAR(20) NOT NULL,
    variety         VARCHAR(100),
    season_year     INTEGER NOT NULL,
    planting_date   DATE,
    harvest_date    DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'planned',
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example properties:
    -- {
    --   "target_yield_t_ha": 12.5,
    --   "seed_rate_seeds_ha": 80000,
    --   "row_spacing_cm": 76,
    --   "tillage_system": "no_till",
    --   "crop_insurance_policy": "MPCI-2026-4421"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_season_field ON season(field_id);
CREATE INDEX idx_season_year ON season(season_year);
```

---

## Soil Data (JSONB Observations)

```sql
-- Soil sample: combines site, profile, and observations into one table
CREATE TABLE soil_sample (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id),
    location        GEOGRAPHY(POINT, 4326),
    sampling_date   DATE NOT NULL,
    sampled_by_id   UUID REFERENCES app_user(id),
    depth_range_cm  VARCHAR(20) NOT NULL DEFAULT '0-15', -- e.g., 0-15, 15-30, 0-30
    lab_reference   VARCHAR(100),
    results         JSONB NOT NULL DEFAULT '{}',
    -- Example results (Iowa corn/soybean field):
    -- {
    --   "pH": 6.4,
    --   "organic_matter_pct": 3.8,
    --   "cec_meq_100g": 18.5,
    --   "nitrogen_ppm": 22,
    --   "phosphorus_ppm": 35,       -- Mehlich-3
    --   "potassium_ppm": 180,       -- Mehlich-3
    --   "calcium_ppm": 2400,
    --   "magnesium_ppm": 320,
    --   "sulfur_ppm": 12,
    --   "zinc_ppm": 2.1,
    --   "manganese_ppm": 28,
    --   "boron_ppm": 0.8,
    --   "iron_ppm": 85,
    --   "texture": {"sand_pct": 28, "silt_pct": 52, "clay_pct": 20, "class": "silt_loam"},
    --   "method": "Mehlich-3",
    --   "lab_name": "A&L Great Lakes"
    -- }
    --
    -- Example results (Maharashtra, India):
    -- {
    --   "pH": 7.8,
    --   "electrical_conductivity_ds_m": 0.42,
    --   "organic_carbon_pct": 0.65,
    --   "available_nitrogen_kg_ha": 245,
    --   "available_phosphorus_kg_ha": 18,
    --   "available_potassium_kg_ha": 320,
    --   "micronutrients": {"zinc_ppm": 0.8, "iron_ppm": 4.2, "copper_ppm": 1.1, "manganese_ppm": 3.5},
    --   "method": "Olsen",
    --   "lab_name": "ICAR Soil Testing Lab"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_soil_sample_field ON soil_sample(field_id);
CREATE INDEX idx_soil_sample_date ON soil_sample(sampling_date);
CREATE INDEX idx_soil_sample_results ON soil_sample USING GIN(results);
```

**Query example — finding fields with low potassium:**
```sql
SELECT f.name, s.sampling_date, (s.results->>'potassium_ppm')::numeric AS k_ppm
FROM soil_sample s
JOIN field f ON f.id = s.field_id
WHERE (s.results->>'potassium_ppm')::numeric < 150
  AND s.sampling_date > now() - interval '2 years'
ORDER BY k_ppm;
```

---

## Sensor & Weather Data

```sql
-- Sensor device: IoT device deployed in a field
CREATE TABLE sensor_device (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id),
    name            VARCHAR(255) NOT NULL,
    device_type     VARCHAR(50) NOT NULL,
    location        GEOGRAPHY(POINT, 4326),
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "manufacturer": "CropX",
    --   "model": "CropX-S1",
    --   "depths_cm": [10, 30, 60],
    --   "connectivity": "LoRaWAN",
    --   "reading_interval_min": 15,
    --   "battery_type": "solar"
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sensor_device_field ON sensor_device(field_id);

-- Sensor reading: time-series data from sensors
CREATE TABLE sensor_reading (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id       UUID NOT NULL REFERENCES sensor_device(id),
    recorded_at     TIMESTAMPTZ NOT NULL,
    readings        JSONB NOT NULL,
    -- Example readings (soil probe):
    -- {
    --   "soil_moisture_pct_10cm": 32.5,
    --   "soil_moisture_pct_30cm": 41.2,
    --   "soil_moisture_pct_60cm": 45.8,
    --   "soil_temp_c_10cm": 18.3,
    --   "soil_temp_c_30cm": 16.1,
    --   "soil_ec_ds_m": 0.45,
    --   "battery_pct": 87
    -- }
    --
    -- Example readings (weather station):
    -- {
    --   "air_temp_c": 28.5,
    --   "humidity_pct": 65,
    --   "wind_speed_ms": 3.2,
    --   "wind_direction_deg": 225,
    --   "rainfall_mm": 0.0,
    --   "solar_radiation_w_m2": 680,
    --   "barometric_pressure_hpa": 1013.2,
    --   "leaf_wetness_min": 0
    -- }
    quality_flag    VARCHAR(20) DEFAULT 'good'
) PARTITION BY RANGE (recorded_at);
CREATE INDEX idx_sensor_reading_device ON sensor_reading(device_id, recorded_at);
CREATE INDEX idx_sensor_reading_data ON sensor_reading USING GIN(readings);

-- Weather daily: aggregated weather for a field location
CREATE TABLE weather_daily (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id),
    observation_date DATE NOT NULL,
    source          VARCHAR(50) NOT NULL,
    data            JSONB NOT NULL,
    -- Example data:
    -- {
    --   "temp_max_c": 32.1,
    --   "temp_min_c": 18.4,
    --   "temp_mean_c": 25.2,
    --   "precipitation_mm": 0.0,
    --   "humidity_mean_pct": 58,
    --   "wind_speed_mean_ms": 4.1,
    --   "solar_radiation_mj_m2": 22.5,
    --   "et0_mm": 5.8,
    --   "gdd_base10": 15.2,
    --   "gdd_cumulative": 1245.8,
    --   "soil_moisture_0_10cm_pct": 28,
    --   "dew_point_c": 16.3
    -- }
    UNIQUE (field_id, observation_date, source)
) PARTITION BY RANGE (observation_date);
CREATE INDEX idx_weather_daily_field ON weather_daily(field_id, observation_date);
```

---

## Satellite & Remote Sensing

```sql
-- Satellite observation: vegetation indices from satellite imagery
CREATE TABLE satellite_observation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    field_id        UUID NOT NULL REFERENCES field(id),
    observation_date DATE NOT NULL,
    source          VARCHAR(50) NOT NULL,
    cloud_cover_pct NUMERIC(5,2),
    indices         JSONB NOT NULL,
    -- Example indices:
    -- {
    --   "ndvi": {"mean": 0.72, "min": 0.45, "max": 0.88, "stddev": 0.08},
    --   "ndre": {"mean": 0.38, "min": 0.22, "max": 0.51},
    --   "evi":  {"mean": 0.55},
    --   "ndwi": {"mean": 0.18},
    --   "savi": {"mean": 0.64},
    --   "lai":  {"mean": 3.2}
    -- }
    zone_map        JSONB,
    -- Example zone_map:
    -- {
    --   "zones": [
    --     {"label": "low",    "ndvi_range": [0.0, 0.5],  "area_ha": 12.3},
    --     {"label": "medium", "ndvi_range": [0.5, 0.7],  "area_ha": 45.6},
    --     {"label": "high",   "ndvi_range": [0.7, 1.0],  "area_ha": 32.1}
    --   ]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (field_id, observation_date, source)
);
CREATE INDEX idx_satellite_obs_field ON satellite_observation(field_id, observation_date);
CREATE INDEX idx_satellite_obs_indices ON satellite_observation USING GIN(indices);
```

---

## Scouting & Field Observations

```sql
-- Scouting visit: a field inspection event with flexible observations
CREATE TABLE scouting_visit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    scout_user_id   UUID NOT NULL REFERENCES app_user(id),
    visit_date      TIMESTAMPTZ NOT NULL,
    location        GEOGRAPHY(POINT, 4326),
    growth_stage    VARCHAR(30),
    observations    JSONB NOT NULL DEFAULT '[]',
    -- Example observations:
    -- [
    --   {
    --     "type": "pest",
    --     "pest_code": "DIABVI",
    --     "common_name": "Western corn rootworm",
    --     "severity": "moderate",
    --     "incidence_pct": 15,
    --     "location": {"lat": 41.8827, "lng": -93.0977},
    --     "notes": "Adult beetles feeding on silks, 2-3 per plant"
    --   },
    --   {
    --     "type": "disease",
    --     "pest_code": "CERCZE",
    --     "common_name": "Gray leaf spot",
    --     "severity": "low",
    --     "incidence_pct": 5,
    --     "affected_leaves": "lower canopy only"
    --   },
    --   {
    --     "type": "nutrient_deficiency",
    --     "nutrient": "nitrogen",
    --     "severity": "moderate",
    --     "symptoms": "lower leaf yellowing, V-shaped chlorosis",
    --     "area_affected_pct": 10
    --   },
    --   {
    --     "type": "general",
    --     "category": "crop_condition",
    --     "notes": "Stand is uneven in low spots, possible compaction"
    --   }
    -- ]
    general_notes   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scouting_season ON scouting_visit(season_id);
CREATE INDEX idx_scouting_observations ON scouting_visit USING GIN(observations);

-- Scouting image: photos with AI analysis results
CREATE TABLE scouting_image (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scouting_visit_id UUID NOT NULL REFERENCES scouting_visit(id),
    storage_key     VARCHAR(500) NOT NULL,
    content_type    VARCHAR(50) NOT NULL DEFAULT 'image/jpeg',
    file_size_bytes BIGINT,
    location        GEOGRAPHY(POINT, 4326),
    ai_results      JSONB NOT NULL DEFAULT '{}',
    -- Example ai_results:
    -- {
    --   "model": "agro-vision-v3",
    --   "model_version": "3.2.1",
    --   "detections": [
    --     {
    --       "label": "gray_leaf_spot",
    --       "pest_code": "CERCZE",
    --       "confidence": 0.92,
    --       "bounding_box": {"x": 0.15, "y": 0.22, "w": 0.35, "h": 0.40}
    --     },
    --     {
    --       "label": "healthy_leaf",
    --       "confidence": 0.98,
    --       "bounding_box": {"x": 0.55, "y": 0.10, "w": 0.30, "h": 0.45}
    --     }
    --   ],
    --   "overall_health": "moderate_stress",
    --   "processed_at": "2026-07-15T14:23:00Z"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_scouting_img_visit ON scouting_image(scouting_visit_id);
```

---

## Recommendations & Prescriptions

```sql
-- Recommendation: advisory with flexible detail structure
CREATE TABLE recommendation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    created_by_id   UUID NOT NULL REFERENCES app_user(id),
    recommendation_type VARCHAR(30) NOT NULL,
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    title           VARCHAR(255) NOT NULL,
    description     TEXT NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details (fertiliser, 4R-aligned):
    -- {
    --   "category": "chemical",
    --   "right_source": "UAN 28-0-0",
    --   "right_rate": 140,
    --   "right_rate_unit": "kg_N/ha",
    --   "right_time": "V6 sidedress",
    --   "right_place": "banded 15cm from row",
    --   "product_name": "UAN 28-0-0",
    --   "cost_estimate": {"value": 85.50, "currency": "USD", "per": "ha"},
    --   "yield_impact_estimate_pct": 12,
    --   "evidence": ["soil_test_N_low", "ndvi_below_average", "weather_favorable"]
    -- }
    --
    -- Example details (pest control, IPM-aligned):
    -- {
    --   "category": "chemical",
    --   "ipm_tier": "chemical",
    --   "target_pest_code": "DIABVI",
    --   "target_pest_name": "Western corn rootworm",
    --   "threshold_reached": true,
    --   "economic_threshold": "2 beetles per plant at silking",
    --   "observed_level": "2-3 beetles per plant",
    --   "treatment_options": [
    --     {"product": "Warrior II", "rate": "0.12 L/ha", "rei_hours": 24, "phi_days": 21},
    --     {"product": "Brigade 2EC", "rate": "0.15 L/ha", "rei_hours": 12, "phi_days": 30}
    --   ],
    --   "cultural_alternatives": ["Crop rotation to soybean next season"]
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    issued_at       TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    grower_response VARCHAR(20),
    grower_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_recommendation_season ON recommendation(season_id);
CREATE INDEX idx_recommendation_type ON recommendation(recommendation_type);
CREATE INDEX idx_recommendation_status ON recommendation(status);
CREATE INDEX idx_recommendation_details ON recommendation USING GIN(details);

-- Prescription: variable-rate application map
CREATE TABLE prescription (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recommendation_id UUID REFERENCES recommendation(id),
    field_id        UUID NOT NULL REFERENCES field(id),
    season_id       UUID NOT NULL REFERENCES season(id),
    prescription_type VARCHAR(30) NOT NULL,
    product_name    VARCHAR(255),
    rate_unit       VARCHAR(20) NOT NULL,
    zones           JSONB NOT NULL,
    -- Example zones:
    -- [
    --   {
    --     "geometry": {"type": "Polygon", "coordinates": [[[...]], ...]},
    --     "rate": 160,
    --     "area_ha": 12.3,
    --     "label": "high_yield_zone",
    --     "basis": "high_ndvi_high_yield_history"
    --   },
    --   {
    --     "geometry": {"type": "Polygon", "coordinates": [[[...]], ...]},
    --     "rate": 120,
    --     "area_ha": 8.7,
    --     "label": "low_yield_zone",
    --     "basis": "low_ndvi_sandy_soil"
    --   }
    -- ]
    summary         JSONB NOT NULL DEFAULT '{}',
    -- Example summary:
    -- {
    --   "total_area_ha": 90.0,
    --   "mean_rate": 142,
    --   "min_rate": 100,
    --   "max_rate": 180,
    --   "total_product_kg": 12780,
    --   "zone_count": 5
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_prescription_field ON prescription(field_id);
CREATE INDEX idx_prescription_season ON prescription(season_id);
```

---

## Field Operations

```sql
-- Field operation: recorded agronomic activities
CREATE TABLE field_operation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    operation_type  VARCHAR(30) NOT NULL,
    operation_date  DATE NOT NULL,
    prescription_id UUID REFERENCES prescription(id),
    operator_id     UUID REFERENCES app_user(id),
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example details (fertiliser application):
    -- {
    --   "product": "UAN 28-0-0",
    --   "rate_applied": 140,
    --   "rate_unit": "kg_N/ha",
    --   "area_treated_ha": 45.2,
    --   "application_method": "sidedress_banded",
    --   "equipment": "John Deere 4730 sprayer",
    --   "cost_per_ha": 85.50,
    --   "total_cost": 3864.60,
    --   "currency": "USD",
    --   "weather_at_application": {"temp_c": 24, "wind_ms": 2.1, "humidity_pct": 55}
    -- }
    --
    -- Example details (harvest):
    -- {
    --   "yield_t_ha": 12.8,
    --   "moisture_pct": 15.5,
    --   "test_weight_kg_hl": 72.3,
    --   "protein_pct": 8.2,
    --   "area_harvested_ha": 45.2,
    --   "total_production_t": 578.6,
    --   "equipment": "John Deere S780 combine",
    --   "quality_grade": "US No. 2 Yellow"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_field_op_season ON field_operation(season_id);
CREATE INDEX idx_field_op_type ON field_operation(operation_type);
CREATE INDEX idx_field_op_date ON field_operation(operation_date);
CREATE INDEX idx_field_op_details ON field_operation USING GIN(details);
```

---

## Yield Prediction & AI

```sql
-- Yield prediction: AI-generated forecasts with flexible inputs/outputs
CREATE TABLE yield_prediction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    season_id       UUID NOT NULL REFERENCES season(id),
    prediction_date DATE NOT NULL,
    predicted_yield NUMERIC(10,2) NOT NULL,
    yield_unit      VARCHAR(20) NOT NULL,
    model_info      JSONB NOT NULL,
    -- Example model_info:
    -- {
    --   "model_name": "yield-xgboost-v4",
    --   "model_version": "4.1.2",
    --   "confidence_interval": {"low": 11.2, "high": 14.1, "pct": 90},
    --   "input_features": {
    --     "weather_gdd_cumulative": 1245,
    --     "ndvi_mean_30d": 0.72,
    --     "soil_om_pct": 3.8,
    --     "soil_k_ppm": 180,
    --     "precipitation_season_mm": 420,
    --     "previous_yield_t_ha": 11.5
    --   },
    --   "feature_importance": {
    --     "weather_gdd_cumulative": 0.28,
    --     "ndvi_mean_30d": 0.22,
    --     "precipitation_season_mm": 0.18,
    --     "soil_om_pct": 0.12
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_yield_pred_season ON yield_prediction(season_id);
```

---

## Chat & Alerts

```sql
-- Chat conversation with context
CREATE TABLE chat_conversation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    season_id       UUID REFERENCES season(id),
    title           VARCHAR(255),
    context         JSONB NOT NULL DEFAULT '{}',
    -- Example context:
    -- {
    --   "field_name": "North 80",
    --   "crop": "corn",
    --   "growth_stage": "V12",
    --   "recent_soil_test_id": "uuid...",
    --   "active_recommendations": ["uuid1", "uuid2"]
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_chat_conv_user ON chat_conversation(user_id);

CREATE TABLE chat_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES chat_conversation(id),
    role            VARCHAR(20) NOT NULL,
    content         TEXT NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example metadata:
    -- {
    --   "model": "claude-4-sonnet",
    --   "token_count": 342,
    --   "tools_used": ["soil_lookup", "weather_forecast"],
    --   "sources_cited": ["soil_sample:uuid...", "weather_daily:uuid..."]
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_chat_msg_conv ON chat_message(conversation_id);

-- Alert
CREATE TABLE alert (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    season_id       UUID REFERENCES season(id),
    alert_type      VARCHAR(30) NOT NULL,
    channel         VARCHAR(20) NOT NULL,
    title           VARCHAR(255) NOT NULL,
    body            TEXT NOT NULL,
    delivery        JSONB NOT NULL DEFAULT '{}',
    -- Example delivery:
    -- {
    --   "phone": "+1-515-555-0123",
    --   "sent_at": "2026-07-15T06:00:00Z",
    --   "delivered_at": "2026-07-15T06:00:03Z",
    --   "provider": "twilio",
    --   "message_sid": "SM..."
    -- }
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_user ON alert(user_id);
CREATE INDEX idx_alert_status ON alert(status);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    table_name      VARCHAR(100) NOT NULL,
    record_id       UUID NOT NULL,
    action          VARCHAR(10) NOT NULL,
    user_id         UUID REFERENCES app_user(id),
    changes         JSONB,
    -- Example changes (UPDATE):
    -- {
    --   "old": {"status": "draft", "updated_at": "2026-07-14T10:00:00Z"},
    --   "new": {"status": "issued", "updated_at": "2026-07-15T08:30:00Z", "issued_at": "2026-07-15T08:30:00Z"}
    -- }
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);
CREATE INDEX idx_audit_log_table ON audit_log(table_name, record_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 2 | organisation, app_user |
| Grower/Farm/Field/Season | 4 | Relational hierarchy with JSONB properties |
| Soil Data | 1 | Single table with JSONB results (replaces 4 tables) |
| Sensors & Weather | 3 | device, reading (JSONB), weather_daily (JSONB) |
| Satellite | 1 | Indices and zone maps in JSONB |
| Scouting | 2 | Visit with JSONB observations, image with JSONB AI results |
| Recommendations & Prescriptions | 2 | Flexible JSONB details and zones |
| Field Operations | 1 | JSONB details per operation type |
| Yield Prediction | 1 | JSONB model info and features |
| Chat & Alerts | 3 | Conversation, message, alert |
| Audit | 1 | JSONB change tracking |
| **Total** | **21** | ~43% fewer tables than fully normalized |

---

## Key Design Decisions

1. **JSONB for domain-variable attributes, relational for structural invariants** — the grower/farm/field/season hierarchy is always relational because it defines the fundamental navigation and access control paths. Everything that varies by crop, region, or jurisdiction goes into JSONB.

2. **GIN indexes on all JSONB columns** — enables efficient containment queries (`@>`), key-exists checks (`?`), and path queries (`->>`). The GIN index on `soil_sample.results` supports queries like `WHERE results @> '{"method": "Mehlich-3"}'` without full table scans.

3. **Single soil_sample table replaces four normalized tables** — the ISO 28258 site/profile/horizon/observation hierarchy is collapsed into one row per sample with depth_range_cm and a JSONB results column. This works well for the common case (one sample at one depth) and still supports multi-depth by inserting multiple rows.

4. **Prescription zones stored as JSONB array with GeoJSON geometry** — rather than a separate junction table, zones live inside the prescription. This simplifies reads (one query gets the full prescription) and aligns with how ISOXML exports work (one document per prescription).

5. **Organisation-level settings JSONB** — the `settings` field on organisation controls which soil parameters to display, which modules are enabled, and which regulatory region applies. This replaces a constellation of feature-flag tables.

6. **Inline JSONB examples document the schema** — since JSONB columns have no DDL-level structure, comprehensive commented examples in the CREATE TABLE statements serve as the schema documentation. Application-level JSON Schema validation should enforce these structures.

7. **Partitioned time-series tables** — `sensor_reading`, `weather_daily`, and `audit_log` are partitioned by time for the same performance reasons as the normalized model.

8. **EPPO codes remain as VARCHAR fields within JSONB** — pest and crop identification uses EPPO codes stored as string values inside JSONB observations, enabling cross-reference without a mandatory foreign key to a reference table. A crop/pest lookup table can optionally exist for autocomplete but isn't required for data integrity.

9. **AI results embedded in scouting_image** — detection results, model version, and bounding boxes are stored as JSONB on the image row rather than in a separate table, keeping the scouting workflow's read path to a single JOIN.

10. **Trade-off acknowledgement: validation shifts to application layer** — the major cost of this design is that PostgreSQL cannot enforce the shape of JSONB data. The application must validate JSONB payloads against JSON Schema definitions before insertion. This is acceptable for a single-codebase SaaS product but becomes a liability if third parties write directly to the database.
