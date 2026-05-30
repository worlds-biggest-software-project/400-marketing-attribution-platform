# Marketing Attribution Platform — Phased Development Plan

> Project: 400-marketing-attribution-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three `data-model-suggestion-*.md` files. It adopts **Data Model 1 (Entity-Centric Normalized Relational)** as the canonical schema, because the platform's core differentiator is *multi-model attribution comparison with auditable lineage from server-side event → touchpoint → conversion → attribution result*. The normalized model makes "first-click vs. data-driven by channel" a single `GROUP BY model`, supports both DTC and B2B in one schema, and gives data teams full SQL access — exactly the open-source-Northbeam positioning. Where event-level auditability of closed-loop signals matters (Model 3's strength), we add a dedicated append-only `activation_signals` table rather than adopting full event sourcing, keeping the operational model simple.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The attribution and MMM core is statistics/ML-heavy. Google Meridian (TensorFlow Probability), PyMC/Stan, pandas, and the official Meta/Google/TikTok Business SDKs are all Python-first. One language for ingestion, attribution, MMM, and the AI analyst avoids a polyglot split. |
| API framework | **FastAPI** | Async (needed for high-volume CAPI ingestion and fan-out to ad-platform APIs), native Pydantic v2 validation, and auto-generated **OpenAPI 3.1** — a stated standards requirement (`standards.md`). |
| ASGI server | **Uvicorn + Gunicorn** | Standard production ASGI deployment; Gunicorn manages Uvicorn workers. |
| Database | **PostgreSQL 16** | Data Model 1 relies on declarative range **partitioning** (touchpoints, conversions, audit_log), `JSONB` (consent signals, MMM results), `GIN` indexes, and `INET`/`NUMERIC` types. SQLite is used only for fast unit tests of pure logic. |
| ORM / migrations | **SQLAlchemy 2.0 (async) + Alembic** | Mature async support over asyncpg; Alembic handles partitioned-table migrations and per-phase schema evolution. |
| DB driver | **asyncpg** | Fastest Postgres driver for async FastAPI; required for ingestion throughput. |
| Task queue | **Celery + Redis** | Attribution scoring, MMM training (long-running), CAPI dispatch with retries, and survey/CRM sync are async. Redis doubles as broker and result backend. Celery Beat schedules nightly re-attribution and MMM retraining. |
| MMM engine | **Google Meridian** (Apache 2.0) | Only actively maintained open-source Bayesian MMM with geo-level granularity; `features.md` and `research.md` both name it. Robyn is archived. Wrapped behind an `MMMEngine` interface so a `custom` PyMC backend can be added later. |
| Attribution ML | **scikit-learn (logistic regression / gradient boosting) + SHAP** | Data-driven attribution weights touchpoints by predicted conversion contribution. SHAP values give per-touchpoint credit that is *explainable* — directly serving the "transparent, auditable attribution" opportunity in `features.md`. |
| LLM / AI analyst | **Anthropic Claude via the official SDK, with prompt caching** | Natural-language querying, anomaly commentary, and incrementality-design suggestions. Tool-use maps NL questions to parameterised SQL against read views (never raw model-authored SQL). |
| MCP server | **Python `mcp` SDK** | Exposes attribution queries and what-if budget scenarios as MCP tools — the differentiator called out in `standards.md` and `features.md` (only SegmentStream ships one today). |
| Frontend | **Next.js 15 (App Router) + TypeScript + Tailwind + shadcn/ui + Recharts** | Dashboard-first product (channel → campaign → creative drill-down). Server Components for data fetching, Recharts for spend/ROAS/attribution charts. |
| Auth | **OAuth 2.0 + OIDC (Authlib) + JWT (RFC 7519)** | `standards.md` mandates OAuth 2.0/PKCE for ad-platform tokens and OIDC for enterprise SSO. JWT for first-party session/API auth. |
| Secrets / ad-platform tokens | **Encrypted at rest (Fernet/AES-GCM) in `oauth_credentials`, KMS-wrapped key** | Long-lived Meta/Google/TikTok system-user tokens must never be stored in plaintext. |
| Containerisation | **Docker + docker-compose** | Self-hosted is the primary deployment mode (`README.md`). Compose wires api, worker, beat, postgres, redis, web. |
| Testing | **pytest + pytest-asyncio + httpx.AsyncClient + testcontainers-python** | Unit (pure logic on SQLite/in-memory), integration (real Postgres via testcontainers), mocked-API (respx for ad-platform HTTP), and e2e. |
| Code quality | **ruff (lint + format) + mypy (strict) + pre-commit** | Single fast linter/formatter; strict typing across the attribution math. |
| Package manager | **uv** | Fast, lockfile-based, reproducible Docker builds. |
| Warehouse export | **dbt-core package + Parquet via pyarrow** | `features.md` MVP requires dbt-compatible models; `standards.md` names Parquet for bulk export to Snowflake/BigQuery. |
| Validation / data quality | **Pydantic v2 + Great Expectations-style rule checks** | Consent-signal validation, currency/cents invariants, and pre-attribution data-quality gates. |

### Project Structure

