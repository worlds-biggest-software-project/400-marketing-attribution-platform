# Data Model Suggestion 3: Event-Sourced / Audit-First

> Project: Marketing Attribution Platform · Created: 2026-05-29

## Philosophy

This model treats every touchpoint, conversion, attribution computation, MMM run, incrementality test, budget recommendation, and closed-loop signal as an immutable event. Attribution results are not mutable records — they are projections computed by replaying touchpoint and conversion events through the active attribution model. Re-attribution with a different model replays the same events and produces new attribution events. MMM training and scoring are events with full parameter traceability. Incrementality test lifecycle is an event sequence.

Marketing attribution is a natural fit for event sourcing because the core analytical question is temporal: "which touchpoints in what sequence led to this conversion?" An event-sourced model preserves the exact touchpoint sequence, enables retroactive re-attribution (switching from last-click to data-driven without re-ingesting data), and permanently records every budget recommendation and closed-loop signal sent back to ad platforms.

**Best for:** Organisations requiring full attribution audit trails, retroactive re-attribution when models change, permanent recording of every signal sent to ad platform bidding (closed-loop), and temporal queries for attribution window analysis.

**Trade-offs:**
- (+) Re-attribution without re-ingestion: replay events through any model
- (+) Closed-loop signals permanently auditable
- (+) MMM and incrementality results traceable to exact inputs
- (+) New attribution models added by creating new projections
- (-) Channel ROAS dashboard requires read model
- (-) Aggregate attribution queries need materialised projections
- (-) High touchpoint volume requires careful partition management
- (-) More complex for marketing teams unfamiliar with event sourcing

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| CloudEvents v1.0 | Event store uses CloudEvents envelope |
| Meta CAPI / Google EC / TikTok Events API | Touchpoint ingestion events |
| Google Consent Mode v2 | Consent signals on touchpoint events |
| GDPR / CCPA | Consent events, data deletion via crypto-shredding |

---

## Event Infrastructure

```sql
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_type     TEXT NOT NULL CHECK (stream_type IN (
        'tenant', 'user', 'channel', 'data_source',
        'visitor', 'account', 'touchpoint', 'conversion',
        'attribution', 'mmm', 'incrementality',
        'activation', 'spend', 'ai', 'config'
    )),
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    event_type      TEXT NOT NULL,
    actor_type      TEXT NOT NULL CHECK (actor_type IN (
        'user', 'system', 'api_key', 'webhook', 'ai',
        'capi_ingestion', 'attribution_engine', 'mmm_engine',
        'activation_engine', 'scheduler', 'projection_engine'
    )),
    actor_id        TEXT,
    tenant_id       UUID,

    ce_source       TEXT NOT NULL,
    ce_type         TEXT NOT NULL,
    ce_time         TIMESTAMPTZ NOT NULL,
    ce_specversion  TEXT NOT NULL DEFAULT '1.0',

    data            JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_type, stream_id, version)
) PARTITION BY RANGE (ce_time);

CREATE INDEX idx_events_stream ON event_store(stream_type, stream_id, version);
CREATE INDEX idx_events_tenant ON event_store(tenant_id, ce_time);
CREATE INDEX idx_events_type ON event_store(event_type, ce_time);

CREATE TABLE stream_snapshots (
    stream_type     TEXT NOT NULL,
    stream_id       UUID NOT NULL,
    version         BIGINT NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_type, stream_id, version)
);

CREATE TABLE projection_checkpoints (
    projection_name TEXT NOT NULL,
    partition_key   TEXT NOT NULL DEFAULT '_global',
    last_event_id   UUID NOT NULL,
    last_event_time TIMESTAMPTZ NOT NULL,
    state           JSONB NOT NULL DEFAULT '{}',
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (projection_name, partition_key)
);
```

## Event Taxonomy

### Touchpoint Events
- `touchpoint_recorded` — {visitor_id, account_id, channel_id, touchpoint_type, campaign_id, campaign_name, creative_id, creative_name, keyword, landing_page, utm_source/medium/campaign, cost_cents, device_type, country, consent_signals, data_source}
- `touchpoint_deduplicated` — {touchpoint_id, duplicate_of}

