# Marketing Attribution Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native attribution stack spanning rules-based models, data-driven multi-touch attribution, media mix modelling, and incrementality testing in a single platform.

This project aims to give marketing teams a credible, privacy-resilient view of which channels and touchpoints actually drive conversions, rather than over-crediting whatever happened to be clicked last. It targets both DTC e-commerce and B2B revenue teams who today must stitch together separate tools for MTA, MMM, and incrementality, and who increasingly need first-party, server-side data collection to survive third-party cookie deprecation and walled-garden fragmentation.

---

## Why Marketing Attribution Platform?

- Last-click attribution systematically over-credits bottom-funnel channels and erodes brand investment, while privacy changes have degraded the accuracy of legacy multi-touch implementations.
- Incumbents like Northbeam and Rockerbox combine MTA + MMM + incrementality but are priced for DTC brands spending $50K+ per month on paid media, leaving smaller and B2B teams underserved.
- Walled-garden platforms (Google, Meta, TikTok, Amazon) each expose attribution data in incompatible formats; unifying them into a single source of truth via Conversions APIs is technically demanding.
- Open-source MMM frameworks exist (Google Meridian, Meta Robyn — now archived) but require data science teams and provide no end-to-end attribution platform; there is no open-source equivalent to Northbeam or Rockerbox.
- B2B attribution (HockeyStack, Dreamdata) and DTC attribution (Northbeam, Triple Whale) are siloed into separate tool categories despite sharing the same underlying methodology.

---

## Key Features

### Server-Side Data Collection & Identity

- Server-side first-party event collection via Meta Conversions API, Google Enhanced Conversions, and TikTok Events API
- GDPR and CCPA compliant data collection with consent signal handling, including Google Consent Mode v2
- Cross-device tracking and probabilistic identity resolution
- Privacy-compliant probabilistic conversion modelling for non-consent users

### Attribution Models

- Rules-based models: first-click, last-click, linear, time-decay, and position-based
- Data-driven (ML) attribution as the primary recommended model, weighting touchpoints by predicted conversion probability
- Account-level attribution aggregating touchpoints by company domain for B2B buying committees
- View-through attribution for video and display channels using deterministic matching

### Media Mix Modelling & Incrementality

- Bayesian media mix modelling using Google Meridian or equivalent open-source framework
- Automated budget reallocation recommendations based on marginal ROAS from MMM outputs
- Incrementality test design assistant with automated market selection, sample sizing, and significance monitoring
- Post-purchase survey integration for self-reported attribution capture

### AI Analyst & Closed-Loop Optimisation

- Natural-language querying of attribution data across channels, campaigns, and creative
- Automated anomaly detection for tracking failures, platform API changes, and data quality issues
- Closed-loop optimisation pushing enriched first-party signals back to ad platform bidding via Conversions APIs
- MCP Server API for AI agent integration

### Unified Dashboard & Data Foundation

- Cross-channel dashboard showing spend, conversions, and attributed revenue by channel and campaign
- Shopify, Salesforce, and HubSpot integrations for conversion and CRM data ingestion
- dbt-compatible data models for warehouse-based teams
- Data warehouse export to Snowflake and BigQuery

---

## AI-Native Advantage

AI is used to surface attribution insights in plain language (replacing complex SQL and BI queries), to detect anomalies in tracking data before they corrupt downstream models, and to recommend incrementality test configurations without expert services engagement. ML-based attribution weighting and Bayesian MMM are combined to produce automated channel-level budget reallocation recommendations grounded in both touchpoint-level and aggregate spend efficiency analysis. This collapses work that today is split between specialist analytics vendors and in-house data science teams.

---

## Tech Stack & Deployment

The platform is designed around server-side Conversions APIs (Meta, Google, TikTok), open-source Bayesian MMM frameworks (Google Meridian on Python, optionally Meta Robyn on R), and a warehouse-native architecture using Snowflake or BigQuery with dbt for transformations. Identity resolution and event routing draw on customer data platform patterns (Segment, Rudderstack), while orchestration uses Airflow or Prefect for MMM retraining and attribution scoring. Self-hosted deployment is the primary mode, with an MCP Server interface for AI agent integration.

---

## Market Context

The marketing attribution software market reached USD 5.4 billion in 2026, nearly doubling from USD 3.1 billion in 2021 ([research.md](research.md)). Primary buyers split into DTC e-commerce brands spending significantly on paid media (currently served by Northbeam, Rockerbox, Triple Whale) and B2B revenue marketing teams (HockeyStack, Dreamdata); enterprise tooling typically starts at price points that exclude SMBs and growth-stage brands.

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
