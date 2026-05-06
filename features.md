# Marketing Attribution Platform — Feature & Functionality Survey

> Candidate #400 · Researched: 2026-05-06

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Northbeam | MTA + MMM + Incrementality | Commercial (SaaS) | https://www.northbeam.io/ |
| Rockerbox | MTA + MMM + Incrementality | Commercial (SaaS) | https://www.rockerbox.com/ |
| Triple Whale | MTA + MMM + Incrementality | Commercial (SaaS) | https://www.triplewhale.com/ |
| HockeyStack | MTA + AI Analyst (B2B) | Commercial (SaaS) | https://www.hockeystack.com/ |
| Cometly | MTA + AI + Conversion Sync | Commercial (SaaS) | https://www.cometly.com/ |
| SegmentStream | MTA + ML Modelling (B2B) | Commercial (SaaS) | https://segmentstream.com/ |
| Dreamdata | MTA + B2B Revenue Attribution | Commercial (SaaS) | https://dreamdata.io/ |
| Improvado | Marketing ETL + Attribution | Commercial (SaaS) | https://improvado.io/ |
| Google Meridian | Bayesian MMM | Open Source (Apache 2.0) | https://github.com/google/meridian |
| Meta Robyn | Bayesian MMM | Open Source (MIT — archived) | https://github.com/facebookexperimental/Robyn |

---

## Feature Analysis by Solution

### Northbeam

**Core features**
- Seven attribution models including last-click, first-click, linear, time-decay, position-based, data-driven MTA, and the proprietary Clicks + Deterministic Views model
- MMM+ module for strategic budget forecasting and scenario planning across channels
- Northbeam Incrementality: automated end-to-end Meta channel holdout testing (other channels planned)
- Apex integration layer that feeds attribution signals back into ad platform bidding algorithms
- Sales attribution dashboards, product analytics, and creative analytics
- Profit benchmarks and Metrics Explorer for correlation analysis across KPIs
- iOS mobile app for on-the-go dashboard access

**Differentiating features**
- "Clicks + Deterministic Views" — world's first deterministic view-through attribution model connecting ad views and clicks to real revenue across platforms
- Automated incrementality testing removing the manual experimental design burden
- Apex feedback loop enabling closed-loop optimisation by returning first-party signals to ad networks

**UX patterns**
- Purpose-built for DTC brands spending $50K+ per month on paid media across three or more channels
- Dashboard-first interface with drill-down from channel to campaign to creative
- Profit benchmarks provide industry-context comparisons rather than isolated brand metrics

**Integration points**
- Meta, Google, TikTok, Snap, Pinterest, CTV, and more via first-party pixel and server-side events
- Ad platform APIs for bid signal feedback (Apex)
- E-commerce platforms (Shopify-native)

**Known gaps**
- Not suited for B2B or SaaS businesses with long sales cycles
- Incrementality testing initially limited to Meta and US; international and other channels rolling out through 2026
- High cost threshold means smaller brands cannot justify the platform

**Licence / IP notes**
- Proprietary SaaS; "Clicks + Deterministic Views" positioning suggests potential IP around the deterministic view-through methodology

---

### Rockerbox

**Core features**
- Multi-touch attribution across digital and offline channels (paid social, search, display, video, CTV, linear TV, direct mail, podcasts)
- Media mix modelling for strategic budget planning and forecasting
- Manual incrementality testing with controlled holdout experiments
- Post-purchase survey integration for self-reported attribution capture
- Promo code and QR code tracking for hard-to-attribute offline channels
- Real-time data ingestion for campaign performance monitoring
- SOC 2 Type II certified data storage and processing

**Differentiating features**
- Broadest offline channel coverage among attribution platforms including linear TV, direct mail, and podcast tracking
- Post-purchase survey integration provides a "human in the loop" signal to supplement modelled attribution
- Combined MTA + MMM + incrementality in a single workflow with reconciliation views

**UX patterns**
- Designed for omnichannel enterprise DTC brands with complex multi-channel strategies
- Unified data foundation as the core selling point; methodology views on top
- 100+ out-of-the-box integrations with minimal setup

**Integration points**
- 100+ pre-built integrations covering all major ad platforms
- CRM and e-commerce platform connectors
- Data warehouse export (Snowflake, BigQuery)