```
marketing-attribution-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .pre-commit-config.yaml
├── README.md
├── openapi.json                      # exported spec (CI artifact)
├── migrations/                       # Alembic versions (incl. partition DDL)
│   └── versions/
├── dbt/                              # dbt-core package (Phase 8)
│   ├── dbt_project.yml
│   └── models/
│       ├── staging/
│       └── marts/
├── src/
│   └── attribution/
│       ├── __init__.py
│       ├── main.py                   # FastAPI app factory, router mounting
│       ├── config.py                 # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py            # async engine, session factory
│       │   ├── base.py               # DeclarativeBase, mixins
│       │   └── models/               # SQLAlchemy ORM mapping Data Model 1
│       │       ├── tenant.py
│       │       ├── user.py
│       │       ├── channel.py
│       │       ├── touchpoint.py
│       │       ├── conversion.py
│       │       ├── attribution_result.py
│       │       ├── channel_spend.py
│       │       ├── mmm_result.py
│       │       ├── incrementality_test.py
│       │       ├── activation_signal.py
│       │       ├── ai_suggestion.py
│       │       └── audit_log.py
│       ├── schemas/                  # Pydantic request/response + CloudEvents
│       │   ├── events.py
│       │   ├── attribution.py
│       │   ├── mmm.py
│       │   └── ...
│       ├── api/
│       │   ├── deps.py               # auth, tenant scoping, RBAC
│       │   ├── routers/
│       │   │   ├── ingest.py         # POST /v1/events (CloudEvents)
│       │   │   ├── conversions.py
│       │   │   ├── channels.py
│       │   │   ├── attribution.py    # query attributed revenue by model
│       │   │   ├── spend.py
│       │   │   ├── mmm.py
│       │   │   ├── incrementality.py
│       │   │   ├── activation.py
│       │   │   ├── ai.py             # NL query, suggestions
│       │   │   ├── exports.py        # Parquet/warehouse export
│       │   │   └── admin.py          # tenants, users, oauth connect
│       ├── ingestion/
│       │   ├── cloudevents.py        # CloudEvents 1.0 parse/validate
│       │   ├── identity.py           # visitor/account resolution
│       │   ├── dedup.py              # event_id dedup (pixel + CAPI)
│       │   └── consent.py            # Consent Mode v2 / GPC handling
│       ├── connectors/               # outbound + inbound integrations
│       │   ├── base.py               # Connector ABC
│       │   ├── meta_capi.py
│       │   ├── google_ec.py
│       │   ├── tiktok_events.py
│       │   ├── linkedin_capi.py
│       │   ├── shopify.py
│       │   ├── hubspot.py
│       │   └── salesforce.py
│       ├── attribution/              # the engine
│       │   ├── journey.py            # build ordered touchpoint journeys
│       │   ├── models/
│       │   │   ├── base.py           # AttributionModel ABC
│       │   │   ├── rules.py          # first/last/linear/time_decay/position
│       │   │   └── data_driven.py    # ML + SHAP credit
│       │   ├── account.py            # B2B account-level aggregation
│       │   └── runner.py             # batch attribution orchestration
│       ├── mmm/
│       │   ├── engine.py             # MMMEngine ABC
│       │   ├── meridian_engine.py
│       │   └── budget.py             # marginal-ROAS reallocation
│       ├── incrementality/
│       │   ├── design.py             # market selection, power/sample size
│       │   └── analysis.py           # lift, p-value, CI
│       ├── ai/
│       │   ├── analyst.py            # NL → tool-use → SQL
│       │   ├── anomaly.py            # tracking/data-quality detection
│       │   ├── prompts.py            # system/user templates
│       │   └── client.py             # Anthropic client w/ prompt caching
│       ├── activation/
│       │   └── dispatcher.py         # closed-loop signal push
│       ├── mcp/
│       │   └── server.py             # MCP tool definitions
│       ├── tasks/                    # Celery
│       │   ├── celery_app.py
│       │   ├── attribution_tasks.py
│       │   ├── mmm_tasks.py
│       │   ├── ingestion_tasks.py
│       │   └── beat_schedule.py
│       ├── security/
│       │   ├── crypto.py             # PII hashing (SHA-256), token encryption
│       │   └── auth.py
│       └── audit.py                  # audit_log writer
├── web/                              # Next.js dashboard
│   ├── package.json
│   └── app/
└── tests/
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                     # sample CloudEvents, journeys, spend CSV
```

The structure is grouped by concern, not by phase; every phase adds modules without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Config, Tenancy & Schema

### Purpose
Establish the project scaffold, configuration, database connectivity, multi-tenant data model core, authentication, and audit logging. After this phase the API boots, runs migrations, authenticates a user scoped to a tenant, and persists the entity-management tables. Everything else builds on this.

### Tasks

#### 1.1 — Project scaffold, tooling, and Docker
**What**: Create the repo skeleton with uv, ruff, mypy, pytest, Docker, and docker-compose.

**Design**:
- `pyproject.toml` declares dependencies (fastapi, uvicorn[standard], gunicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic, pydantic-settings, celery, redis, anthropic, scikit-learn, shap, google-meridian, pyarrow, authlib, respx, pytest, pytest-asyncio, httpx, testcontainers, ruff, mypy).
- `config.py` uses `pydantic_settings.BaseSettings`:
  ```python
  class Settings(BaseSettings):
      database_url: str
      redis_url: str = "redis://redis:6379/0"
      jwt_secret: str
      token_encryption_key: str            # base64 Fernet key
      anthropic_api_key: str | None = None
      default_attribution_window_days: int = 30
      environment: Literal["dev", "test", "prod"] = "dev"
      model_config = SettingsConfigDict(env_prefix="MAP_", env_file=".env")
  ```
- `docker-compose.yml` services: `api` (uvicorn), `worker` (celery), `beat` (celery beat), `postgres:16`, `redis:7`, `web` (Next.js). Healthchecks gate startup order.
- `Dockerfile` multi-stage: `uv sync --frozen` in builder, copy venv to slim runtime.

**Testing**:
- `Unit: Settings loads from env vars → correct typed values, defaults applied`
- `Unit: missing MAP_DATABASE_URL → ValidationError naming the field`
- `Integration: docker compose up → /healthz returns 200 within 60s`

