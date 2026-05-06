# Standards & API Reference

> Project: Marketing Attribution Platform · Generated: 2026-05-06

---

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 27001 — Information Security Management**
- URL: https://www.iso.org/isoiec-27001-information-security.html
- Relevant as the baseline information-security standard for any platform processing first-party customer event data and conversion records. SOC 2 Type II (common in this space; Rockerbox is certified) is the US equivalent certification built on similar controls.

**ISO/IEC 27018 — Protection of Personally Identifiable Information (PII) in Public Clouds**
- URL: https://www.iso.org/standard/76559.html
- Directly applicable to cloud-hosted attribution platforms processing customer identity data for cross-device matching and identity resolution.

**ISO 31000 — Risk Management**
- URL: https://www.iso.org/iso-31000-risk-management.html
- Applicable to model risk management: attribution platforms must document the assumptions, limitations, and failure modes of their probabilistic attribution models to satisfy enterprise risk governance requirements.

---

### W3C & IETF Standards

**W3C Attribution Reporting API (Privacy Sandbox)**
- URL: https://wicg.github.io/attribution-reporting-api/
- The browser-native standard for privacy-preserving conversion measurement, replacing third-party cookie-based attribution. Provides two report types: event-level reports (ad click/view → conversion with limited granularity) and summary reports (aggregate conversions with differential privacy noise). Any attribution platform building browser-side measurement must implement or interpret these reports.

**W3C Privacy Principles**
- URL: https://www.w3.org/TR/privacy-principles/
- Foundational guidance underpinning all W3C work on privacy-preserving measurement; relevant to architecture decisions around identity resolution and data minimisation.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The authentication standard used by Google Ads API, Meta Marketing API, and TikTok Ads API for secure server-to-server event submission. All Conversions API integrations require OAuth 2.0 credential management.

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Used in authentication flows for API-to-API data exchange between attribution platforms and ad networks.

**RFC 9110 — HTTP Semantics**
- URL: https://datatracker.ietf.org/doc/html/rfc9110
- Base HTTP standard applicable to all REST API integrations with ad platform conversion APIs.

**RFC 8259 — JSON Data Interchange Format**
- URL: https://datatracker.ietf.org/doc/html/rfc8259
- Conversion event payloads submitted to Meta CAPI, Google Enhanced Conversions, and TikTok Events API are all JSON-encoded; strict adherence to RFC 8259 is required.

---

### Data Model & API Specifications

**OpenAPI 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The dominant specification format for REST API documentation. Attribution platforms exposing developer APIs should provide an OpenAPI 3.1 spec for their event ingestion, attribution query, and report export endpoints.

**CloudEvents 1.0 (CNCF)**
- URL: https://cloudevents.io/
- Vendor-neutral specification for describing event data in a common way. Applicable to the design of the attribution platform's server-side event ingestion API; using CloudEvents format enables interoperability with existing CDPs and event streaming pipelines.

**dbt Semantic Layer**
- URL: https://docs.getdbt.com/docs/use-dbt-semantic-layer/dbt-semantic-layer
- De-facto standard for defining marketing metrics (spend, conversions, ROAS, attribution windows) consistently across a data warehouse. Attribution platforms targeting data-warehouse-centric teams should publish dbt packages or semantic layer integrations.

**Apache Parquet**
- URL: https://parquet.apache.org/
- Column-oriented storage format used by Snowflake, BigQuery, and other analytical warehouses for bulk attribution data export. Attribution results at scale are most efficiently exported in Parquet format.

**Snowplow Open Data Protocol**
- URL: https://docs.snowplow.io/docs/understanding-tracking-design/
- Open-source event tracking standard with a structured JSON schema registry; used as a foundation for first-party event collection pipelines feeding attribution systems.

---

### Security & Authentication Standards

**OAuth 2.0 with PKCE (RFC 7636)**
- URL: https://datatracker.ietf.org/doc/html/rfc7636
- Required by Google Ads API and Meta Marketing API for secure token exchange in server-side integration scenarios.

**OpenID Connect 1.0**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0; relevant for attribution platforms offering SSO and enterprise identity federation.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Baseline for securing the attribution platform's own APIs (event ingestion, attribution query, admin). Most relevant risks: broken object-level authorisation (BOLA) when scoping attribution data by brand/workspace, and excessive data exposure in conversion event payloads.