**Known gaps**
- Incrementality testing is manual and expert-guided rather than automated
- AI-powered journey mapping described as recent/in beta rather than mature

**Licence / IP notes**
- Proprietary SaaS; SOC 2 Type II certification is a compliance selling point

---

### Triple Whale

**Core features**
- Compass unified measurement framework combining MTA, MMM, and incrementality testing
- Seven attribution models (single-touch and multi-touch, including Clicks & Deterministic Views)
- Email & SMS attribution pulling from Klaviyo, Attentive, Omnisend, and Postscript
- Sonar Send: enriches Klaviyo flows with Triple Whale attribution data for revenue lift
- Moby 2 AI analyst rebuilt on real-time data with a Context Engine for e-commerce questions
- Creative analytics for ad-level performance analysis
- Profit tracking with real-time COGS and margin integration

**Differentiating features**
- Strongest Shopify-native integration in the market; profit tracking at SKU and order level
- Email and SMS attribution closing a major gap for owned-channel measurement
- Sonar Send demonstrating closed-loop value by lifting Klaviyo revenue by an average of 14.2%
- Moby 2 AI for natural-language querying of all attribution and e-commerce data

**UX patterns**
- Positioned as an "Analytics OS" for Shopify brands; single pane of glass across ads and e-commerce
- Tiered pricing from $179/month making it more accessible than Northbeam
- Progressive disclosure: summary dashboard → channel deep-dive → creative analysis

**Integration points**
- Shopify (native, deep integration)
- Klaviyo, Attentive, Omnisend, Postscript for email and SMS attribution
- Meta, Google, TikTok, Pinterest, Snap, and more ad platforms
- Data export via API

**Known gaps**
- Less suited to B2B or non-Shopify e-commerce (BigCommerce, WooCommerce support more limited)
- MMM maturity reportedly behind Northbeam's MMM+ module
- Incrementality testing available but less automated than Northbeam

**Licence / IP notes**
- Proprietary SaaS; Sonar Send and Moby 2 are proprietary differentiators

---

### HockeyStack

**Core features**
- Full-funnel B2B attribution across anonymous site visits, ad impressions, content engagement, sales activity, and product usage, all tied at account level
- Multi-touch attribution with instant model switching and predictive attribution
- Incremental lift measurement to distinguish causal impact from correlation
- Odin AI analyst for natural-language GTM queries ("which campaigns drove pipeline last quarter?")
- Out-of-the-box dashboard templates with custom reporting capability
- Account-level engagement scoring aggregating touchpoints from the same company
- Forecasting based on real-time buyer behaviour

**Differentiating features**
- Account-level attribution is a structural B2B advantage; individual-user MTA cannot capture committee buying
- Odin AI analyst surfacing GTM insights in plain language is a significant UX differentiation
- Bridges PLG (product-led growth) and sales-led motions in a single attribution view

**UX patterns**
- Positioned for enterprise B2B GTM teams (marketing + sales alignment)
- Live dashboard templates enable immediate value; custom reporting for deeper analysis
- Natural-language interface (Odin) abstracts complex attribution queries

**Integration points**
- Salesforce, HubSpot CRM
- Gong, Zoom call recording
- LinkedIn, Google, Meta, and other ad platforms
- Revenue marketing and outreach platforms

**Known gaps**
- E-commerce and DTC use cases are not the target; weaker for transaction-focused attribution
- Pricing is enterprise-tier; not suitable for SMB B2B teams
- MMM capabilities less prominent compared to DTC-focused competitors

**Licence / IP notes**
- Proprietary SaaS; Odin AI branding suggests IP investment in the AI analyst layer

---

### Cometly

**Core features**
- Multi-touch attribution with real-time analytics dashboards across all major ad platforms
- AI-powered identification of high-performing ads and campaigns
- One-click conversion sync sending enriched event data back to Meta, Google, and TikTok
- AI Ads Manager providing actionable optimisation recommendations
- Machine learning attribution models, MMM, and incrementality testing
- Cross-device tracking and probabilistic identity resolution
- 100+ app integrations