#### 1.2 — Database session, base, and migration harness
**What**: Async SQLAlchemy engine, session dependency, declarative base with mixins, Alembic wired for partitioned tables.

**Design**:
- `db/session.py`: `create_async_engine(settings.database_url, pool_size=20)`, `async_sessionmaker`, FastAPI dependency `get_session()`.
- `db/base.py`: `class Base(DeclarativeBase)`; `TimestampMixin` (created_at, updated_at), `TenantScopedMixin` (tenant_id FK + index).
- Alembic configured with async template; first migration creates `pgcrypto` extension (for `gen_random_uuid()`).
- Partition helper: migrations create parent partitioned tables plus monthly partitions for the current ±N months, with a documented `create_partition(table, month)` SQL function for the beat job in Phase 2.

**Testing**:
- `Integration (testcontainers Postgres): run alembic upgrade head → all tables + partitions exist`
- `Integration: insert into touchpoints with occurred_at in current month → lands in correct partition`
- `Unit: get_session yields a session and closes it on context exit`

#### 1.3 — Entity-management tables (tenants, users, channels)
**What**: Implement the `tenants`, `users`, and `channels` tables from Data Model 1 as ORM models + migration.

**Design**: Exact DDL from Data Model 1 (`tenants` with `business_model` dtc/b2b/hybrid, `plan`, `attribution_window_days`; `users` with `role` enum incl. `api_service`; `channels` with `channel_type` and `platform`). Pydantic schemas `TenantCreate/Read`, `UserCreate/Read`, `ChannelCreate/Read`. Repository functions enforce `tenant_id` scoping on every query.

**Testing**:
- `Unit: ChannelCreate with channel_type='invalid' → ValidationError`
- `Integration: create tenant + user → UNIQUE(tenant_id, email) enforced on duplicate`
- `Integration: list channels filtered by tenant_id → never returns other tenants' rows`

#### 1.4 — Authentication, RBAC, and tenant scoping
**What**: JWT-based first-party auth, OIDC SSO hook, and a dependency that injects the current `(user, tenant)` and enforces role-based access — addressing OWASP API Top-10 BOLA.

**Design**:
- `security/auth.py`: password/OIDC login → signed JWT (`sub`, `tenant_id`, `role`, `exp`).
- `api/deps.py`: `current_principal()` decodes JWT; `require_role(*roles)` dependency factory. **All data routers depend on `current_principal` and filter by its `tenant_id`** — tenant_id is never taken from request body or path for data scoping.
- Roles: owner, admin, marketer, analyst, viewer, api_service (from Data Model 1).

**Testing**:
- `Unit: expired JWT → 401`
- `Unit: role=viewer hitting a write endpoint guarded by require_role('admin') → 403`
- `Integration: user from tenant A requests /v1/channels/{id} for tenant B's channel → 404 (BOLA prevented)`

#### 1.5 — Audit log
**What**: Append-only `audit_log` (partitioned) plus a writer used by all mutating endpoints.

**Design**: DDL from Data Model 1 (`actor_type`, `action`, `resource_type`, `resource_id`, `changes` JSONB, `ip_address` INET). `audit.py: async def record(session, *, actor_type, actor_id, action, resource_type, resource_id, changes, request)`.

**Testing**:
- `Integration: create channel → audit_log row with action='channel.created', correct actor + changes diff`
- `Integration: failed request (validation error) → no audit row written`

---

## Phase 2: Event Ingestion, Identity & Consent

### Purpose
Build the first-party event intake: a CloudEvents 1.0 ingestion endpoint, deduplication, identity resolution (visitor + B2B account), and Consent Mode v2 / GPC handling. After this phase the platform can receive and durably store touchpoints and conversions with full consent context — the raw material for all attribution.

### Tasks

#### 2.1 — Touchpoints & conversions tables (partitioned)
**What**: Implement the partitioned `touchpoints` and `conversions` tables from Data Model 1, plus a monthly partition-creation beat job.

**Design**: Exact DDL from Data Model 1 including all indexes (visitor, account partial, channel, campaign, tenant+occurred_at). `tasks/beat_schedule.py` adds a monthly task calling `create_partition` for next month.

**Testing**:
- `Integration: insert 10k touchpoints across 3 months → land in correct partitions, indexes used (EXPLAIN shows index scan on idx_touchpoints_visitor)`
- `Integration: beat partition task creates next-month partition idempotently`

#### 2.2 — CloudEvents ingestion endpoint
**What**: `POST /v1/events` accepting a CloudEvents 1.0 batch, validating, and enqueuing for persistence.

**Design**:
- Request schema (Pydantic, CloudEvents 1.0 binding):
  ```python
  class TouchpointData(BaseModel):
      visitor_id: str
      account_id: str | None = None
      channel_ref: str                      # channel name or id
      touchpoint_type: TouchpointType
      occurred_at: datetime
      campaign_id: str | None = None
      creative_id: str | None = None
      utm: UTMParams | None = None
      cost_cents: int | None = None
      device_type: DeviceType | None = None
      country: str | None = Field(None, max_length=2)
      consent: ConsentSignals | None = None
      data_source: DataSource
      event_id: str                         # idempotency key for dedup
  class CloudEvent(BaseModel):
      specversion: Literal["1.0"]
      type: str                             # "com.map.touchpoint" | "com.map.conversion"
      source: str
      id: str
      time: datetime
      data: TouchpointData | ConversionData
  ```
- Endpoint validates, writes to `ingestion` Celery task (returns 202 with accepted count). Synchronous path available for low-volume (`?sync=true`) returning 200.
- Maps `channel_ref` → `channel_id`; auto-creates channel if `auto_create_channels` tenant config is on.

