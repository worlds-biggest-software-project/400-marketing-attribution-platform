# Data Model Suggestion 2: Hybrid Relational + JSONB

> Project: Marketing Attribution Platform · Created: 2026-05-29

## Philosophy

This model keeps touchpoints and conversions as relational tables (for high-volume time-series queries and multi-model attribution joins) while embedding channel definitions, attribution model results, MMM outputs, incrementality experiments, and campaign metadata into JSONB on their parent records. Each conversion embeds its full touchpoint journey with per-model credit weights. The tenant holds channel taxonomy, model configurations, and MMM results.

Attribution analysis has two access patterns: population-level queries ("total attributed revenue by channel by model this month") that need relational indexing on channel, model, and date; and conversion-level queries ("show me the full journey for this conversion") that benefit from a self-contained document. The hybrid approach serves both.

**Best for:** Teams that want rapid iteration on attribution model configurations and channel taxonomies without migrations, while maintaining relational performance for aggregate channel-level ROAS dashboards and bulk data warehouse exports.

**Trade-offs:**
- (+) Conversion carries its full journey — single fetch for journey drill-down
- (+) New attribution models added as JSONB keys without schema changes
- (+) Channel and MMM configuration iterates without ALTER TABLE
- (+) Touchpoints remain relational for aggregate queries
- (-) Cross-conversion journey pattern analysis requires JSONB operators
- (-) Attribution credit embedded on conversions duplicates touchpoint references
- (-) Larger conversion rows for long multi-touch journeys

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Meta CAPI / Google EC / TikTok Events API | Touchpoint ingestion from server-side APIs |
| Google Consent Mode v2 | Consent signals on touchpoints |
| CloudEvents v1.0 | Event ingestion format |
| OpenAPI 3.1 | REST API |
| OAuth 2.0 | Ad platform API authentication |
| GDPR / CCPA | Consent tracking, PII hashing |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    business_model  TEXT NOT NULL CHECK (business_model IN ('dtc', 'b2b', 'hybrid')),
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',

    users_json      JSONB NOT NULL DEFAULT '[]',

    channels_json   JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Meta Paid Social", "channel_type": "paid_social",
    --   "platform": "meta", "capi_configured": true
    -- }]

    data_sources_json JSONB NOT NULL DEFAULT '[]',

    models_json     JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "name": "Data-Driven v3", "model_type": "data_driven",
    --   "status": "deployed", "attribution_window_days": 30,
    --   "config": {"min_touchpoints": 100, "regularization": "l2"}
    -- }]

    mmm_json        JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "run_date": "2026-05-25", "framework": "meridian",
    --   "training_period": {"start": "2025-06-01", "end": "2026-05-25"},
    --   "channel_contributions": [{
    --     "channel_id": "uuid", "name": "Meta", "contribution_pct": 0.28,
    --     "marginal_roas": 2.8, "saturation_cents": 500000,
    --     "recommended_change_pct": 0.15
    --   }],
    --   "budget_recommendations": {"total_budget_cents": 10000000, "optimal": [...]},
    --   "diagnostics": {"r_squared": 0.92, "mape": 0.08}
    -- }]

    incrementality_json JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "id": "uuid", "channel_id": "uuid", "name": "Meta Holdout Q2",
    --   "test_type": "geo_holdout", "status": "completed",
    --   "start_date": "2026-04-01", "end_date": "2026-04-30",
    --   "config": {"holdout_markets": ["DMA-501"], "significance": 0.95},
    --   "results": {"lift_pct": 0.12, "p_value": 0.003, "incremental_roas": 3.2}
    -- }]

    config_json     JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "plan": "growth",
    --   "attribution_window_days": 30,
    --   "consent": {"consent_mode_v2": true, "probabilistic_modelling": true},
    --   "activation": {"meta_capi": true, "google_ec": true, "tiktok_events": true},
    --   "alerts": {"tracking_anomaly_threshold": 0.2, "channels": ["slack"]}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tenants_slug ON tenants(slug);