**Differentiating features**
- Conversion Sync as a core primitive: enriched first-party conversion events fed back to ad platform algorithms to improve targeting and bidding
- AI Ads Manager providing prescriptive (not just descriptive) recommendations
- Positioned as the most accessible advanced attribution platform for growth-stage brands

**UX patterns**
- Simpler onboarding than enterprise competitors; designed for performance marketers without data science staff
- Dashboard-first with AI surfacing recommendations proactively
- iOS limitations and cookie deprecation positioned as problems Cometly solves out of the box

**Integration points**
- Meta Conversions API, Google Enhanced Conversions, TikTok Events API
- Shopify and major e-commerce platforms
- 100+ integrations directory

**Known gaps**
- MMM and incrementality testing appear less mature than Northbeam or Rockerbox
- B2B use cases not well supported
- Less depth in offline channel attribution

**Licence / IP notes**
- Proprietary SaaS

---

### SegmentStream

**Core features**
- Multi-model attribution suite: First-Touch, Last Paid Click, Last Paid Non-Brand Click, and ML Visit Scoring MTA
- Expert-led incrementality testing via geo holdout experiments with statistical powering
- Automated Marketing Mix Optimisation with weekly budget execution across ad platforms
- Cookieless attribution using first-party data and machine learning beyond third-party cookies
- Re-Attribution methodology combining self-reported survey data, coupon codes, and QR codes
- AI-ready infrastructure via MCP Server integration
- GDPR-compliant probabilistic conversion modelling for non-consent users

**Differentiating features**
- ML Visit Scoring MTA assigns weights to touchpoints based on predicted conversion probability rather than rules
- MCP Server integration positions it as AI-agent-ready infrastructure
- GDPR-compliant conversion recovery for users who opted out of tracking (probabilistic inference)
- Re-Attribution methodology recovers "Direct" and "Brand Search" misattribution using survey data

**UX patterns**
- Positioned for sophisticated B2B marketing teams requiring methodological transparency
- Expert-led services alongside the SaaS platform (geo holdout experiment design)
- Budget optimisation automation reduces manual reallocation effort

**Integration points**
- Google Ads, Meta Ads, LinkedIn Ads, and other major ad platforms
- CRM and analytics platforms
- Data warehouse export (BigQuery, Snowflake)
- MCP Server for AI agent integration

**Known gaps**
- Less product-led self-serve; requires expert involvement for incrementality testing
- Smaller global footprint than Rockerbox or Triple Whale
- E-commerce DTC use cases less prominent than B2B focus

**Licence / IP notes**
- Proprietary SaaS; MCP Server integration is an emerging differentiator

---

### Dreamdata

**Core features**
- B2B account-level multi-touch attribution connecting CRM, ad platforms, and website data
- Revenue and pipeline analytics with closed-won attribution reporting
- AI-powered audience building and activation syncing to major ad networks
- Intent Signals: AI-identified buying intent signals with notifications
- LinkedIn Conversions API and Company Intelligence integrations
- One-click conversion syncs for pipeline data to ad platforms
- BigQuery and Snowflake data warehouse access; Reverse ETL into BI tools

**Differentiating features**
- LinkedIn CAPI integration capturing far more B2B engagement signals than generic attribution tools
- AI agents surfacing pipeline insights, ICP fit anomalies, and channel ROI shifts proactively
- Audience Hub enabling precision targeting from attribution-qualified account lists

**UX patterns**
- Built for B2B revenue marketing and sales alignment teams
- Dreamdata Free plan provides a lower-cost entry point for SMB B2B teams
- Pipedrive and major CRM marketplace presence indicates self-serve discovery

**Integration points**
- Salesforce, HubSpot, ZohoCRM, Pipedrive
- LinkedIn Ads (deep CAPI integration), Google Ads, Meta Ads
- BigQuery, Snowflake
- Intent data providers and outreach platforms (Gong, etc.)

**Known gaps**
- E-commerce and DTC attribution is not the focus
- MMM capabilities not prominently featured
- Smaller ecosystem of integrations than Rockerbox (100+)

**Licence / IP notes**
- Proprietary SaaS; Dreamdata Free tier available

---

### Improvado