**Testing**:
- `Unit: valid CloudEvent batch → parsed models`
- `Unit: specversion != "1.0" → 422`
- `Integration (sync): touchpoint event → row persisted with resolved channel_id`
- `Integration: 202 returned for async batch, task enqueued`

#### 2.3 — Deduplication
**What**: Dedup events sharing an `event_id` across pixel + CAPI (Meta/TikTok dedup pattern).

**Design**: `ingestion/dedup.py`: maintain a unique constraint on `(tenant_id, data_source, event_id)` in a lightweight `ingested_event_keys` table (Redis bloom filter as fast pre-check, DB constraint as source of truth). On collision, drop the later arrival and increment a `dedup_dropped` counter.

**Testing**:
- `Integration: same event_id from pixel then meta_capi → single touchpoint stored, dedup counter=1`
- `Integration: distinct event_ids → both stored`

#### 2.4 — Identity resolution
**What**: Resolve `visitor_id` (cross-device, probabilistic) and `account_id` (B2B, by company domain).

**Design**:
- `ingestion/identity.py`: deterministic merge when a logged-in identifier (hashed email) links two visitor_ids → write to `identity_links(tenant_id, primary_visitor_id, alias_visitor_id, method, confidence)`. Probabilistic stitching (IP + user-agent + timing) gated behind tenant config and confidence threshold.
- B2B: derive `account_id` from email domain or reverse-IP company lookup; store mapping.

**Testing**:
- `Unit: two visitor_ids sharing hashed email → merged to primary, confidence=1.0 (deterministic)`
- `Unit: probabilistic match below threshold → not merged`
- `Integration: B2B touchpoint with work email → account_id = company domain`

#### 2.5 — Consent handling (Consent Mode v2 / GPC)
**What**: Capture and enforce GDPR/CCPA consent on every event.

**Design**:
- `ConsentSignals` model: `ad_storage`, `analytics_storage`, `ad_personalization`, `functionality_storage` ∈ {granted, denied}, plus `gpc: bool`.
- `ingestion/consent.py`: stored in `touchpoints.consent_signals` JSONB. Attribution and activation later filter on consent; non-consented events are flagged `consent_modeled=true` and routed only to probabilistic modelling, never to deterministic activation/CAPI push.

**Testing**:
- `Unit: ad_storage='denied' → event flagged consent_modeled, excluded from activation-eligible set`
- `Unit: GPC=true (California) → ad_personalization forced to denied`
- `Integration: query attribution-eligible touchpoints → excludes denied-consent rows when tenant config requires consent`

---

## Phase 3: Attribution Engine — Rules-Based Models (Core Value)

### Purpose
Ship the heart of the product: journey construction and the five rules-based attribution models, writing per-touchpoint credit weights to `attribution_results`. After this phase a tenant can ingest events and immediately see attributed revenue by channel under first-click, last-click, linear, time-decay, and position-based models. This is the MVP's first user-visible value.

### Tasks

#### 3.1 — attribution_results table + journey builder
**What**: Implement `attribution_results` (Data Model 1) and a journey builder that orders a visitor's/account's touchpoints within the attribution window preceding a conversion.

**Design**:
- DDL from Data Model 1 (`model` enum, `credit_weight NUMERIC(8,6)`, `attributed_revenue_cents`, `touchpoint_position`, `total_touchpoints`).
- `attribution/journey.py`:
  ```python
  @dataclass
  class JourneyTouchpoint:
      touchpoint_id: UUID
      channel_id: UUID
      occurred_at: datetime
      position: int            # 1-based
  @dataclass
  class Journey:
      conversion_id: UUID
      revenue_cents: int | None
      touchpoints: list[JourneyTouchpoint]   # chronological
  def build_journey(conversion, touchpoints, window_days, by="visitor") -> Journey
  ```
- `by="account"` aggregates touchpoints across all visitors of the account (B2B).

**Testing**:
- `Unit: touchpoints outside window_days excluded`
- `Unit: touchpoints ordered chronologically, positions 1..n`
- `Unit: by='account' merges touchpoints across visitor_ids of same account`

#### 3.2 — Rules-based attribution models
**What**: Implement the five rules models behind a common interface.

**Design**:
- `attribution/models/base.py`:
  ```python
  class AttributionModel(Protocol):
      name: str
      def assign_credit(self, journey: Journey) -> list[float]:  # sums to 1.0
          ...
  ```
- `rules.py`:
  - **first_click**: `[1,0,...,0]`
  - **last_click**: `[0,...,0,1]`
  - **linear**: `1/n` each
  - **time_decay**: weight ∝ `2 ** (-Δt / half_life_days)` normalised; default half-life 7 days.
  - **position_based**: 40% first, 40% last, 20% split among middle (U-shaped); for n<3 degrade gracefully (n=1→[1]; n=2→[0.5,0.5]).
- `runner.py`: for each conversion, build journey, run each enabled model, write `attribution_results` rows with `attributed_revenue_cents = round(revenue_cents * weight)`. Idempotent: deletes prior results for `(conversion_id, model)` before writing.

**Testing**:
- `Unit: linear over 4 touchpoints → [0.25]*4, sum=1.0`
- `Unit: time_decay → later touchpoints get more credit; weights sum to 1.0`
- `Unit: position_based n=2 → [0.5,0.5]; n=1 → [1.0]; n=5 → [0.4,0.0667,0.0667,0.0667,0.4]`
- `Unit: attributed_revenue_cents sums to conversion revenue (±rounding tolerance)`
- `Integration: re-run runner for same conversion → no duplicate rows (idempotent)`

#### 3.3 — Batch attribution Celery task + scheduling
**What**: Async batch scoring of unattributed/newly-converted records, scheduled nightly and triggerable on demand.