**NIST SP 800-188 — De-Identification of Government Datasets**
- URL: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-188.pdf
- Guidance on de-identification techniques applicable to hashing PII (email, phone) before transmission to ad platforms via Conversions APIs.

**GDPR (EU 2016/679)**
- URL: https://gdpr-info.eu/
- Requires opt-in consent before processing personal data for marketing attribution in the EU/EEA. Consent Mode v2 signals (ad_storage, ad_personalization, analytics_storage, functionality_storage) must be respected. Modelled/probabilistic attribution for non-consent users (as implemented by SegmentStream) must be demonstrably privacy-preserving.

**CCPA / CPRA (California)**
- URL: https://oag.ca.gov/privacy/ccpa
- Opt-out model for sale or sharing of personal data; Global Privacy Control (GPC) signal must be honoured by attribution platforms processing California consumer data. As of January 2026, new processing activities require Data Privacy Impact Assessments (DPIAs).

**Google Consent Mode v2**
- URL: https://support.google.com/analytics/answer/9976101
- Mandatory for any platform managing Google Ads or Analytics for EEA/UK traffic since 2024. Platforms must transmit the four consent signals to Google; modelled conversions fill the gap for non-consented users.

---

### IAB Tech Lab & MRC Measurement Standards

**IAB Tech Lab Open Measurement SDK (OM SDK)**
- URL: https://iabtechlab.com/standards/open-measurement-sdk/
- Standardises third-party collection of viewability and verification metrics across mobile in-app and web environments. Attribution platforms incorporating impression-level data (view-through attribution) should consume OM SDK signals.

**MRC Viewable Impression Guidelines**
- URL: https://www.iab.com/guidelines/mrc-viewable-impression-guidelines/
- Defines the minimum viewability threshold (50% of pixels in view for at least 1 second for display; 50% of pixels for 2 seconds for video) that qualifies an impression for attribution purposes.

**IAB/MRC Attention Measurement Guidelines (November 2025)**
- URL: https://www.iab.com/wp-content/uploads/2025/11/IAB_MRC_Attention_Measurement_Guidelines_November_2025.pdf
- Emerging standard for measuring attention as a signal beyond raw viewability. Attribution platforms incorporating attention weighting in their models should reference this framework.

**IAB Tech Lab Attribution Data Matching Protocol (ADMaP)**
- URL: https://iabtechlab.com/standards/
- Privacy-centric protocol enabling advertisers and publishers to perform attribution in a Data Clean Room using Privacy-Enhancing Technologies (PETs) without exposing raw PII. Relevant for enterprise attribution use cases requiring publisher-side data sharing.

**IAB Europe Commerce Media Measurement Standards V2 (January 2026)**
- URL: https://iabeurope.eu/wp-content/uploads/IAB-Europes-Commerce-Media-Measurement-Standards-V2-Jan-2026.pdf
- Standards for retail media attribution measurement across commerce platforms; relevant as Amazon, Walmart, and Instacart attribution expand in scope.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic**
- URL: https://modelcontextprotocol.io/
- Open protocol enabling AI agents to connect to external data sources and tools. SegmentStream has already implemented an MCP Server, positioning attribution data as queryable context for AI coding and analysis agents. An AI-native attribution platform should expose an MCP Server allowing AI agents to query attribution models, run what-if budget scenarios, and retrieve channel performance data in natural language workflows.

---

## Similar Products — Developer Documentation & APIs

### Meta Conversions API (CAPI)

- **Description:** Server-side API enabling advertisers to send web, app, and offline conversion events directly from their servers to Meta, bypassing browser-based pixel limitations caused by ITP, cookie deprecation, and ad blockers.
- **API Documentation:** https://developers.facebook.com/docs/marketing-api/conversions-api/
- **Get Started Guide:** https://developers.facebook.com/docs/marketing-api/conversions-api/get-started/
- **Gateway Setup:** https://developers.facebook.com/docs/marketing-api/gateway-products/conversions-api-gateway/setup
- **SDKs/Libraries:** Official Business SDK (PHP, Python, Node.js, Ruby, Java) at https://developers.facebook.com/docs/business-sdk/
- **Standards:** REST/JSON; HTTP POST to `https://graph.facebook.com/{API_VERSION}/{PIXEL_ID}/events`
- **Authentication:** System User Access Token (OAuth 2.0 flow) generated via Meta Business Manager; token scoped to `ads_management` permission