**Core features**
- 1,000+ pre-built connectors to advertising platforms, analytics tools, CRMs, and marketing automation systems
- ETL, ELT, and Reverse ETL pipelines with automated transformations
- 40,000+ metrics and dimensions across connected data sources
- 250+ pre-built validation rules specific to marketing data
- Custom attribution models showing how prospects interact with ads through to conversion
- Data Activation (native Reverse ETL) syncing processed data back to Salesforce, HubSpot, and ad networks
- Historical data preservation when upstream APIs change schema

**Differentiating features**
- Broadest connector library (1,000+) makes it the go-to for data engineering teams unifying fragmented marketing data
- JSON/XML connector with OAuth and Bearer token support for arbitrary REST APIs reduces custom integration work
- 250+ validation rules enforce data quality before attribution computation

**UX patterns**
- Positioned for marketing data engineering and IT teams rather than marketing analysts directly
- Managed service option alongside self-serve SaaS
- Connector-first: value is in data unification rather than attribution modelling per se

**Integration points**
- 1,000+ ad platforms, analytics, CRM, and marketing automation connectors
- Salesforce, HubSpot, and major CRM platforms for Reverse ETL
- Snowflake, BigQuery, Redshift, and other analytical warehouses

**Known gaps**
- Attribution modelling capabilities are less sophisticated than dedicated attribution tools
- Requires data engineering investment to realise full value
- UI designed for technical users rather than marketing analysts

**Licence / IP notes**
- Proprietary SaaS

---

### Google Meridian

**Core features**
- Open-source Bayesian MMM framework built on Python
- Handles geo-level data for regional and local-level media mix insights
- Integrates YouTube reach and frequency data and Google Search data natively
- Highly customisable modelling framework based on Bayesian causal inference
- Designed for in-house implementation by brand data science teams

**Differentiating features**
- Only open-source MMM framework with native YouTube reach/frequency integration
- Geo-level granularity enables regional budget allocation rather than just national-level
- Google-backed, actively maintained (unlike Meta Robyn which is archived)

**UX patterns**
- Developer and data scientist audience; Python notebook-based workflow
- Documentation at developers.google.com/meridian
- Self-hosted; no vendor relationship required

**Integration points**
- Python ecosystem (pandas, NumPy, TensorFlow Probability)
- Google Ads API and YouTube Data API for input data
- BigQuery for large-scale data ingestion

**Known gaps**
- No UI; requires data science expertise to configure and interpret
- No automated budget reallocation; outputs inform manual or custom-coded decisions
- Google channel data has native advantage; other channel integrations require manual data preparation

**Licence / IP notes**
- Apache 2.0 licence; freely usable including for commercial purposes

---

### Meta Robyn

**Core features**
- Open-source Bayesian MMM framework built on R
- Ridge regression with saturation and adstock transformations
- Budget allocator and response curve analysis
- Automated hyperparameter optimisation using Nevergrad
- Calibration with incrementality tests

**Differentiating features**
- First widely adopted open-source MMM framework; large community of implementations
- Budget allocator producing actionable spend reallocation recommendations
- R-native making it accessible to statisticians and econometricians

**UX patterns**
- R script-based; Jupyter and RMarkdown notebook workflows
- GitHub-hosted with community examples
- Now archived by Meta; community forks continue active development

**Integration points**
- R ecosystem (tidyverse, prophet)
- Any data source loadable as a CSV or data frame

**Known gaps**
- Meta has discontinued active development; community must maintain it
- R dependency limits adoption among Python-first data teams
- No native view-through or digital signal integration beyond what users supply

**Licence / IP notes**
- MIT licence; freely usable including for commercial purposes

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Multi-touch attribution with at least first-click, last-click, linear, and time-decay models
- First-party server-side event collection (Conversions API integration with Meta, Google, TikTok)
- Cross-channel unified dashboard showing spend, conversions, and attributed revenue by channel
- Data-driven or ML-based attribution model alongside rules-based models
- Privacy-compliant data collection (GDPR and CCPA consent handling)
- Integration with major e-commerce platforms (Shopify at minimum) or CRMs (Salesforce, HubSpot)