**Design**: `tasks/attribution_tasks.py: score_conversions(tenant_id, model_list, since)`. Beat schedules nightly full re-score; an on-ingest hook enqueues incremental scoring for new conversions. Progress tracked; failures logged to `ai_suggestions` as `data_quality_issue` only if systemic.

**Testing**:
- `Integration: ingest 100 conversions → task produces attribution_results for all enabled models`
- `Integration (mocked clock): nightly beat trigger re-scores conversions modified since last run`

#### 3.4 — Attribution query API
**What**: `GET /v1/attribution` returning attributed revenue/conversions by channel × model × date range.

**Design**:
- Query params: `model`, `start`, `end`, `group_by` (channel|campaign|creative), `business_view` (visitor|account).
- Response:
  ```json
  {"model": "linear", "rows": [
     {"channel_id": "...", "channel_name": "Meta Paid Social",
      "attributed_conversions": 55.0, "attributed_revenue_cents": 5500000}]}
  ```
- Backed by an indexed aggregation on `attribution_results` joined to `channels`.

**Testing**:
- `Integration: query model=last_click grouped by channel → matches hand-computed fixture totals`
- `Integration: model comparison (first_click vs data_driven later) returns differing channel splits`
- `Unit: invalid model param → 422`

---

## Phase 4: Ad-Platform Connectors & Spend Ingestion

### Purpose
Connect to the walled gardens. Implement inbound spend ingestion and OAuth credential management for Meta, Google, TikTok, and LinkedIn, plus the `channel_spend` table. After this phase the dashboard can show spend alongside attributed revenue — i.e., ROAS — the cross-channel unified view that is table-stakes (`features.md`).

### Tasks

#### 4.1 — OAuth credential storage + connect flow
**What**: Secure storage and OAuth 2.0/PKCE connect flow for ad-platform tokens.

**Design**:
- `oauth_credentials(id, tenant_id, platform, encrypted_token BYTEA, scopes, expires_at, refresh_token_enc, created_at)`. Tokens encrypted with Fernet (`security/crypto.py`), key from `token_encryption_key`.
- `POST /v1/admin/connect/{platform}` initiates OAuth (Authlib), callback stores encrypted token. Refresh handled by a beat task before expiry.

**Testing**:
- `Unit: token round-trips through encrypt/decrypt unchanged; ciphertext != plaintext`
- `Integration (mocked OAuth server via respx): callback exchanges code → encrypted credential row`
- `Integration: expired token → refresh task obtains new token`

#### 4.2 — Connector base + channel_spend table
**What**: `Connector` ABC and the `channel_spend` table (Data Model 1).

**Design**:
- `connectors/base.py`:
  ```python
  class Connector(ABC):
      platform: str
      @abstractmethod
      async def fetch_spend(self, creds, start: date, end: date) -> list[SpendRow]: ...
      @abstractmethod
      async def send_conversions(self, creds, events: list[ActivationEvent]) -> ActivationResult: ...
  ```
- `channel_spend` DDL from Data Model 1 with `UNIQUE(tenant_id, channel_id, spend_date)`; upsert on conflict.

**Testing**:
- `Unit: SpendRow validation; negative spend_cents → ValidationError`
- `Integration: upsert same (channel, date) twice → single row, latest values`

#### 4.3 — Meta / Google / TikTok / LinkedIn spend connectors
**What**: Concrete inbound spend fetchers using the official SDKs/REST endpoints from `standards.md`.

**Design**: Each maps platform reporting (spend, impressions, clicks, platform-reported conversions) → `SpendRow`. Endpoints/auth per `standards.md` (Meta Marketing API, Google Ads API, TikTok Business API, LinkedIn Marketing API). Daily beat task pulls yesterday's spend per connected channel; retries with exponential backoff via Celery.

**Testing**:
- `Integration (mocked API, respx): Meta reporting response fixture → correct SpendRows`
- `Integration (mocked): API 429 → retried per backoff policy`
- `Integration (real, optional, marked): live sandbox token fetches one day of spend`

#### 4.4 — ROAS aggregation in attribution API
**What**: Extend `GET /v1/attribution` to join `channel_spend`, returning spend, attributed revenue, ROAS, and CPA per channel/model.