---

### Google Ads Enhanced Conversions API

- **Description:** Server-side conversion import API that supplements click-based conversion tracking with hashed first-party customer data (email, phone, address) to improve attribution accuracy and match rate under privacy constraints.
- **API Documentation:** https://developers.google.com/google-ads/api/docs/conversions/overview
- **Getting Started:** https://developers.google.com/google-ads/api/docs/conversions/getting-started
- **Enhanced Conversions for Web:** https://support.google.com/google-ads/answer/13261987
- **SDKs/Libraries:** Google Ads API client libraries for Python, Java, Ruby, PHP, .NET, Perl at https://developers.google.com/google-ads/api/docs/client-libs/
- **Standards:** REST/JSON and gRPC; OpenAPI-based resource model (`ConversionAction`, `ClickConversion`, `CallConversion`)
- **Authentication:** OAuth 2.0 with developer token; customer ID (CID) scoped access; as of February 2026, Data Manager API is the primary path for complex conversion data management

---

### Google Meridian MMM Framework

- **Description:** Open-source Bayesian MMM framework from Google enabling advertisers to run in-house media mix models with geo-level granularity and native YouTube reach/frequency integration.
- **API Documentation:** https://developers.google.com/meridian
- **GitHub Repository:** https://github.com/google/meridian
- **SDKs/Libraries:** Python package (`meridian`); dependencies include TensorFlow Probability, pandas, NumPy
- **Standards:** Python-native; no REST API (batch computation framework); outputs as data frames and JSON summary files
- **Authentication:** N/A (local execution); Google Ads API OAuth required for fetching input data via the Google Ads Data API
- **Licence:** Apache 2.0

---

### TikTok Events API

- **Description:** Server-side API enabling advertisers to send web and app conversion events directly to TikTok for Ads, supplementing the TikTok Pixel for improved attribution accuracy. Supports event deduplication when used alongside the pixel.
- **API Documentation:** https://ads.tiktok.com/help/article/events-api
- **Payload Converter Tool:** https://ads.tiktok.com/help/article/about-events-api-payload-converter
- **SDKs/Libraries:** Official TikTok Business API SDK (Python, Java, Node.js) at https://github.com/TikTok/tiktok-business-api-sdk
- **Standards:** REST/JSON; HTTP POST to `https://business-api.tiktok.com/open_api/v1.3/event/track/`; event names mapped from standard e-commerce events (Purchase → `CompletePayment`)
- **Authentication:** Access Token (long-lived, generated in TikTok Ads Manager); App ID + Secret for OAuth flows

---

### Segment (Twilio) Tracking API

- **Description:** Customer Data Platform (CDP) providing a unified event collection and routing layer; commonly used as the upstream data source feeding attribution platforms. Segment's Tracking API collects events from web, mobile, server, and cloud apps and routes them to attribution tools.
- **API Documentation:** https://segment.com/docs/connections/sources/catalog/libraries/server/http-api/
- **SDKs/Libraries:** Analytics.js (web), analytics-node (Node.js), analytics-python, analytics-java, analytics-ruby, analytics-go, analytics-ios, analytics-android
- **Developer Guide:** https://segment.com/docs/getting-started/
- **Standards:** REST/JSON; OpenAPI spec available; standard event types (`track`, `identify`, `page`, `group`, `alias`)
- **Authentication:** Write Key (per source); server-side calls use HTTP Basic Auth with Write Key as username

---

### RudderStack (Open Source CDP)

- **Description:** Open-source alternative to Segment providing server-side event collection, identity resolution, and routing. Commonly used by cost-conscious teams to build attribution data pipelines without vendor lock-in.
- **API Documentation:** https://www.rudderstack.com/docs/api/http-api/
- **GitHub Repository:** https://github.com/rudderlabs/rudder-server
- **SDKs/Libraries:** JavaScript, Python, Go, Java, Node.js, Ruby, PHP SDKs; mirrors Segment API format
- **Standards:** REST/JSON; Segment-compatible API format enabling drop-in replacement; OpenAPI spec available
- **Authentication:** Write Key per source; same HTTP Basic Auth pattern as Segment
- **Licence:** ELv2 (core); Rudderstack Cloud SaaS also available