### Differentiating Features
- Media mix modelling alongside MTA (not all platforms offer both)
- Automated incrementality testing (Northbeam is first to automate; others require expert involvement)
- View-through attribution beyond click-based tracking (Northbeam and Triple Whale both claiming "deterministic views")
- AI analyst for natural-language querying of attribution data (HockeyStack Odin, Triple Whale Moby 2)
- Closed-loop optimisation: feeding attribution signals back into ad platform bidding algorithms (Northbeam Apex, Cometly Conversion Sync)
- Account-level attribution for B2B buying committees (HockeyStack, Dreamdata)
- GDPR-compliant probabilistic conversion modelling for non-consent users (SegmentStream)
- MCP Server / AI agent integration (SegmentStream)

### Underserved Areas / Opportunities
- Unified attribution across DTC and B2B in a single platform (currently served by separate tool categories)
- Offline-to-online attribution without requiring expensive data partnerships or manual survey work
- Real-time incrementality testing (current holdout experiments require weeks to accumulate significance)
- Transparent, explainable attribution models that non-technical marketers can audit and trust
- Open-source end-to-end attribution platform covering MTA, MMM, and incrementality (no open-source equivalent to Northbeam or Rockerbox exists)
- Automated geo holdout experiment design without expert services engagement
- Attribution for retail media networks (Amazon, Walmart, Instacart) beyond conventional digital channels
- Self-serve MMM for SMBs (Meridian and Robyn require data science teams)

### AI-Augmentation Candidates
- Natural-language querying of attribution data across all channels and models (replacing complex SQL/BI queries)
- Automated anomaly detection in attribution data identifying tracking breaks, platform API changes, or data quality issues before they corrupt models
- AI-generated incrementality test design recommending appropriate holdout configurations, market selection, and sample size
- Predictive attribution forecasting future channel contribution based on historical pattern modelling
- AI-powered creative attribution linking specific ad creative elements (colour, copy, format) to conversion lift rather than just campaign-level ROAS
- Automated channel-level budget reallocation recommendations grounded in both MTA and MMM outputs

---

## Legal & IP Summary

No patent concerns were identified in public sources for any of the tools surveyed. The core attribution methodologies (last-click, MTA, MMM, Bayesian inference) are well-established academic and industry techniques with no proprietary barriers. Open-source frameworks Google Meridian (Apache 2.0) and Meta Robyn (MIT) can be used freely including in commercial products. SegmentStream, HockeyStack, Dreamdata, and other SaaS platforms are proprietary but their methodologies are publicly documented in marketing literature. The "Clicks + Deterministic Views" branding used by both Northbeam and Triple Whale may indicate a shared methodology or parallel development; the term is used by both without explicit cross-licensing references in public sources. GDPR and CCPA compliance requirements apply to any platform processing personal data for attribution purposes, and Google Consent Mode v2 is now mandatory for EEA/UK Google Ads and Analytics usage; these represent legal obligations rather than IP concerns.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Server-side first-party event collection supporting Meta CAPI, Google Enhanced Conversions, and TikTok Events API
- Rules-based attribution models: first-click, last-click, linear, time-decay, and position-based
- Unified cross-channel dashboard with spend, conversions, and attributed revenue by channel and campaign
- Data-driven (ML) attribution model as the primary recommended model
- GDPR and CCPA compliant data collection and processing with consent signal handling
- Shopify and/or Salesforce/HubSpot integration for conversion and CRM data ingestion
- dbt-compatible data models for warehouse-based teams

**Should-have (v1.1)**
- Bayesian media mix modelling module using Google Meridian or equivalent open-source framework
- Automated budget reallocation recommendations based on marginal ROAS from MMM outputs
- Post-purchase survey integration for self-reported attribution capture
- Incrementality test design assistant: automated market selection, sample sizing, and significance monitoring
- Account-level attribution for B2B use cases (aggregate touchpoints by company domain)
- AI analyst interface for natural-language attribution queries (channel, campaign, and creative performance)

**Nice-to-have (backlog)**
- Closed-loop optimisation: direct API integration to push attribution signals back to ad platform bidding
- View-through attribution for video and display channels using deterministic matching
- Offline channel attribution for TV, direct mail, and podcast via media schedule import and media mix modelling
- MCP Server API for AI agent integration
- Retail media attribution (Amazon Ads API, Walmart Connect API)
- Automated anomaly detection for tracking failures and data quality issues