**Design**: ROAS = `attributed_revenue_cents / spend_cents`; CPA = `spend_cents / attributed_conversions`. Also surface `conversions_reported` (platform) vs attributed for discrepancy analysis (Data Model 1 decision #10).

**Testing**:
- `Integration: channel with spend + attributed revenue → correct ROAS in response`
- `Integration: zero spend → ROAS null, not divide-by-zero error`

---

## Phase 5: Conversion Sources — Shopify, Salesforce/HubSpot, Surveys

### Purpose
Ingest conversions from where they actually happen: e-commerce (DTC) and CRM (B2B), plus post-purchase survey capture. After this phase the platform closes the loop from touchpoint to real revenue/pipeline for both target personas, satisfying the MVP integration requirement (`features.md`).

### Tasks

#### 5.1 — Shopify connector (DTC conversions)
**What**: Ingest orders as `purchase` conversions via Shopify webhooks + backfill.

**Design**: `connectors/shopify.py`: register `orders/create` webhook; verify HMAC signature; map order → `ConversionData` (revenue_cents, currency, order_id, visitor_id from cart attribution / landing-page UTM correlation). Backfill via Admin API for history.

**Testing**:
- `Integration (mocked): valid HMAC webhook → conversion stored`
- `Integration (mocked): invalid HMAC → 401, no conversion`
- `Unit: order line items → revenue_cents incl. correct currency`

#### 5.2 — HubSpot / Salesforce connector (B2B conversions)
**What**: Ingest deals/opportunities and lifecycle-stage changes as B2B conversions tied to `account_id`.

**Design**: `connectors/hubspot.py`, `connectors/salesforce.py`: OAuth; map deal stage → conversion_type (`mql`, `sql`, `opportunity`, `closed_won`), `deal_id`, `account_id` (company domain), `revenue_cents` (deal amount). Sync via polling + webhooks where available.

**Testing**:
- `Integration (mocked): closed_won deal → conversion_type='closed_won', account_id set`
- `Unit: deal stage map covers all conversion_type B2B values`

#### 5.3 — Post-purchase survey capture
**What**: Capture self-reported attribution ("How did you hear about us?") into `conversions.survey_channel`.

**Design**: `POST /v1/conversions/{id}/survey {channel}`; also a Shopify post-checkout survey ingestion path. Used in Phase 7 to calibrate model outputs.

**Testing**:
- `Integration: post survey response → conversions.survey_channel updated`
- `Unit: unknown survey channel string → stored verbatim (free-text allowed)`

---

## Phase 6: Data-Driven (ML) Attribution — Primary Recommended Model

### Purpose
Deliver the platform's recommended model: ML-based, explainable, per-touchpoint credit. After this phase `data_driven` appears alongside the rules models in every query and dashboard, with SHAP-based explanations that non-technical marketers can audit — directly serving the "transparent, explainable attribution" opportunity.

### Tasks

#### 6.1 — Feature engineering & training pipeline
**What**: Build a training set of journeys (converters + matched non-converters) and train a contribution model.

**Design**:
- `attribution/models/data_driven.py`: construct journey-level features — channel presence/counts, position features, recency, device, time-to-conversion buckets. Negative samples = visitor journeys that did not convert in the window.
- Model: gradient-boosted classifier (`sklearn.ensemble.HistGradientBoostingClassifier`) predicting conversion. Persist model artifact + `model_version` (hash of training window + config).
- Cold-start guard: if `< min_journeys` (default 100, per Data Model 2 config), fall back to `position_based` and emit an `ai_suggestion` (`model_drift`/info) explaining the fallback.

**Testing**:
- `Unit: feature vector shape matches journey; channel one-hot covers all tenant channels`
- `Unit: < min_journeys → fallback path selected, suggestion emitted`
- `Integration: train on fixture → model artifact persisted with deterministic model_version`

#### 6.2 — SHAP credit assignment
**What**: Convert model outputs into per-touchpoint credit weights that sum to 1.0.

**Design**: Compute SHAP values for each touchpoint's marginal contribution to predicted conversion probability; normalise non-negative SHAP contributions across the journey to credit weights. Write to `attribution_results` with `model='data_driven'`. Store top contributing factors in a per-conversion explanation cache for the AI analyst.

**Testing**:
- `Unit: credit weights non-negative and sum to 1.0`
- `Unit: a touchpoint with zero marginal contribution → ~0 credit`
- `Integration: data_driven results queryable alongside rules models in /v1/attribution`

#### 6.3 — Model registry & scheduled retraining
**What**: Track model versions and retrain on a schedule with drift detection.

**Design**: `ml_models(id, tenant_id, model_type, model_version, status, metrics JSONB, trained_at)`; statuses pending→training→deployed→retired. Beat task retrains weekly; compares holdout AUC to deployed model; auto-deploys if improved, else emits `model_drift` suggestion.

**Testing**:
- `Integration: retrain with better AUC → new version deployed, old retired`
- `Integration: retrain with worse AUC → deployed unchanged, suggestion emitted`

---

## Phase 7: Media Mix Modelling & Budget Reallocation

### Purpose
Add the aggregate, privacy-resilient lens: Bayesian MMM via Google Meridian and automated budget reallocation from marginal ROAS. After this phase tenants get channel-level contribution, saturation curves, and concrete spend-shift recommendations — the differentiator separating this from MTA-only tools (`features.md`).

### Tasks

#### 7.1 — mmm_results table + MMMEngine interface
**What**: Implement `mmm_results` (Data Model 1) and an engine abstraction.

**Design**: DDL from Data Model 1 (`framework`, `channel_contributions` JSONB, `budget_recommendations` JSONB, `model_diagnostics` JSONB, `model_version`). `mmm/engine.py`:
```python
class MMMEngine(ABC):
    @abstractmethod
    def fit(self, weekly_spend: DataFrame, weekly_kpi: DataFrame, config: MMMConfig) -> MMMFit: ...
    @abstractmethod
    def contributions(self, fit) -> list[ChannelContribution]: ...
    @abstractmethod
    def optimize_budget(self, fit, total_budget_cents: int) -> BudgetPlan: ...
```

**Testing**:
- `Unit: MMMConfig validation (training window ≥ documented minimum weeks → warn if < 52)`
- `Integration: insert mmm_results with JSONB contributions → round-trips`

#### 7.2 — Meridian engine adapter
**What**: Wrap Google Meridian: assemble weekly spend/KPI per channel, fit, extract contributions, saturation, marginal ROAS.

**Design**: `mmm/meridian_engine.py` builds Meridian input arrays from `channel_spend` + attributed/observed conversions; runs MCMC fit; extracts `contribution_pct`, `marginal_roas`, `saturation_point_cents` per channel; writes diagnostics (`r_squared`, `mape`). Runs in a dedicated long-timeout Celery task. Data-requirement guard: warn (suggestion) when < 18 months of weekly data (`research.md` challenge).

**Testing**:
- `Integration (small synthetic dataset): fit completes, contributions sum ≈ 1.0, diagnostics present`
- `Unit: < 18 months data → data-sufficiency warning suggestion emitted`
- `Integration (real Meridian, optional/slow, marked): fit on fixture converges`

#### 7.3 — Budget reallocation recommendations
**What**: Turn marginal-ROAS curves into a reallocation plan and store it.

**Design**: `mmm/budget.py: optimize_budget` shifts spend toward channels with higher marginal ROAS until marginal ROAS equalises, respecting saturation points and per-channel min/max guardrails. Output → `mmm_results.budget_recommendations` and an `ai_suggestion` (`budget_reallocation`, severity by expected lift).

**Testing**:
- `Unit: two channels, one with higher marginal ROAS → plan shifts budget toward it, total budget conserved`
- `Unit: saturation cap respected (no channel exceeds saturation_point_cents)`
- `Integration: optimize → budget_reallocation suggestion created`

---

## Phase 8: Incrementality Testing, dbt Models & Warehouse Export

### Purpose
Add causal validation and warehouse interoperability: the incrementality test designer/analyser, dbt-compatible models, and Parquet export. After this phase tenants can validate channel causality with geo holdouts and plug attribution into their warehouse — closing out the MVP/v1.1 data-foundation scope.

### Tasks

#### 8.1 — incrementality_tests table + design assistant
**What**: Implement `incrementality_tests` (Data Model 1) and automated market selection + sample sizing.

**Design**: DDL from Data Model 1 (`test_type`, `status` planning→running→completed→cancelled, `test_config` JSONB, `results` JSONB). `incrementality/design.py`: cluster geos (DMAs) into matched test/control groups by baseline conversion volume; compute required sample size for a target MDE and significance (power analysis, `statsmodels`).

**Testing**:
- `Unit: power analysis → sample size increases as MDE decreases`
- `Unit: matched market selection balances baseline volume between groups`
- `Integration: create test → status='planning', config persisted`

#### 8.2 — Incrementality analysis
**What**: Compute lift, p-value, confidence interval, and incremental ROAS from test data.

**Design**: `incrementality/analysis.py`: diff-in-diff / synthetic-control on test vs control conversions over the window; output `{incremental_conversions, incremental_revenue_cents, lift_pct, p_value, is_significant, incremental_roas, confidence_interval}` → `results` JSONB; `is_significant = p_value < (1 - significance_level)`. Beat task monitors running tests and flips status to `completed` when significance reached or end_date passed.

**Testing**:
- `Unit: synthetic test with known lift → recovered lift within tolerance, p_value < 0.05`
- `Unit: no real effect → not significant`
- `Integration: monitoring task completes a test when end_date passed`

#### 8.3 — dbt package + Parquet/warehouse export
**What**: Ship dbt staging/mart models and a Parquet export endpoint.

**Design**:
- `dbt/models/marts/`: `channel_performance`, `conversion_journeys`, `attribution_by_model` conforming to the dbt Semantic Layer metric definitions (spend, attributed_revenue, roas, cpa) named in `standards.md`.
- `GET /v1/exports/{dataset}?format=parquet&start=&end=` streams Parquet (pyarrow) for Snowflake/BigQuery external-table load.

**Testing**:
- `Integration: dbt build against test Postgres → mart tables populated, metrics match API`
- `Integration: Parquet export → file reads back via pyarrow with expected schema`

---

## Phase 9: AI Analyst, Anomaly Detection & MCP Server

### Purpose
Deliver the AI-native layer: natural-language querying, automated anomaly detection, and the MCP server. After this phase marketers ask questions in plain language, the platform proactively flags tracking failures, and AI agents can query attribution via MCP — the platform's headline differentiation (`README.md`).

### Tasks

#### 9.1 — ai_suggestions table + anomaly detection
**What**: Implement `ai_suggestions` (Data Model 1) and detectors for tracking/data-quality anomalies.

**Design**: DDL from Data Model 1 (`suggestion_type`, `severity`, `evidence` JSONB, `status`, `confidence`). `ai/anomaly.py` detectors run on beat: sudden touchpoint-volume drop per channel (tracking break), CAPI match-rate decline, consent-gap spikes, conversions-reported vs attributed divergence (platform API change). Each emits a typed suggestion with evidence.

**Testing**:
- `Unit: 80% volume drop vs trailing baseline → tracking_anomaly suggestion, severity=critical`
- `Unit: within normal variance → no suggestion`
- `Integration: detector run writes suggestion; GET /v1/ai/suggestions returns pending ones`

#### 9.2 — Natural-language analyst (tool-use, not free SQL)
**What**: `POST /v1/ai/query {question}` → Claude maps the question to a parameterised query over read views and returns a narrated answer with the data.

**Design**:
- `ai/client.py`: Anthropic client with **prompt caching** on the system prompt (schema/tool catalogue) to cut cost.
- Tool-use: Claude selects from safe tools (`query_attribution(model, group_by, start, end)`, `compare_models(...)`, `channel_roas(...)`, `mmm_contributions()`, `whatif_budget(total_cents)`), each backed by parameterised SQL — the model never authors raw SQL. Result rows + a natural-language summary returned.
- `ai/prompts.py`: system prompt template defining role, available tools, tenant context, and a strict "only answer from tool results" instruction.

**Testing**:
- `Integration (mocked LLM): "which channel had best ROAS in April under data-driven?" → calls channel_roas tool with correct params, returns top channel`
- `Unit: model attempts unknown tool → rejected, error surfaced`
- `Integration: prompt caching header set on system block`

#### 9.3 — MCP server
**What**: Expose attribution queries and what-if budget scenarios as MCP tools for external AI agents.

**Design**: `mcp/server.py` (Python `mcp` SDK) registers tools mirroring 9.2's safe tool set plus `whatif_budget`. Auth via API key (`api_service` role) scoping to a tenant. Read-only except budget what-ifs (which never mutate, only simulate).

**Testing**:
- `Integration (MCP client harness): list tools → expected catalogue; call query_attribution → structured result`
- `Integration: API key for tenant A cannot read tenant B data`

---

## Phase 10: Closed-Loop Activation, View-Through & Dashboard

### Purpose
Complete the loop and the UI: push enriched first-party conversions back to ad platforms (closed-loop optimisation), add deterministic view-through attribution, and ship the Next.js dashboard. After this phase the product is end-to-end usable by a marketer through a browser, with attribution feeding back into ad-platform bidding.

### Tasks

#### 10.1 — activation_signals table + closed-loop dispatcher
**What**: Append-only `activation_signals` log and a dispatcher pushing conversions to Meta CAPI / Google EC / TikTok / LinkedIn with hashed PII.

**Design**:
- Table: `activation_signals(id, tenant_id, platform, conversion_id, event_name, hashed_identifiers JSONB, sent_at, response_code, match_rate, status)` — append-only audit (Model 3 strength retained without full event sourcing).
- `activation/dispatcher.py`: build platform payload via the Phase 4 connector's `send_conversions`; hash email/phone with SHA-256 normalised per `NIST SP 800-188` guidance; **only consent-eligible conversions** dispatched. Records request + platform response.

**Testing**:
- `Unit: email normalised+SHA-256 hashed (lowercase, trimmed) → matches platform spec vector`
- `Integration (mocked CAPI): consented conversion → signal sent, response + match_rate logged`
- `Integration: denied-consent conversion → never dispatched`

#### 10.2 — View-through attribution
**What**: Add deterministic view-through credit for impression touchpoints meeting viewability thresholds.

**Design**: Qualify `impression`/`view_through` touchpoints against MRC Viewable Impression Guidelines (50% pixels ≥1s display / ≥2s video) using OM SDK signals where present; include qualified views in journey building with a configurable, lower default credit ceiling vs clicks. New `model='deterministic_views'` value already in Data Model 1.

**Testing**:
- `Unit: impression below viewability threshold → excluded from journey`
- `Unit: qualified view-through included with capped credit`

#### 10.3 — Next.js dashboard
**What**: The marketer-facing UI: channel overview → campaign → creative drill-down, model switcher, MMM view, incrementality tracker, AI analyst chat, and suggestions inbox.

**Design**: Next.js App Router; Server Components fetch from the FastAPI v1 endpoints (auth via session JWT). Pages: `/dashboard` (channel ROAS table + spend/attributed-revenue charts with a model selector), `/journeys/[conversionId]`, `/mmm`, `/incrementality`, `/analyst` (NL chat to `/v1/ai/query`), `/suggestions`. Recharts for visualisations; shadcn/ui components.

**Testing**:
- `E2E (Playwright): log in → dashboard shows channels; switch model first_click→data_driven → table values change`
- `E2E: open conversion journey → ordered touchpoints with per-model credit`
- `E2E: ask analyst a question → answer + data table render`

#### 10.4 — OpenAPI spec export & docs
**What**: Export the OpenAPI 3.1 spec and serve interactive docs.

**Design**: FastAPI auto-generates `/openapi.json` (3.1); CI writes `openapi.json` artifact; `/docs` (Swagger) and `/redoc` enabled. Every v1 endpoint documented with examples.

**Testing**:
- `Integration: GET /openapi.json → openapi field == "3.1.0", all v1 paths present`
- `CI: spec diff check fails build if endpoints change without spec regeneration`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (skeleton, tenancy, auth, audit) ─── required by everything
    │
Phase 2: Ingestion, Identity & Consent ─── requires P1
    │
Phase 3: Attribution Engine (rules models) ─── requires P2   ← CORE VALUE SHIPS HERE
    │
    ├── Phase 4: Ad-Platform Connectors & Spend ─── requires P3 (can parallel P5)
    │       │
    │       └── Phase 7: MMM & Budget Reallocation ─── requires P4 (+P3 attributed/observed KPI)
    │
    ├── Phase 5: Conversion Sources (Shopify/CRM/survey) ─── requires P2 (can parallel P4)
    │
    └── Phase 6: Data-Driven ML Attribution ─── requires P3 + P5 (needs real conversions)
            │
            └── Phase 8: Incrementality, dbt & Export ─── requires P4 + P6
                    │
                    └── Phase 9: AI Analyst, Anomaly, MCP ─── requires P6, P7 (richest signals)
                            │
                            └── Phase 10: Activation, View-Through & Dashboard ─── requires P4, P6, P9
```

**Parallelism opportunities:**
- After Phase 3: **Phase 4** (ad-platform/spend) and **Phase 5** (conversion sources) can be built concurrently by separate developers.
- **Phase 6** (ML attribution) can begin in parallel with Phase 4 once Phase 5 provides real conversions.
- The **Next.js dashboard (10.3)** can be scaffolded against mocked API responses starting after Phase 3 and integrated progressively.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and mocked-integration tests pass; real-dependency integration tests pass against a testcontainers Postgres/Redis.
3. `ruff check` and `ruff format --check` pass with no errors.
4. `mypy --strict` passes for the phase's modules.
5. `docker compose build` succeeds and `docker compose up` reaches healthy state.
6. The phase's feature works end-to-end (demonstrated by an integration or e2e test using committed fixtures).
7. New config options are documented in `README.md` and reflected in `config.py` defaults.
8. New/changed API endpoints appear in the auto-generated OpenAPI 3.1 spec (`/openapi.json`), with request/response examples.
9. New tables/columns have an Alembic migration (including partition DDL where applicable) that runs cleanly forward and is reversible.
10. Any external data shared with ad platforms is consent-gated and PII is hashed (GDPR/CCPA, NIST SP 800-188); a corresponding `audit_log` or `activation_signals` record is written.
```