---

### Snowflake Marketing Data Cloud API

- **Description:** Cloud data warehouse providing the analytical compute layer for attribution computation and MMM; Snowflake's Native App Framework and Snowpark enable attribution models to run in-warehouse without data movement.
- **API Documentation:** https://docs.snowflake.com/en/developer-guide/snowflake-rest-api/
- **SDKs/Libraries:** Snowflake Connector for Python, snowflake-sqlalchemy, Snowpark Python/Java/Scala, JDBC/ODBC drivers
- **Developer Guide:** https://docs.snowflake.com/en/developer-guide/
- **Standards:** REST/JSON API (Snowflake REST API); SQL-compatible; supports OpenAPI for Snowflake Native App integrations
- **Authentication:** OAuth 2.0, key-pair authentication, or username/password; supports SSO and SAML 2.0 for enterprise

---

### LinkedIn Marketing API (Insight Tag + Conversions API)

- **Description:** LinkedIn's server-side Conversions API enables B2B advertisers to send conversion events (form fills, demo bookings, pipeline events) from their CRM or server to LinkedIn for attribution and campaign optimisation — critical for B2B marketing attribution platforms.
- **API Documentation:** https://learn.microsoft.com/en-us/linkedin/marketing/integrations/ads-reporting/conversions-api
- **Developer Guide:** https://learn.microsoft.com/en-us/linkedin/marketing/
- **SDKs/Libraries:** No official SDK; REST/JSON integration using HTTP POST; community Python wrappers available
- **Standards:** REST/JSON; HTTP POST to LinkedIn Marketing API endpoint; event schema includes `conversionHappenedAt`, `conversionValue`, `user` (email hash or LinkedIn member ID)
- **Authentication:** OAuth 2.0 with `rw_conversions` scope; access tokens managed via LinkedIn Developer Portal

---

### Google Analytics 4 (Measurement Protocol)

- **Description:** GA4's server-side Measurement Protocol allows attribution platforms to send events directly to Google Analytics from servers, enabling enrichment of GA4 data with server-side conversion signals for use in GA4's data-driven attribution model.
- **API Documentation:** https://developers.google.com/analytics/devguides/collection/protocol/ga4
- **Reference:** https://developers.google.com/analytics/devguides/collection/protocol/ga4/reference
- **SDKs/Libraries:** No official SDK; raw HTTP POST integration; community libraries available for Python and Node.js
- **Standards:** REST/JSON; HTTP POST to `https://www.google-analytics.com/mp/collect`; event schema follows GA4 event taxonomy
- **Authentication:** API Secret (per data stream) combined with client_id or app_instance_id; GA4 Measurement Protocol does not use OAuth

---

## Notes

**Evolving Privacy Sandbox status:** Google's Privacy Sandbox Attribution Reporting API has had an uncertain rollout timeline. As of 2026, third-party cookie deprecation in Chrome has proceeded in stages, but the industry has largely pivoted to server-side Conversions APIs rather than relying on the browser-native Attribution Reporting API. Attribution platforms should implement CAPI integrations as the primary track and treat the Attribution Reporting API as a supplementary signal.

**Data clean rooms:** AWS Clean Rooms, Google Ads Data Hub, and Meta Advanced Analytics are increasingly relevant for enterprise attribution use cases requiring cross-publisher data matching without raw PII sharing. The IAB ADMaP protocol is the emerging standard in this space; a marketing attribution platform targeting enterprise buyers should plan to support at least one clean room integration.

**Model Context Protocol (MCP) is early-stage:** MCP adoption as an integration standard for AI agents is growing rapidly but not yet a universal expectation. SegmentStream is an early mover; most attribution platforms have not yet published MCP Servers. An AI-native attribution platform publishing an MCP Server would have a meaningful differentiation in 2026–2027 as AI coding and analysis agents become mainstream.
