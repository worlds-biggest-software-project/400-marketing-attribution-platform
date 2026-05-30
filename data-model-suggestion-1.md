# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Marketing Attribution Platform · Created: 2026-05-29

## Philosophy

This model gives every attribution concept its own table with explicit foreign keys. The data flows through a clear pipeline: first-party events from server-side APIs (Meta CAPI, Google Enhanced Conversions, TikTok Events API) are ingested as touchpoints; conversions from e-commerce or CRM platforms are linked to touchpoint journeys; attribution models (rules-based and data-driven) compute credit weights per touchpoint per conversion; media mix modelling operates on aggregate channel spend and conversion data; and incrementality tests are tracked as structured experiments. Both DTC (transaction-based) and B2B (pipeline/deal-based) conversion types are supported.

The schema supports all major attribution methodologies in a single data model: rules-based (first-click, last-click, linear, time-decay, position-based), data-driven ML attribution, media mix modelling (Meridian/Robyn), and incrementality testing. Each methodology produces results that are comparable at the channel level.

**Best for:** Data teams requiring full SQL access to touchpoint-level journey data, multi-model attribution comparison, and auditable data lineage from server-side event through touchpoint to attributed conversion, across both DTC and B2B business models.

**Trade-offs:**
- (+) Full referential integrity from event → touchpoint → journey → conversion → attribution result
- (+) Multiple attribution models comparable in a single table
- (+) Both DTC and B2B conversion types in one schema
- (+) Incrementality tests as structured experiments with statistical outputs
- (-) Higher table count for a complex analytical domain
- (-) High-volume touchpoint data requires partitioned storage
- (-) Adding new ad platform integrations requires connector development

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Meta Conversions API | Server-side event ingestion mapped to touchpoints |
| Google Enhanced Conversions | Hashed first-party data for conversion matching |
| TikTok Events API | Server-side event ingestion |
| Google Consent Mode v2 | Consent signals stored on events |
| W3C Attribution Reporting API | Browser-side attribution signals as supplementary touchpoints |
| CloudEvents v1.0 | Event ingestion format |
| IAB/MRC Viewability Guidelines | View-through touchpoint qualification |
| IAB ADMaP | Clean room attribution protocol |
| OpenAPI 3.1 | REST API documentation |
| OAuth 2.0 | Ad platform and CRM API authentication |
| GDPR / CCPA | Consent tracking, data minimisation, PII hashing |

---

## Entity Management

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    business_model  TEXT NOT NULL CHECK (business_model IN ('dtc', 'b2b', 'hybrid')),
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    plan            TEXT NOT NULL CHECK (plan IN ('free', 'growth', 'enterprise')),
    attribution_window_days INT NOT NULL DEFAULT 30,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    name            TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'marketer', 'analyst', 'viewer', 'api_service')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);

CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    channel_type    TEXT NOT NULL CHECK (channel_type IN (
        'paid_search', 'paid_social', 'organic_search', 'organic_social',
        'email', 'sms', 'direct', 'referral', 'affiliate',
        'display', 'video', 'ctv', 'podcast', 'direct_mail', 'retail_media', 'other'
    )),
    platform        TEXT,  -- meta, google, tiktok, linkedin, pinterest, snap, etc.
    capi_configured BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_channels_tenant ON channels(tenant_id);
```

## Touchpoints & Journeys

```sql
CREATE TABLE touchpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    visitor_id      TEXT NOT NULL,  -- anonymised first-party ID
    account_id      TEXT,  -- B2B: company domain or account ID
    channel_id      UUID NOT NULL REFERENCES channels(id),
    touchpoint_type TEXT NOT NULL CHECK (touchpoint_type IN (
        'click', 'impression', 'view_through', 'email_open', 'email_click',
        'sms_click', 'direct_visit', 'organic_visit', 'referral_click',
        'video_view', 'content_engagement', 'webinar_attendance',
        'demo_request', 'survey_response'
    )),
    occurred_at     TIMESTAMPTZ NOT NULL,
    campaign_id     TEXT,
    campaign_name   TEXT,
    ad_group_id     TEXT,
    ad_group_name   TEXT,
    creative_id     TEXT,
    creative_name   TEXT,
    keyword         TEXT,
    landing_page    TEXT,
    utm_source      TEXT,
    utm_medium      TEXT,
    utm_campaign    TEXT,
    utm_content     TEXT,
    utm_term        TEXT,
    cost_cents      BIGINT,
    impressions     INT,
    device_type     TEXT CHECK (device_type IN ('desktop', 'mobile', 'tablet', 'ctv')),
    country         CHAR(2),
    consent_signals JSONB,
    -- {"ad_storage": "granted", "analytics_storage": "granted", "ad_personalization": "denied"}
    data_source     TEXT NOT NULL CHECK (data_source IN (
        'meta_capi', 'google_ec', 'tiktok_events', 'linkedin_capi',
        'pixel', 'server_side', 'segment', 'rudderstack', 'manual'
    )),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_touchpoints_visitor ON touchpoints(tenant_id, visitor_id);
