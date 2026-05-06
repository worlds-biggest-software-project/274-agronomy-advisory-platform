# Agronomy Advisory Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source agronomy advisory platform delivering soil health recommendations, crop disease identification, and yield optimization for growers and advisors.

The Agronomy Advisory Platform synthesizes weather, soil, satellite, and field-level scouting data into actionable agronomic recommendations. It is built for commercial crop advisors, growers, cooperatives, and extension services who need objective, vendor-neutral advice across irrigation, fertility, pest and disease management, and yield planning.

---

## Why Agronomy Advisory Platform?

- Incumbents like Climate FieldView (Bayer), Syngenta Cropwise, and Granular (Corteva) are tied to their parent company's input and seed products, limiting recommendation objectivity.
- Hardware-dependent platforms such as CropX and Trimble Ag Software demand significant upfront investment and are poorly suited to smallholder, specialty crop, or developing-region markets.
- Satellite-only tools like Farmonaut are affordable but limited by 5-7 day revisit cycles, cloud cover, and shallow integration with farm operations.
- US row-crop centric platforms (Climate FieldView, Granular) leave specialty crops, horticulture, and non-US regions underserved.
- Lab-based soil biology services (Trace Genomics) carry 1-2 week turnaround and high per-test costs ($250-500+), preventing real-time advisory.

---

## Key Features

### Crop Advisory Core

- Weather-driven advisory for irrigation, fertility, pest, and disease decisions
- Multi-field and multi-crop management with region-specific models
- Mobile and web interfaces for field access
- Historical recommendation tracking and trend analysis
- Report generation for decision documentation

### Disease, Pest, and Crop Health

- Disease and pest identification from smartphone-captured field images
- Weather-driven pest and disease risk forecasting
- Treatment recommendations grounded in IPM principles
- Crop health monitoring using satellite-derived indices (NDVI, NDRE, SAVI)

### Soil and Yield Intelligence

- Soil variability and health assessment
- Yield prediction and forecasting integrating weather, soil, and crop health
- Variable-rate application prescription generation
- Multi-year trend and performance benchmarking

### Advisor and Operations Workflow

- Agronomist collaboration and client management workflows
- Field scouting data collection and decision documentation
- Financial integration and ROI analysis of agronomic decisions
- User management and access control

### Accessibility and Reach

- SMS and voice interface for feature phones
- Smallholder-accessible deployment without mandatory hardware
- Regional crop database with localized recommendations

---

## AI-Native Advantage

Computer vision models identify diseases and pests from smartphone photos, replacing scheduled scouting visits with instant in-field diagnosis. Yield prediction models combine historical yield maps, soil variability layers, weather forecasts, and current crop health indices to produce field-by-field harvest estimates weeks ahead. An LLM-powered advisory chatbot translates soil tests and scouting observations into plain-language recommendations, while adaptive variable-rate prescriptions learn from grower acceptance and rejection patterns to personalise advice over time.

---

## Tech Stack & Deployment

The platform is designed around open agronomic and geospatial standards including 4R Nutrient Stewardship, IPM frameworks, FAO Global Soil Partnership data conventions, ISO 11074 / ISO 10381 soil quality standards, IPPC pest reporting standards, and OGC spatial interoperability standards. Expected integrations include satellite providers (Sentinel, Planet, Landsat), weather APIs, soil sensor networks, drone imagery, farm equipment telemetry, and existing farm management systems. Deployment targets include cloud-hosted SaaS for advisors and growers, with mobile-first access and SMS/voice channels for low-connectivity regions.

---

## Market Context

The global precision agriculture market was valued at approximately USD 12 billion in 2025 and is projected to exceed USD 25 billion by 2030 at around 13% CAGR (research.md). Incumbent pricing ranges from a few dollars per month for satellite-only tools (Farmonaut) through per-acre fees of $5.99-$15/acre/yr (Climate FieldView) to custom enterprise quotes of $10,000-$100,000+/yr (CropX, Trimble, Granular). Primary buyers are commercial crop advisors and agronomists, large-scale row crop and specialty growers, cooperative and input-retailer agronomy teams, crop insurance underwriters, and food company procurement teams requiring sustainability data.

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
