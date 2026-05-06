# Standards & API Reference

> Project: Agronomy Advisory Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 11783 (ISOBUS)**
- URL: https://www.iso.org/standard/57556.html
- Serial control and communications data network for tractors and agricultural machinery, comprising 14 parts. ISO 11783-10 covers Task Controller and Management Information System (MIS) data interchange, defining the ISOXML format for exchanging prescriptions, field operations, and agronomic data between farm equipment and farm management software. Fundamental for variable-rate application integration in any agronomy advisory platform.

**ISO 11787:1995 — Agricultural Data Interchange Syntax (ADIS)**
- URL: https://www.iso.org/standard/3247.html
- Specifies an Agricultural Data Interchange Syntax (ADIS) to enable electronic data exchange between on-farm process computers (stationary and mobile machinery) and management computers. While not intended for real-time exchange, ADIS defines a common structure for field records, soil measurements, and agronomic observations useful for data import/export pipelines.

**ISO 10381 — Soil Quality Sampling**
- URL: https://www.iso.org/search.html?q=10381
- A multi-part ISO standard defining guidance and requirements for soil sampling procedures, including soil sampling programmes, sampling techniques, and packaging and transport. Relevant as the normative basis for soil health data collection integrated into agronomy advisory platforms.

**ISO 19115-1:2014 — Geographic Information Metadata**
- URL: https://www.iso.org/standard/53798.html
- Defines the schema for describing geographic information and services by means of metadata, covering identification, extent, quality, spatial reference, and distribution properties. Applies to field boundary datasets, soil variability maps, yield maps, and any geospatial layer used in agronomy advisory platforms.

**ISO 28258:2013 — Digital Exchange of Soil-Related Data**
- URL: https://www.iso.org/standard/44859.html
- Defines a conceptual schema for soil data with emphasis on spatial and temporal aspects of soil observations. Provides the data model vocabulary for standardised representation of soil measurements (texture, pH, nutrient levels, moisture) in agronomy systems.

### W3C & IETF Standards

**OGC API Features (OGC 17-069r4)**
- URL: https://www.ogc.org/standards/ogcapi-features/
- RESTful API standard for discovering, querying, and retrieving geospatial feature data over the web. The primary standard for exposing field boundaries and farm spatial data via web APIs. Supersedes WFS and is built on OpenAPI 3.0, GeoJSON, and HTTP.

**OGC SensorThings API (OGC 15-078r6)**
- URL: https://www.ogc.org/standards/sensorthings
- Open, geospatial-enabled standard for connecting IoT devices, sensor observations, and applications over the web using REST and MQTT. Directly applicable to soil sensor networks, weather station data ingestion, and real-time field monitoring in an agronomy advisory platform.

**OGC Web Map Service (WMS) / Web Coverage Service (WCS)**
- URL: https://www.ogc.org/standards/wms
- Standards for serving georeferenced map images and gridded coverage data over the web. WMS and WCS are supported by the Sentinel Hub API and other satellite imagery providers as delivery formats for NDVI and other vegetation indices.

**RFC 7946 — GeoJSON Format**
- URL: https://tools.ietf.org/html/rfc7946
- IETF standard defining the GeoJSON format for encoding geographic data structures using JSON. GeoJSON is the de facto format for field boundary exchange, prescription zone geometry, and spatial data interchange in modern agricultural APIs. The fiboa (Field Boundaries for Agriculture) specification builds directly on RFC 7946.

**RFC 6749 / RFC 6750 — OAuth 2.0 & Bearer Token**
- URL: https://tools.ietf.org/html/rfc6749
- The industry-standard authorisation framework used by all major agricultural APIs (John Deere, Trimble, Leaf, Sentinel Hub, Tomorrow.io) for delegated access to farmer data. Any agronomy advisory platform must implement OAuth 2.0 to participate in the connected agriculture data ecosystem.

### Data Model & API Specifications

**ADAPT Standard 1.0 (AgGateway)**
- URL: https://adaptstandard.org/
- The Agriculture Data Application Programming Toolkit, released as version 1.0 in June 2024. Provides an Agricultural Application Data Model, a common API, and open-source data conversion plugins for interoperability between farm management systems, agronomic platforms, and precision ag hardware. Defines standardised schemas for prescriptions, field operations (planting, application, harvest), soil observations, and grower/farm/field hierarchies. The key interoperability standard for any agronomy advisory platform targeting the precision agriculture ecosystem.