CREATE INDEX idx_touchpoints_account ON touchpoints(tenant_id, account_id) WHERE account_id IS NOT NULL;
CREATE INDEX idx_touchpoints_channel ON touchpoints(channel_id);
CREATE INDEX idx_touchpoints_campaign ON touchpoints(tenant_id, campaign_id);
CREATE INDEX idx_touchpoints_tenant ON touchpoints(tenant_id, occurred_at);

CREATE TABLE conversions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    visitor_id      TEXT NOT NULL,
    account_id      TEXT,
    conversion_type TEXT NOT NULL CHECK (conversion_type IN (
        'purchase', 'subscription_start', 'lead', 'signup', 'demo_request',
        'trial_start', 'mql', 'sql', 'opportunity', 'closed_won', 'form_submit', 'custom'
    )),
    occurred_at     TIMESTAMPTZ NOT NULL,
    revenue_cents   BIGINT,
    currency        CHAR(3),
    order_id        TEXT,
    deal_id         TEXT,
    pipeline_stage  TEXT,
    data_source     TEXT NOT NULL CHECK (data_source IN (
        'shopify', 'bigcommerce', 'woocommerce', 'stripe',
        'salesforce', 'hubspot', 'custom_api', 'csv_import'
    )),
    survey_channel  TEXT,  -- self-reported attribution from post-purchase survey
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_conversions_visitor ON conversions(tenant_id, visitor_id);
CREATE INDEX idx_conversions_account ON conversions(tenant_id, account_id) WHERE account_id IS NOT NULL;
CREATE INDEX idx_conversions_tenant ON conversions(tenant_id, occurred_at);
CREATE INDEX idx_conversions_type ON conversions(tenant_id, conversion_type);
```

## Attribution Results

```sql
CREATE TABLE attribution_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    conversion_id   UUID NOT NULL REFERENCES conversions(id),
    touchpoint_id   UUID NOT NULL REFERENCES touchpoints(id),
    channel_id      UUID NOT NULL REFERENCES channels(id),
    model           TEXT NOT NULL CHECK (model IN (
        'first_click', 'last_click', 'linear', 'time_decay',
        'position_based', 'data_driven', 'deterministic_views'
    )),
    credit_weight   NUMERIC(8,6) NOT NULL,
    attributed_revenue_cents BIGINT,
    days_to_conversion INT,
    touchpoint_position INT,  -- position in journey (1 = first)
    total_touchpoints INT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_attribution_conversion ON attribution_results(conversion_id);
CREATE INDEX idx_attribution_channel ON attribution_results(tenant_id, channel_id, model);
CREATE INDEX idx_attribution_touchpoint ON attribution_results(touchpoint_id);
CREATE INDEX idx_attribution_model ON attribution_results(tenant_id, model);
```

## Channel Spend & MMM

```sql
CREATE TABLE channel_spend (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    channel_id      UUID NOT NULL REFERENCES channels(id),
    spend_date      DATE NOT NULL,
    spend_cents     BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL,
    impressions     BIGINT,
    clicks          BIGINT,
    conversions_reported INT,  -- platform-reported (may differ from attributed)
    data_source     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, channel_id, spend_date)
);

CREATE INDEX idx_channel_spend_tenant ON channel_spend(tenant_id, spend_date);

CREATE TABLE mmm_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    model_run_date  DATE NOT NULL,
    framework       TEXT NOT NULL CHECK (framework IN ('meridian', 'robyn', 'custom')),
    training_period_start DATE NOT NULL,
    training_period_end   DATE NOT NULL,
    geo_level       TEXT CHECK (geo_level IN ('national', 'regional', 'dma', 'custom')),
    channel_contributions JSONB NOT NULL,
    -- [{
    --   "channel_id": "uuid", "channel_name": "Meta Paid Social",
    --   "contribution_pct": 0.28, "marginal_roas": 2.8,
    --   "saturation_point_cents": 500000,
    --   "recommended_spend_change_pct": 0.15
    -- }]
    budget_recommendations JSONB NOT NULL DEFAULT '{}',
    -- {"total_budget_cents": 10000000, "optimal_allocation": [{...}]}
    model_parameters JSONB NOT NULL DEFAULT '{}',
    model_diagnostics JSONB NOT NULL DEFAULT '{}',
    -- {"r_squared": 0.92, "mape": 0.08, "dw_stat": 1.95}
    model_version   TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mmm_tenant ON mmm_results(tenant_id, model_run_date);