### Conversion Events
- `conversion_recorded` — {visitor_id, account_id, conversion_type, revenue_cents, currency, order_id, deal_id, data_source, survey_channel}
- `conversion_corrected` — {original_id, corrected_fields, reason}

### Attribution Events
- `attribution_computed` — {conversion_id, model, touchpoints: [{touchpoint_id, channel_id, credit_weight, attributed_revenue_cents, position, days_to_conversion}]}
- `attribution_recomputed` — {conversion_id, model, reason, previous_results, new_results}
- `attribution_batch_completed` — {model, conversions_attributed, period, model_version}

### Spend Events
- `channel_spend_recorded` — {channel_id, spend_date, spend_cents, currency, impressions, clicks, conversions_reported, data_source}

### MMM Events
- `mmm_training_started` — {framework, training_period_start, training_period_end, geo_level, config}
- `mmm_training_completed` — {model_id, channel_contributions, budget_recommendations, diagnostics, model_version}
- `mmm_budget_recommendation_generated` — {total_budget_cents, optimal_allocation, marginal_roas_by_channel}

### Incrementality Events
- `incrementality_test_designed` — {channel_id, test_name, test_type, config}
- `incrementality_test_started` — {test_id, start_date}
- `incrementality_test_data_collected` — {test_id, metrics_snapshot}
- `incrementality_test_completed` — {test_id, results}
- `incrementality_test_cancelled` — {test_id, reason}

### Activation Events
- `activation_signal_sent` — {platform, signal_type, conversion_id, hashed_identifiers, event_name}
- `activation_signal_acknowledged` — {platform, response_code, match_rate}
- `activation_audience_exported` — {platform, audience_type, filter_criteria, customer_count}

### AI Events
- `ai_suggestion_generated` — {suggestion_type, entity_type, entity_id, title, description, evidence, confidence}
- `ai_suggestion_accepted` — {suggestion_id}
- `ai_suggestion_dismissed` — {suggestion_id, reason}
- `ai_anomaly_detected` — {anomaly_type, channel_id, metric, expected_value, actual_value}
- `ai_attribution_commentary_generated` — {period, commentary, key_drivers}
- `ai_incrementality_design_recommended` — {channel_id, recommended_config, rationale}

---

## Read Models