```

## Touchpoints & Conversions

```sql
CREATE TABLE touchpoints (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    visitor_id      TEXT NOT NULL,
    account_id      TEXT,
    channel_id      UUID NOT NULL,
    touchpoint_type TEXT NOT NULL CHECK (touchpoint_type IN (
        'click', 'impression', 'view_through', 'email_open', 'email_click',
        'sms_click', 'direct_visit', 'organic_visit', 'referral_click',
        'video_view', 'content_engagement', 'demo_request', 'survey_response'
    )),
    occurred_at     TIMESTAMPTZ NOT NULL,
    campaign_id     TEXT,
    campaign_name   TEXT,
    creative_id     TEXT,
    creative_name   TEXT,
    keyword         TEXT,
    landing_page    TEXT,
    utm_source      TEXT,
    utm_medium      TEXT,
    utm_campaign    TEXT,
    cost_cents      BIGINT,
    device_type     TEXT,
    country         CHAR(2),
    consent_signals JSONB,
    data_source     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_touchpoints_visitor ON touchpoints(tenant_id, visitor_id);
CREATE INDEX idx_touchpoints_account ON touchpoints(tenant_id, account_id) WHERE account_id IS NOT NULL;
CREATE INDEX idx_touchpoints_channel ON touchpoints(channel_id);
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
    data_source     TEXT NOT NULL,
    survey_channel  TEXT,

    -- Embedded journey with multi-model attribution
    journey_json    JSONB NOT NULL DEFAULT '[]',
    -- [{
    --   "touchpoint_id": "uuid", "channel": "Meta Paid Social", "channel_id": "uuid",
    --   "type": "click", "at": "2026-05-20T...",
    --   "campaign": "Summer Sale", "creative": "Video A",
    --   "attribution": {
    --     "first_click": 0.0, "last_click": 0.0, "linear": 0.33,
    --     "time_decay": 0.15, "position_based": 0.10, "data_driven": 0.22
    --   }
    -- }, {
    --   "touchpoint_id": "uuid", "channel": "Email", "channel_id": "uuid",
    --   "type": "email_click", "at": "2026-05-23T...",
    --   "attribution": {"first_click": 0.0, "last_click": 0.0, "linear": 0.33,
    --     "time_decay": 0.35, "position_based": 0.40, "data_driven": 0.38}
    -- }, {
    --   "touchpoint_id": "uuid", "channel": "Direct", "channel_id": "uuid",
    --   "type": "direct_visit", "at": "2026-05-25T...",
    --   "attribution": {"first_click": 0.0, "last_click": 1.0, "linear": 0.33,
    --     "time_decay": 0.50, "position_based": 0.50, "data_driven": 0.40}
    -- }]

    -- Summary attribution per model
    attribution_summary JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "total_touchpoints": 3, "days_to_conversion": 5,
    --   "winning_channel": {"data_driven": "Email", "last_click": "Direct"},
    --   "channel_credits": {
    --     "data_driven": {"Meta Paid Social": 0.22, "Email": 0.38, "Direct": 0.40}
    --   }
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_conversions_visitor ON conversions(tenant_id, visitor_id);
CREATE INDEX idx_conversions_account ON conversions(tenant_id, account_id) WHERE account_id IS NOT NULL;
CREATE INDEX idx_conversions_tenant ON conversions(tenant_id, occurred_at);
CREATE INDEX idx_conversions_type ON conversions(tenant_id, conversion_type);
CREATE INDEX idx_conversions_journey ON conversions USING GIN (journey_json);
```

## Spend, AI & Audit

```sql
CREATE TABLE channel_spend (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    channel_id      UUID NOT NULL,
    spend_date      DATE NOT NULL,
    spend_cents     BIGINT NOT NULL,
    currency        CHAR(3) NOT NULL,
    impressions     BIGINT,
    clicks          BIGINT,
    conversions_reported INT,
    data_source     TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, channel_id, spend_date)
);

CREATE INDEX idx_channel_spend_tenant ON channel_spend(tenant_id, spend_date);

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
| Configuration | 1 | tenants (embeds users, channels, models, MMM, incrementality, config) |
| Touchpoints & Conversions | 2 | touchpoints (partitioned), conversions (partitioned, embeds journey + attribution) |
| Spend, AI & Audit | 3 | channel_spend, ai_suggestions, audit_log (partitioned) |
| **Total** | **6** | 3 partitioned tables |

---

## Key Design Decisions

1. **Journey embedded on conversions** — each conversion carries its full touchpoint journey with per-model credit weights in journey_json. The conversion detail view renders the complete path from a single row.

2. **Multi-model attribution embedded** — each touchpoint in the journey carries attribution weights for all models (first_click, last_click, linear, time_decay, position_based, data_driven). Model comparison is a JSONB extraction, not a separate table join.

3. **MMM results on tenant** — mmm_json stores per-channel contribution, marginal ROAS, saturation curves, and budget recommendations from each model run. Historical runs are preserved in the array.

4. **Incrementality tests on tenant** — incrementality_json stores test configuration and results, keeping experiments as tenant-level analytical artefacts.

5. **Touchpoints remain relational** — the highest-volume table stays relational with partitioning for aggregate channel spend queries and journey construction.

6. **Channel spend relational** — channel_spend enables ROAS calculation (attributed revenue from conversions.journey_json / spend) with efficient date-range queries.

7. **Attribution summary on conversion** — conversions.attribution_summary provides pre-computed channel credits per model for the dashboard, without parsing the full journey_json.

8. **Survey channel on conversion** — conversions.survey_channel captures self-reported attribution alongside modelled attribution for calibration.