CREATE TABLE incrementality_tests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    channel_id      UUID NOT NULL REFERENCES channels(id),
    test_name       TEXT NOT NULL,
    test_type       TEXT NOT NULL CHECK (test_type IN (
        'geo_holdout', 'channel_holdout', 'budget_uplift', 'creative_holdout'
    )),
    status          TEXT NOT NULL DEFAULT 'planning' CHECK (status IN (
        'planning', 'running', 'completed', 'cancelled'
    )),
    start_date      DATE NOT NULL,
    end_date        DATE,
    test_config     JSONB NOT NULL,
    -- {"holdout_markets": ["DMA-501", "DMA-602"], "control_markets": ["DMA-803", "DMA-819"],
    --  "target_sample_size": 50000, "significance_level": 0.95}
    results         JSONB,
    -- {"incremental_conversions": 450, "incremental_revenue_cents": 4500000,
    --  "lift_pct": 0.12, "p_value": 0.003, "is_significant": true,
    --  "incremental_roas": 3.2, "confidence_interval": [0.08, 0.16]}
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_incrementality_tenant ON incrementality_tests(tenant_id);
CREATE INDEX idx_incrementality_channel ON incrementality_tests(channel_id);
```

## AI & Audit

```sql
CREATE TABLE ai_suggestions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    suggestion_type TEXT NOT NULL CHECK (suggestion_type IN (
        'budget_reallocation', 'tracking_anomaly', 'data_quality_issue',
        'channel_insight', 'creative_analysis', 'incrementality_design',
        'attribution_commentary', 'consent_gap', 'platform_api_change',
        'model_drift'
    )),
    entity_type     TEXT,
    entity_id       UUID,
    severity        TEXT NOT NULL CHECK (severity IN ('info', 'warning', 'critical')),
    title           TEXT NOT NULL,
    description     TEXT NOT NULL,
    evidence        JSONB NOT NULL DEFAULT '{}',
    recommended_action TEXT,
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'accepted', 'dismissed', 'expired', 'auto_applied')),
    confidence      NUMERIC(5,4),
    model_version   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);

CREATE INDEX idx_ai_suggestions_tenant ON ai_suggestions(tenant_id);
CREATE INDEX idx_ai_suggestions_pending ON ai_suggestions(tenant_id) WHERE status = 'pending';

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user', 'system', 'api_key', 'webhook', 'ai', 'scheduler')),
    actor_id        TEXT,
    action          TEXT NOT NULL,
    resource_type   TEXT NOT NULL,
    resource_id     UUID,
    changes         JSONB,
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE INDEX idx_audit_log_tenant ON audit_log(tenant_id, created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Entity Management | 3 | tenants, users, channels |
| Touchpoints & Conversions | 2 | touchpoints (partitioned), conversions (partitioned) |
| Attribution Results | 1 | attribution_results |
| Spend, MMM & Incrementality | 3 | channel_spend, mmm_results, incrementality_tests |
| AI & Audit | 2 | ai_suggestions, audit_log (partitioned) |
| **Total** | **11** | 3 partitioned tables |

---

## Key Design Decisions

1. **Touchpoints and conversions as separate tables** — touchpoints capture interactions (clicks, impressions, views); conversions capture outcomes (purchases, leads, deals). Attribution results link them with per-model credit weights.

2. **Multi-model attribution in one table** — attribution_results stores credit weights per touchpoint per conversion per model. Comparing first-click vs. data-driven attribution is a GROUP BY on the model column.

3. **B2B account-level attribution** — touchpoints.account_id and conversions.account_id enable B2B attribution by aggregating touchpoints per company domain, supporting the committee buying use case.

4. **Consent signals on touchpoints** — touchpoints.consent_signals stores Google Consent Mode v2 signals (ad_storage, analytics_storage, ad_personalization), enabling consent-filtered attribution and GDPR-compliant probabilistic modelling for non-consent users.

5. **MMM results with budget recommendations** — mmm_results stores per-channel contribution percentages, marginal ROAS, saturation points, and optimal allocation recommendations from Meridian/Robyn.

6. **Incrementality tests as structured experiments** — incrementality_tests stores test configuration (holdout markets, sample size, significance level) and results (lift, p-value, incremental ROAS) as first-class records.

7. **Survey-based attribution** — conversions.survey_channel captures post-purchase self-reported attribution ("How did you hear about us?"), providing a human signal to calibrate model outputs.

8. **Creative-level tracking** — touchpoints carry creative_id and creative_name, enabling creative-level ROAS analysis and AI-powered creative performance insights.

9. **Channel-level CAPI status** — channels.capi_configured tracks which channels have server-side Conversions API integration, informing data quality assessment per channel.

10. **Platform-reported vs. attributed conversions** — channel_spend.conversions_reported stores platform-reported conversions alongside attributed conversions from attribution_results, enabling discrepancy analysis.