**ISOXML (part of ISO 11783-10)**
- URL: https://www.isobus.net/
- XML-based data format for exchanging task data between Task Controllers (on machinery) and Farm Management Information Systems (FMIS). ISOXML encodes variable-rate application prescriptions, field boundaries, product applications, and harvest data. Supported by the AEF (Agricultural Industry Electronics Foundation) conformance testing programme.

**fiboa — Field Boundaries for Agriculture**
- URL: https://fiboa.org/ and https://github.com/fiboa/specification
- An open specification for representing field boundary data in GeoJSON and GeoParquet formats with a standardised attribute schema and optional extensions. Provides a lightweight, interoperable data model for field polygon exchange that an agronomy advisory platform can adopt as its canonical field boundary format.

**OpenAPI 3.1 Specification**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry standard for describing RESTful APIs using a machine-readable format. All modern agricultural APIs (Leaf, Trimble, John Deere, Sentinel Hub) provide OpenAPI-compliant specifications. An agronomy advisory platform should publish its own OpenAPI 3.1 spec to enable third-party integration and SDK generation.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) / OpenID Connect**
- URL: https://openid.net/connect/
- OAuth 2.0 is mandatory for delegated access to farmer data across connected agricultural platforms. OpenID Connect (OIDC) adds identity authentication on top of OAuth 2.0, enabling SSO for agronomists managing multiple farm accounts. Required for John Deere Operations Center, Trimble, and Climate FieldView integrations.

**EU Data Act (Regulation 2023/2854)**
- URL: https://digital-strategy.ec.europa.eu/en/policies/data-act
- Applicable from 12 September 2025, the EU Data Act gives farmers (as business users) the right to access data about the performance of their connected agricultural equipment and share it with third-party service providers. An agronomy advisory platform operating in the EU must implement compliant data portability mechanisms and cannot lock in farm data without farmer consent.

**EU GDPR (Regulation 2016/679)**
- URL: https://gdpr.eu/
- Governs the processing of personal data including farmer identity, farm location coordinates (which may qualify as personal data when linked to an identifiable individual), and agronomic records. GDPR applies fully alongside the EU Data Act, with GDPR taking precedence in case of conflict on personal data matters.

**EU Code of Conduct for Agricultural Data Sharing (EUCC)**
- URL: https://copa-cogeca.eu/
- A voluntary industry code of conduct published in 2018 establishing contractual rights and responsibilities for agricultural data sharing between farmers, machinery manufacturers, and software providers. An important reference for data ownership clauses and consent mechanisms in farmer-facing agronomy platforms.

**OWASP API Security Top 10**
- URL: https://owasp.org/www-project-api-security/
- The authoritative checklist of the most critical API security risks, including broken object level authorisation, lack of rate limiting, and mass assignment vulnerabilities. Agronomy platforms exposing field data, sensor readings, and agronomic recommendations via REST APIs should apply OWASP API Security controls during design and testing.

### MCP Server Specifications

The Model Context Protocol (MCP), developed by Anthropic and now a de facto standard for connecting LLMs to external tools and data sources, is highly relevant to an AI-native agronomy advisory platform. As of 2026, MCP is the primary integration surface for agentic AI assistants.

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/ and https://github.com/modelcontextprotocol/modelcontextprotocol
- MCP defines a standard protocol for LLM clients to discover and call tools, access resources, and receive prompts from server implementations. An agronomy advisory platform could publish an MCP server exposing tools for: querying crop health indices, retrieving field-level recommendations, submitting pest/disease observations, fetching weather forecasts, and triggering agronomic advisory workflows. The Tomorrow.io weather API already ships a native MCP server (as of Spring 2026), providing a model for weather data integration.

---

## Similar Products — Developer Documentation & APIs