```sql
CREATE TABLE rm_channel_dashboard (
    tenant_id       UUID NOT NULL,
    channel_id      UUID NOT NULL,
    period_date     DATE NOT NULL,

    channel_name    TEXT NOT NULL,
    channel_type    TEXT NOT NULL,
    platform        TEXT,

    spend_cents         BIGINT NOT NULL DEFAULT 0,
    impressions         BIGINT NOT NULL DEFAULT 0,
    clicks              BIGINT NOT NULL DEFAULT 0,
    conversions_reported INT NOT NULL DEFAULT 0,

    -- Attribution by model
    attribution_json    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "first_click": {"conversions": 45, "revenue_cents": 450000, "roas": 1.8},
    --   "last_click": {"conversions": 62, "revenue_cents": 620000, "roas": 2.5},
    --   "data_driven": {"conversions": 55, "revenue_cents": 550000, "roas": 2.2}
    -- }

    -- Blended metrics
    blended_roas        NUMERIC(10,4),
    cpa_cents           BIGINT,

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, channel_id, period_date)
) PARTITION BY RANGE (period_date);

CREATE TABLE rm_conversion_journey (
    tenant_id       UUID NOT NULL,
    conversion_id   UUID NOT NULL,

    visitor_id      TEXT NOT NULL,
    account_id      TEXT,
    conversion_type TEXT NOT NULL,
    revenue_cents   BIGINT,
    occurred_at     TIMESTAMPTZ NOT NULL,
    survey_channel  TEXT,

    journey         JSONB NOT NULL DEFAULT '[]',
    attribution     JSONB NOT NULL DEFAULT '{}',
    days_to_conversion INT,
    total_touchpoints INT,

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, conversion_id)
);

CREATE INDEX idx_rm_journey_visitor ON rm_conversion_journey(tenant_id, visitor_id);
CREATE INDEX idx_rm_journey_account ON rm_conversion_journey(tenant_id, account_id) WHERE account_id IS NOT NULL;

CREATE TABLE rm_mmm_dashboard (
    tenant_id       UUID NOT NULL,
    model_id        UUID NOT NULL,

    framework       TEXT NOT NULL,
    run_date        DATE NOT NULL,
    training_period JSONB NOT NULL,
    channel_contributions JSONB NOT NULL DEFAULT '[]',
    budget_recommendations JSONB NOT NULL DEFAULT '{}',
    diagnostics     JSONB NOT NULL DEFAULT '{}',

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, model_id)
);

CREATE TABLE rm_incrementality_tracker (
    tenant_id       UUID NOT NULL,
    test_id         UUID NOT NULL,

    channel_name    TEXT NOT NULL,
    test_name       TEXT NOT NULL,
    test_type       TEXT NOT NULL,
    status          TEXT NOT NULL,
    start_date      DATE,
    end_date        DATE,
    config          JSONB NOT NULL,
    results         JSONB,

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, test_id)
);

CREATE TABLE rm_activation_log (
    tenant_id       UUID NOT NULL,
    signal_id       UUID NOT NULL,

    platform        TEXT NOT NULL,
    signal_type     TEXT NOT NULL,
    conversion_id   UUID,
    sent_at         TIMESTAMPTZ NOT NULL,
    response_code   INT,
    match_rate      NUMERIC(5,4),

    last_event_id   UUID NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, signal_id)
);

CREATE INDEX idx_rm_activation_platform ON rm_activation_log(tenant_id, platform, sent_at DESC);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Infrastructure | 3 | event_store (partitioned), stream_snapshots, projection_checkpoints |
| Read Models | 5 | rm_channel_dashboard (partitioned), rm_conversion_journey, rm_mmm_dashboard, rm_incrementality_tracker, rm_activation_log |
| **Total** | **8** | 2 partitioned tables |

---

## Key Design Decisions

1. **Attribution as replayable events** — attribution_computed events store per-touchpoint credit weights per model. Re-attribution with a new model replays touchpoint and conversion events to produce new attribution_computed events; old events remain for audit.

2. **Closed-loop signals as events** — activation_signal_sent and activation_signal_acknowledged events permanently record every conversion signal sent to Meta CAPI, Google Enhanced Conversions, or TikTok Events API, with hashed identifiers and platform response. Critical for debugging match rates and auditing what data was shared.

3. **Channel dashboard as a multi-model projection** — rm_channel_dashboard materialises per-channel per-day attribution results for all active models simultaneously, enabling the model comparison dashboard.

4. **Conversion journey as a read model** — rm_conversion_journey provides the full touchpoint-to-conversion path with multi-model attribution for the journey drill-down view.

5. **MMM training and results as events** — mmm_training_started → mmm_training_completed events record every MMM run with full parameters, diagnostics, and channel contributions. Budget recommendations are traceable to specific model runs.

6. **Incrementality test lifecycle as events** — incrementality_test_designed → started → data_collected → completed creates a complete experiment audit trail with statistical results.

7. **Consent signals preserved on events** — touchpoint events carry Google Consent Mode v2 signals, enabling consent-filtered attribution and GDPR-compliant probabilistic modelling projections.

8. **AI incrementality design as events** — ai_incrementality_design_recommended events record AI-generated experiment configurations with rationale, making the AI assistant's suggestions auditable.

9. **Spend events for ROAS projection** — channel_spend_recorded events feed the rm_channel_dashboard projection's ROAS calculation alongside attributed revenue from attribution events.

10. **Activation audience exports as events** — activation_audience_exported events record every audience list sent to ad platforms with filter criteria and customer count, supporting GDPR audit of data sharing.