### John Deere Operations Center API
- **Description:** John Deere's cloud platform for farm data management, providing APIs for field boundaries, farm/field hierarchies, field operations (planting, application, harvest), equipment telemetry, and agronomic files.
- **API Documentation:** https://developer.deere.com/dev-docs/field-operations
- **SDKs/Libraries:** Unofficial TypeScript SDK (https://github.com/ProductOfAmerica/deere-sdk); accessible via Leaf's unified API (https://withleaf.io/providers/johndeere)
- **Developer Guide:** https://developer.deere.com/
- **Standards:** REST/JSON, OAuth 2.0, GeoJSON for boundaries
- **Authentication:** OAuth 2.0 (developer registration required)

### Trimble Agriculture Cloud API
- **Description:** Precision agriculture data platform with APIs for field boundaries, prescription files, as-applied maps, yield data, and farm management records. Supports integration with Trimble in-field displays and third-party FMIs.
- **API Documentation:** https://agdeveloper.trimble.com/api-docs and https://developer.trimble.com/docs/ag/ag-api/
- **SDKs/Libraries:** Trimble Ag Developer Network SDK resources (https://agdeveloper.trimble.com/)
- **Developer Guide:** https://agdeveloper.trimble.com/trimble-ag-software-integration/
- **Standards:** REST/JSON, OpenAPI, OAuth 2.0, ISOXML for prescription exchange
- **Authentication:** OAuth 2.0 (Ag Developer Network registration required)

### Leaf Agriculture Unified Farm Data API
- **Description:** A unified API that aggregates and normalises farm data from all major agricultural platforms (John Deere, Climate FieldView, CNHi, Trimble, AgLeader). Provides standardised endpoints for field boundaries, field operations (planting, application, harvest, tillage), satellite imagery, and agronomic prescriptions from a single integration point.
- **API Documentation:** https://docs.withleaf.io/docs/introduction/
- **SDKs/Libraries:** cURL, Node.js, Python examples; Postman collection (https://github.com/Leaf-Agriculture/Leaf-API-Postman-Collection)
- **Developer Guide:** https://leaf-agriculture.github.io/docs/docs/index.html
- **Standards:** REST/JSON, GeoJSON (MultiPolygon for field boundaries), OAuth 2.0
- **Authentication:** Leaf API key + provider-specific OAuth 2.0 tokens

### Sentinel Hub / Copernicus Data Space API
- **Description:** RESTful API providing access to satellite imagery archives including Sentinel-2, Landsat, and PlanetScope. Supports on-the-fly calculation of vegetation indices (NDVI, NDRE, EVI, NDWI) for crop health monitoring, batch processing for large area analysis, and statistical time-series for field-level aggregation.
- **API Documentation:** https://docs.sentinel-hub.com/api/latest/ and https://documentation.dataspace.copernicus.eu/
- **SDKs/Libraries:** sentinelhub-py Python library (https://sentinelhub-py.readthedocs.io/)
- **Developer Guide:** https://documentation.dataspace.copernicus.eu/notebook-samples/sentinelhub/introduction_to_SH_APIs.html
- **Standards:** REST/JSON, OGC WMS/WCS/WMTS, OGC API Features (Catalog API)
- **Authentication:** OAuth 2.0 (Copernicus Data Space account)

### Planet Labs Data & Analytics API
- **Description:** Provides daily satellite imagery (PlanetScope) and Planetary Variables (crop biomass, soil water content) via REST API, enabling field-level NDVI and spectral index calculation for crop monitoring at 3m resolution with near-daily revisit frequency.
- **API Documentation:** https://docs.planet.com/develop/apis/
- **SDKs/Libraries:** Python notebooks (https://github.com/planetlabs/notebooks); integration via Leaf API
- **Developer Guide:** https://docs.planet.com/
- **Standards:** REST/JSON, GeoJSON, OGC-compatible delivery formats
- **Authentication:** API Key

### Agromonitoring (OpenWeather Agro API)
- **Description:** Agricultural monitoring platform from OpenWeather providing soil data (temperature and moisture at surface and 10cm/40cm depths), satellite vegetation indices (NDVI, EVI), accumulated temperature and precipitation, and weather forecasts specifically structured for agronomy decision-making via REST API.
- **API Documentation:** https://agromonitoring.com/api
- **SDKs/Libraries:** Community integrations (https://github.com/GitHub4Eddy/agro_monitor)
- **Developer Guide:** https://agromonitoring.com/api/get
- **Standards:** REST/JSON; polygon-based field definition
- **Authentication:** API Key (appid parameter)

### Farmonaut Satellite & Weather API
- **Description:** Agronomy-focused API providing real-time multispectral satellite imagery, NDVI and NDWI field health indices, weather forecasts, crop stress alerts, and yield forecasting for fields defined as polygons. Affordable subscription tiers targeting smaller operators and developers in emerging markets.
- **API Documentation:** https://farmonaut.com/farmonaut-satellite-weather-api-developer-docs
- **SDKs/Libraries:** REST API with JSON responses; mobile and web SDKs available
- **Developer Guide:** https://sat.farmonaut.com/api
- **Standards:** REST/JSON; structured JSON responses compatible with common frontend frameworks
- **Authentication:** API Key

### Tomorrow.io Weather API
- **Description:** High-resolution weather intelligence API providing 60+ data layers including temperature, precipitation, humidity, wind, soil moisture, evapotranspiration (ET₀), and pollen. Supports agricultural planning with 14-day forecasts and historical data. As of Spring 2026, also provides a native MCP server for agentic AI integration.
- **API Documentation:** https://docs.tomorrow.io/
- **SDKs/Libraries:** REST API; GitHub organisation (https://github.com/tomorrow-io-API)
- **Developer Guide:** https://support.tomorrow.io/hc/en-us/articles/31227543026708-How-to-Use-the-Tomorrow-io-API
- **Standards:** REST/JSON, OpenAPI, native MCP server
- **Authentication:** API Key

### EPPO Global Database
- **Description:** European and Mediterranean Plant Protection Organization's authoritative database of 97,800+ agricultural pest and disease species, with regulatory categorisation, host plant lists, geographic distribution, and phytosanitary standards for each pest. Provides the reference data model for plant pest identification and advisory systems.
- **API Documentation:** https://gd.eppo.int/ (structured data export; limited API access — primarily file-based)
- **SDKs/Libraries:** BASF internal EPPO ontology (public reference); community tooling
- **Developer Guide:** https://www.eppo.int/RESOURCES/eppo_databases/global_database
- **Standards:** Controlled vocabulary (EPPO codes); PP1 and PP2 advisory standards
- **Authentication:** Registration for data access; public web interface freely available

### Agworld API
- **Description:** Agworld is an agronomist collaboration platform providing APIs for farm field management, agronomic recommendations, scouting observations, crop plans, and spray applications. Designed for the agronomist-grower workflow with multi-stakeholder data sharing.
- **API Documentation:** https://us.agworld.co/user_api/v1/docs
- **SDKs/Libraries:** REST API; JSON
- **Developer Guide:** Agworld developer documentation (https://us.agworld.co/user_api/v1/docs)
- **Standards:** REST/JSON, OpenAPI
- **Authentication:** API token (user-scoped)

---

## Notes

**Emerging Interoperability Standards**

The agricultural data interoperability landscape is fragmented, with ADAPT Standard 1.0 (2024) and the fiboa specification representing the most recent attempts to create common open schemas. Integration with proprietary platforms (John Deere, Trimble, Climate FieldView) typically requires going through aggregation layers such as Leaf Agriculture's unified API to avoid maintaining individual OAuth integrations with each provider.

**EU Data Act Compliance (from September 2025)**

The EU Data Act creates legally enforceable data portability rights for farmers using connected agricultural equipment. Any agronomy advisory platform operating in Europe must implement data export and portability features to remain compliant. This regulation may accelerate industry adoption of ADAPT and fiboa as common exchange standards.

**MCP for Agronomy AI Agents**

No domain-specific MCP servers for agronomy currently appear in the public registry as of May 2026. This represents an early-mover opportunity: building MCP servers that expose crop advisory tools, pest/disease lookup (from EPPO), field satellite health data (from Sentinel Hub), and weather-driven recommendations would enable any MCP-compatible AI assistant to function as an agronomic advisor without bespoke integrations.

**Soil Carbon & Sustainability Data**

There is no established open standard for soil carbon sequestration quantification or carbon credit documentation in precision agriculture as of May 2026. The Verra Verified Carbon Standard (VCS) and Gold Standard provide project-level methodologies, but field-level data APIs for carbon accounting remain proprietary (e.g., Indigo Ag, Corteva Carbon). This is an underserved integration opportunity.
