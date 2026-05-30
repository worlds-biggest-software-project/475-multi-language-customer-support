# Multi-Language Customer Support — Phased Development Plan

> Project: 475-multi-language-customer-support · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The database design is based on **Data Model Suggestion 1 (Normalized Relational / PostgreSQL)** as the system of record, with the translation-memory (pgvector) and time-series analytics ideas from **Suggestion 4** introduced as optional add-ons in later phases. JSONB escape hatches from **Suggestion 3** are adopted selectively (integration metadata, channel metadata, provider raw responses).

---

## Product Summary

**What it does.** An AI-native translation middleware that sits *between* an existing helpdesk (Zendesk, Salesforce, Freshdesk, Intercom, ServiceNow) and the customer. It detects the customer's language, translates inbound messages into the agent's working language and outbound responses back into the customer's language, applies a per-tenant brand glossary, scores translation quality, learns from agent corrections, localizes knowledge-base articles, and reports per-language analytics. It deploys as self-hosted, cloud-hosted, or hybrid, and never requires the customer to migrate off their helpdesk.

**Primary users.** Support operations teams and CX leaders at global companies; agents working translated conversations; integration engineers wiring the middleware to a helpdesk.

**Key differentiators.** Middleware (no migration) · domain-adapted translation with per-tenant brand glossary · self-improving glossary that learns from agent corrections · proactive translation-quality trend monitoring with degradation alerts · transparent per-translation cost tracking · multi-provider routing (Google, DeepL, Azure, Amazon) with failover.

**Standards anchors (from `standards.md`).** Language identification uses **ISO 639** codes and **IETF BCP 47 / RFC 5646** tags throughout. KB article interchange uses **XLIFF 2.2**. The REST API is documented with **OpenAPI 3.1**. An **MCP server** exposes translate/glossary/route tools. Helpdesk connector auth uses **OAuth 2.0**. Locale-sensitive rendering uses **Unicode CLDR**. Compliance posture targets **GDPR**, **SOC 2 Type II**, and **ISO/IEC 42001** (AI management).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | **Python 3.12** | The core workload is LLM/NMT orchestration, language detection, glossary matching, and quality scoring — all of which have the richest library support in Python (provider SDKs for Google/DeepL/Azure/Amazon, `lingua`/`fasttext` for detection, `langcodes` for BCP 47). |
| API framework | **FastAPI** | Async-native (translation calls are I/O-bound), first-class **OpenAPI 3.1** generation (a `standards.md` requirement), and Pydantic models double as validation and wire-format documentation. |
| ASGI server | **Uvicorn** behind **Gunicorn** | Standard production combination for FastAPI; multiple workers for CPU-bound glossary matching. |
| Data validation | **Pydantic v2** | Single source of truth for request/response schemas, config, and the OpenAPI spec. |
| Database | **PostgreSQL 16** | Data Model 1 requires referential integrity, multi-tenant row-level security, full-text search (`tsvector`), fuzzy glossary matching (`pg_trgm`), and JSONB escape hatches — all native to Postgres. |
| ORM / migrations | **SQLAlchemy 2.0 (async)** + **Alembic** | Async ORM matches FastAPI; Alembic gives version-controlled migrations (DDL is large and evolves per phase). |
| Cache / queue broker | **Redis 7** | Hot glossary-term cache, translation-memory cache, rate-limit counters, and Celery broker/result backend. |
| Task queue | **Celery** (Redis broker) | Webhook ingestion, async translation of long content, KB bulk translation, and nightly analytics aggregation are async workloads that must survive provider latency and retries. |
| Scheduler | **Celery Beat** | Nightly rollups into `*_daily` tables; outdated-article detection; quality-trend alert evaluation. |
| Language detection | **`lingua-py`** (primary) + provider auto-detect (fallback) | `lingua` is offline, fast, and accurate on short support messages; provider detection is the fallback when confidence is low. |
| Language codes | **`langcodes`** + **`language-data`** | Canonicalizes and validates BCP 47 tags, resolves fallbacks (e.g., `es-419` → `es`), maps ISO 639-1/3. |
| Translation providers | **Google Cloud Translation v3**, **DeepL**, **Azure AI Translator v3**, **Amazon Translate** | Multi-provider routing with per-language-pair quality/cost preferences and automatic failover (a differentiator vs. single-engine incumbents). All four support glossaries/custom terminology. |
| KB interchange | **`translate-toolkit`** for **XLIFF 2.2** | Standards-compliant import/export of KB article translation jobs. |
| MCP server | **`mcp` Python SDK** | Exposes `translate`, `detect_language`, `glossary_lookup`, `route_conversation` as MCP tools per `standards.md`. |
| Secrets | **Vault reference pattern** (env-injected in MVP, HashiCorp Vault / cloud KMS in prod) | DDL stores only `credentials_vault_ref`, never raw helpdesk credentials. |
| Frontend | **Next.js 15 (React, TypeScript)** dashboard | Operator/admin console for glossary management, quality dashboards, integration setup, and KB translation review. Agent-facing translation is injected into the existing helpdesk UI, so the dashboard is admin/analytics-only — kept thin and deferred to a late phase. |
| Containerisation | **Docker** + **docker-compose** | Self-hosted is a stated deployment mode; compose wires api + worker + postgres + redis for one-command local/self-hosted bring-up. |
| Testing | **pytest** + **pytest-asyncio** + **httpx** + **respx** (HTTP mocking) + **testcontainers** (real Postgres/Redis) | Unit + mocked-integration + real-integration tiers. |
| Code quality | **Ruff** (lint+format), **mypy** (strict), **pre-commit** | Consistency and type safety across a large codebase. |
| Package manager | **uv** | Fast, reproducible installs and lockfile. |
| Observability | **structlog** (JSON logs) + **OpenTelemetry** + **Prometheus** metrics | Translation latency, provider error rates, queue depth, and per-tenant cost must be observable from day one. |

### Project Structure

```
multi-language-customer-support/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .pre-commit-config.yaml
├── README.md
├── migrations/                      # Alembic revisions
│   └── versions/
├── src/
│   └── mlcs/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory
│       ├── config.py                # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py           # async engine + session, RLS tenant context
│       │   ├── base.py              # declarative base, mixins (TenantMixin, TimestampMixin)
│       │   └── models/              # SQLAlchemy models, one module per domain
│       │       ├── tenancy.py
│       │       ├── language.py
│       │       ├── integration.py
│       │       ├── conversation.py
│       │       ├── glossary.py
│       │       ├── knowledge_base.py
│       │       └── analytics.py
│       ├── schemas/                 # Pydantic request/response models
│       ├── api/
│       │   ├── deps.py              # auth, tenant resolution, pagination deps
│       │   ├── v1/
│       │   │   ├── router.py
│       │   │   ├── translate.py
│       │   │   ├── conversations.py
│       │   │   ├── glossaries.py
│       │   │   ├── knowledge_base.py
│       │   │   ├── analytics.py
│       │   │   ├── integrations.py
│       │   │   └── webhooks.py
│       │   └── mcp_server.py        # MCP tool surface
│       ├── core/                    # business logic (provider-agnostic)
│       │   ├── detection.py         # language detection
│       │   ├── glossary_engine.py   # term matching, do-not-translate, learning
│       │   ├── translation_service.py  # orchestration: detect→glossary→translate→score
│       │   ├── quality.py           # confidence scoring, flagging
│       │   └── routing.py           # language/skill-based conversation routing
│       ├── providers/               # translation provider adapters
│       │   ├── base.py              # TranslationProvider protocol
│       │   ├── google.py
│       │   ├── deepl.py
│       │   ├── azure.py
│       │   ├── amazon.py
│       │   └── registry.py          # provider selection + failover
│       ├── connectors/              # helpdesk integrations
│       │   ├── base.py              # HelpdeskConnector protocol
│       │   ├── zendesk.py
│       │   ├── salesforce.py
│       │   ├── freshdesk.py
│       │   └── webhook_verify.py    # signature verification per platform
│       ├── workers/
│       │   ├── celery_app.py
│       │   ├── tasks.py             # translate_message, sync_article, etc.
│       │   └── beat_schedules.py    # nightly aggregation, outdated detection
│       ├── analytics/
│       │   ├── aggregation.py       # daily rollups
│       │   └── alerts.py            # trend detection + alert firing
│       └── observability/
│           ├── logging.py
│           └── metrics.py
├── frontend/                        # Next.js admin/analytics dashboard (Phase 10)
└── tests/
    ├── conftest.py                  # fixtures: db, redis, tenant, sample data
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                    # sample webhooks, XLIFF files, provider responses
```

The structure is grouped by concern (api / core / providers / connectors / workers / analytics), so each phase adds modules without restructuring.

---

## Phase 1: Foundation — Project Skeleton, Config, Database Core

### Purpose
Establish the runnable skeleton: a FastAPI app, configuration, the async database layer, Alembic migrations, multi-tenant row-level security, the language/locale registry, and the Docker dev environment. Nothing translates yet, but every later phase plugs into these foundations. After this phase, `docker-compose up` yields a healthy API with a seeded language registry and passing migrations.

### Tasks

#### 1.1 — Project scaffolding & tooling

**What**: Create the repository skeleton with `pyproject.toml`, dependency groups, Ruff/mypy/pre-commit config, and a `Dockerfile` + `docker-compose.yml` running api + worker + postgres + redis.

**Design**:
- `pyproject.toml` declares dependencies (fastapi, uvicorn[standard], gunicorn, sqlalchemy[asyncio], asyncpg, alembic, pydantic, pydantic-settings, redis, celery, langcodes, lingua-language-detector, structlog) and dev group (pytest, pytest-asyncio, httpx, respx, testcontainers, ruff, mypy, pre-commit).
- `config.py`:
  ```python
  class Settings(BaseSettings):
      model_config = SettingsConfigDict(env_prefix="MLCS_", env_file=".env")
      database_url: PostgresDsn
      redis_url: RedisDsn
      environment: Literal["development", "staging", "production"] = "development"
      log_level: str = "INFO"
      default_provider: Literal["google", "deepl", "azure", "amazon"] = "google"
      low_confidence_threshold: float = 0.70   # below this → quality_flag = 'low_confidence'
      translation_cache_ttl_seconds: int = 300
  ```
- `docker-compose.yml` services: `postgres:16` (with `pg_trgm` available), `redis:7`, `api` (uvicorn), `worker` (celery), `beat` (celery-beat). Healthchecks on postgres/redis; api `depends_on` healthy postgres+redis.
- `Dockerfile` is multi-stage (uv build → slim runtime).

**Testing**:
- `Unit: Settings loads from env → correct typed values, defaults applied when unset`.
- `Unit: invalid MLCS_LOW_CONFIDENCE_THRESHOLD="abc" → ValidationError`.
- `E2E (compose): docker compose up → GET /health returns 200 {"status":"ok"} and DB connectivity verified`.

#### 1.2 — Async database layer + multi-tenant RLS

**What**: Async SQLAlchemy engine/session, declarative base with `TenantMixin` and `TimestampMixin`, and a request-scoped tenant context that sets `app.tenant_id` for PostgreSQL row-level security.

**Design**:
- `db/session.py` exposes `get_session()` (FastAPI dependency) yielding an `AsyncSession` and, before yielding, executing `SET LOCAL app.tenant_id = :tenant_id` from a `ContextVar` populated by auth middleware.
- `db/base.py`:
  ```python
  class TimestampMixin:
      created_at: Mapped[datetime] = mapped_column(server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(server_default=func.now(), onupdate=func.now())

  class TenantMixin:
      tenant_id: Mapped[UUID] = mapped_column(ForeignKey("tenants.id", ondelete="CASCADE"), index=True)
  ```
- RLS policy template applied in a migration to every tenant-scoped table:
  ```sql
  ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;
  CREATE POLICY tenant_isolation ON <t>
      USING (tenant_id = current_setting('app.tenant_id')::uuid);
  ```

**Testing**:
- `Integration (real Postgres via testcontainers): session sets app.tenant_id → query on a tenant-scoped table returns only that tenant's rows`.
- `Integration: two tenants insert rows → tenant A session cannot SELECT tenant B rows`.
- `Unit: TimestampMixin → updated_at changes on update, created_at stable`.

#### 1.3 — Tenancy & organization schema (Domain 1)

**What**: Alembic migration + SQLAlchemy models for `tenants`, `agents`, `teams`, `team_members`, `agent_languages` (per Data Model 1 §1).

**Design**: Use the DDL from `data-model-suggestion-1.md` §1 verbatim (UUID PKs, `plan_tier`/`role`/`proficiency` CHECK constraints, `UNIQUE (tenant_id, email)`). `primary_language_id` / `agent_languages.language_id` FKs are added in the migration that creates `languages` (Task 1.4), matching the model's deferred-FK approach.

**Testing**:
- `Integration: insert tenant + agent → row persists; duplicate (tenant_id,email) → IntegrityError`.
- `Integration: insert agent.role='invalid' → CheckViolation`.
- `Integration: delete tenant → cascades to agents, teams, team_members`.

#### 1.4 — Language & locale registry (Domain 2) + seed data

**What**: Migration + models for `languages`, `language_variants`, `tenant_languages`; a seed script populating the 60+ supported languages and common variants from CLDR/ISO 639 data.

**Design**:
- DDL from Data Model 1 §2. `bcp47_tag` is `UNIQUE`; `direction` CHECK `('ltr','rtl')`; `script` holds ISO 15924 codes (`Latn`, `Hans`, `Hant`, `Arab`).
- `seed_languages.py` uses `langcodes` to generate rows: `iso_639_1`, `iso_639_3`, `bcp47_tag`, `name_english`, `name_native`, `script`, `direction`. Seed regional variants (`es-419`, `es-ES`, `zh-Hans-CN`, `zh-Hant-TW`, `pt-BR`, `pt-PT`) and `language_variants` rows linking them to parents with `fallback_priority`.
- A `resolve_language(tag: str) -> Language` helper canonicalizes input tags (e.g., `EN_us` → `en-US`) and resolves fallbacks via `language_variants`.

**Testing**:
- `Unit: resolve_language("EN_us") → canonical "en-US" row`.
- `Unit: resolve_language("es-419") → returns es-419; fallback chain includes "es"`.
- `Integration: seed script → ≥60 languages, RTL flag correct for ar/he/fa, zh-Hant marked script='Hant'`.
- `Integration: duplicate bcp47_tag insert → IntegrityError`.

### Definition of Done
Migrations apply cleanly up/down; RLS verified; language registry seeded; `/health` green under compose; Ruff + mypy + tests pass.

---

## Phase 2: Translation Provider Abstraction

### Purpose
Build a uniform adapter layer over the four NMT providers so that the rest of the system is provider-agnostic. This is the lowest layer of the translation engine; the orchestration in Phase 3 depends on it. After this phase, the system can translate a string via any provider and return a normalized result with confidence, latency, token, and cost data.

### Tasks

#### 2.1 — Provider protocol & normalized result types

**What**: Define the `TranslationProvider` protocol and the normalized result/structures every adapter returns.

**Design**:
```python
@dataclass(frozen=True)
class TranslationRequest:
    text: str
    source_lang: str | None        # BCP 47; None = auto-detect
    target_lang: str               # BCP 47
    content_type: Literal["text", "html"] = "text"
    glossary_id: str | None = None # provider-side glossary handle, if any

@dataclass(frozen=True)
class TranslationResult:
    translated_text: str
    detected_source_lang: str
    provider: str
    model_version: str | None
    confidence: float | None       # 0.0–1.0 normalized
    latency_ms: int
    input_tokens: int | None
    cost_usd: Decimal | None
    raw_response: dict             # stored as JSONB (Suggestion 3 escape hatch)

class TranslationProvider(Protocol):
    name: str
    def supports(self, source: str, target: str) -> bool: ...
    async def translate(self, req: TranslationRequest) -> TranslationResult: ...
    async def detect(self, text: str) -> tuple[str, float]: ...  # (BCP47, confidence)
```
- Confidence normalization: providers that don't return a confidence (DeepL) get `None`; quality scoring (Phase 3) handles `None` via heuristics.
- Cost computed from a per-provider per-million-char price table in config.

**Testing**:
- `Unit: TranslationResult is frozen/immutable`.
- `Unit: cost computation for 5,000 chars at known rate → expected Decimal`.

#### 2.2 — Provider adapters (Google, DeepL, Azure, Amazon)

**What**: Implement the four adapters against the protocol, each mapping the provider's API to `TranslationResult`.

**Design**:
- `google.py`: Google Cloud Translation v3 `translateText`; maps `glossaryConfig`; auth via service-account credentials (`credentials_vault_ref`).
- `deepl.py`: `DeepL-Auth-Key` header; supports `tag_handling=html`; no confidence → `None`.
- `azure.py`: `Ocp-Apim-Subscription-Key`; `/translate?api-version=3.0`; returns `detectedLanguage.score` as confidence.
- `amazon.py`: boto3 `translate_text`; SigV4 auth; custom terminology by name.
- BCP 47 → provider code mapping per adapter (e.g., Azure uses `zh-Hans`, Amazon uses `zh`).
- All external HTTP via shared `httpx.AsyncClient` with timeout + retry (tenacity, exponential backoff, max 3).

**Testing**:
- `Integration (mocked via respx): Google 200 response → TranslationResult with confidence, latency_ms>0, raw_response preserved`.
- `Integration (mocked): DeepL 200 → confidence is None, html tag_handling sent when content_type=html`.
- `Integration (mocked): Azure 429 → retried then surfaces ProviderRateLimitError after 3 attempts`.
- `Integration (mocked): Amazon UnsupportedLanguagePairException → ProviderUnsupportedError`.
- `Integration (real, opt-in via env keys, marked @pytest.mark.real): translate "Hello" en→es → non-empty Spanish text`.

#### 2.3 — Provider registry, routing & failover

**What**: Select a provider per request based on tenant preference and per-language-pair capability, with failover to the next provider on error.

**Design**:
- `registry.py`:
  ```python
  class ProviderRegistry:
      def select(self, tenant: Tenant, source: str, target: str) -> list[TranslationProvider]:
          # ordered: tenant_languages.preferred_provider → default_provider → others that support()
      async def translate_with_failover(self, providers, req) -> TranslationResult:
          # try in order; on ProviderError, log + try next; raise AllProvidersFailedError if exhausted
  ```
- Records each attempt (success/failure, latency) for `provider_performance` (Phase 8).

**Testing**:
- `Unit: select() orders preferred provider first, drops providers whose supports()=False`.
- `Integration (mocked): first provider 500, second 200 → result from second, failure logged`.
- `Integration (mocked): all providers fail → AllProvidersFailedError`.

### Definition of Done
All four adapters implemented and mocked-tested; registry failover verified; cost/latency captured; Ruff + mypy + tests pass.

---

## Phase 3: Core Translation Engine

### Purpose
Assemble the heart of the product: a translation service that detects language, applies the glossary, translates via the provider layer, scores quality, and persists messages and translations. This is the core value proposition and ships here. After this phase, a `POST /v1/translate` call performs the full detect→glossary→translate→score→persist pipeline.

### Tasks

#### 3.1 — Language detection

**What**: Detect the source language of a message with a confidence score, preferring offline `lingua`, falling back to provider auto-detect for low-confidence/short inputs.

**Design**:
- `detection.py`:
  ```python
  async def detect_language(text: str, registry: ProviderRegistry,
                            min_confidence: float = 0.65) -> DetectionResult
  # 1. lingua detect → (iso639_1, confidence)
  # 2. if confidence < min_confidence or len(text) < 12: provider.detect() tiebreak
  # 3. canonicalize to BCP47 via langcodes; resolve registry Language row
  ```
- `DetectionResult(language_id, bcp47_tag, confidence, method: Literal["lingua","provider","fallback"])`.

**Testing**:
- `Unit: detect_language("Bonjour, j'ai un problème") → fr, method=lingua, confidence high`.
- `Unit: detect_language("ok") (ambiguous short) → provider fallback path invoked`.
- `Unit: emoji/numeric-only input → low confidence, method=fallback, default to tenant source lang`.

#### 3.2 — Glossary engine (matching & application)

**What**: Apply per-tenant glossary terms to source text before translation (do-not-translate placeholders) and/or to provider glossary configs, with Redis-cached term lookup.

**Design**:
- `glossary_engine.py`:
  ```python
  def apply_glossary(text, glossary_terms, source, target) -> tuple[str, list[Placeholder]]
  # exact/prefix/regex matches per glossary_terms.match_type
  # is_do_not_translate=True → wrap term in sentinel placeholder, restore after translation
  # else → pass target_term to provider glossary, or post-substitute
  def restore_placeholders(translated, placeholders) -> str
  ```
- Terms loaded via `glossary_terms` filtered by `(glossary_id, source_language_id, target_language_id)`, ordered by `match_type` precedence (exact > prefix > fuzzy via `pg_trgm` > regex). Cached in Redis keyed `gloss:{tenant}:{src}:{tgt}` with TTL from config; invalidated on term write.
- Increments `usage_count` / `last_used_at` asynchronously.

**Testing**:
- `Unit: do-not-translate term "AcmeCloud" → placeholder out, exact string restored after`.
- `Unit: exact match preferred over fuzzy when both match`.
- `Integration (real Redis): second lookup served from cache; term update invalidates cache`.

#### 3.3 — Quality scoring & flagging

**What**: Produce a normalized confidence and a `quality_flag` per Data Model 1's `message_translations.quality_flag` enum.

**Design**:
- `quality.py`:
  ```python
  def score(result: TranslationResult, source_len: int) -> QualityAssessment
  # provider confidence if present; else heuristic from length ratio,
  # untranslated-segment detection (target≈source), and glossary-term hit rate
  # flag = 'low_confidence' if score < settings.low_confidence_threshold else 'ok'
  ```
- `QualityAssessment(confidence: float, flag: Literal["ok","low_confidence","needs_review"])`.

**Testing**:
- `Unit: confidence below threshold → flag='low_confidence'`.
- `Unit: target text identical to source (untranslated) → flag='needs_review'`.
- `Unit: DeepL None confidence + reasonable length ratio → heuristic confidence assigned`.

#### 3.4 — Conversation & message schema (Domain 4)

**What**: Migration + models for `customers`, `conversations`, `messages`, `message_translations`, `message_attachments` (Data Model 1 §4).

**Design**: DDL from §4 verbatim, including `message_translations` `UNIQUE (message_id, target_language_id)`, the partial index on `quality_flag != 'ok'`, and `sequence_number` per conversation. Add a `raw_response JSONB` column to `message_translations` (Suggestion 3) to store the provider's full payload for forensics.

**Testing**:
- `Integration: insert conversation→message→translation chain; cascade delete from conversation removes all`.
- `Integration: two translations same (message_id,target_language_id) → IntegrityError`.
- `Integration: csat_score=6 → CheckViolation`.

#### 3.5 — Translation orchestration service + `POST /v1/translate`

**What**: Wire detection + glossary + provider + scoring into one service, exposed via a synchronous translate endpoint and reused by the async pipeline.

**Design**:
- `translation_service.py`:
  ```python
  async def translate_message(tenant, text, source_lang, target_lang,
                              glossary_id=None, persist_message_id=None) -> StoredTranslation
  # 1. detect if source_lang is None
  # 2. translation-memory check (Phase 9 hook; no-op until then)
  # 3. apply_glossary
  # 4. registry.translate_with_failover
  # 5. restore_placeholders
  # 6. quality.score
  # 7. if persist_message_id: write message_translations row
  ```
- Endpoint:
  ```
  POST /v1/translate
  Request:  { "text": str, "source_lang": str|null, "target_lang": str,
              "content_type": "text"|"html", "glossary_id": str|null }
  Response: { "translated_text": str, "detected_source_lang": str, "target_lang": str,
              "provider": str, "confidence": float|null, "quality_flag": str,
              "latency_ms": int, "cost_usd": str }
  ```
- BCP 47 tags validated on input (422 on malformed tag).

**Testing**:
- `Integration (mocked providers): POST /v1/translate {source_lang:null} → detection runs, response includes detected_source_lang`.
- `Integration: malformed target_lang "xx_ZZ_bad" → 422`.
- `Integration: glossary do-not-translate term preserved end-to-end in response`.
- `E2E (mocked providers + real DB): translate with persist_message_id → message_translations row written with correct confidence and quality_flag`.

### Definition of Done
Full pipeline works end-to-end; `/v1/translate` documented in OpenAPI; quality flags persisted; tests pass across tiers.

---

## Phase 4: API Surface, Auth & Conversations

### Purpose
Turn the engine into a usable multi-tenant API: authentication, tenant resolution, conversation/message CRUD, and agent-facing original+translated retrieval. After this phase, an external system can authenticate, create conversations, post messages, and read both original and translated content.

### Tasks

#### 4.1 — Authentication & tenant resolution

**What**: API-key auth (per-tenant keys) plus OAuth 2.0 bearer support, resolving the tenant and populating the RLS context.

**Design**:
- `api/deps.py`:
  ```python
  async def current_tenant(authorization: str = Header(...)) -> Tenant
  # API key: "ApiKey <key>" → hash lookup in api_keys table → tenant
  # OAuth:  "Bearer <jwt>" → validate signature/claims → tenant
  # sets tenant_id ContextVar consumed by get_session() for RLS
  ```
- New migration: `api_keys (id, tenant_id, key_hash, name, scopes JSONB, last_used_at, revoked_at)`.
- Per-key rate limiting via Redis token bucket.

**Testing**:
- `Integration: valid API key → tenant resolved, RLS context set`.
- `Integration: revoked/unknown key → 401`.
- `Integration: rate limit exceeded → 429 with Retry-After`.

#### 4.2 — Conversation & message endpoints

**What**: CRUD for conversations and messages, auto-translating inbound messages on create.

**Design**:
```
POST   /v1/conversations              create (customer, channel, external_ticket_id)
GET    /v1/conversations/{id}         conversation + messages (paginated)
POST   /v1/conversations/{id}/messages
       { sender_type, content, content_type, source_lang?, target_lang? }
       → persists message, enqueues/returns translation (sync for chat, async for email)
GET    /v1/conversations/{id}/messages?include=translations
```
- On customer message create, run detection → set `conversations.customer_language_id`, translate into `agent_language_id`. On agent message, translate into `customer_language_id`.
- Cursor pagination on message lists.

**Testing**:
- `Integration: create conversation + customer message → message persisted, translation row created in agent language`.
- `Integration: agent reply → translated into customer language`.
- `Integration: GET conversation across tenants → RLS blocks cross-tenant access (404)`.

#### 4.3 — Agent-facing dual-text view & OpenAPI 3.1

**What**: A response shape that returns original + translated text side by side (a table-stakes UX), and a finalized, committed OpenAPI 3.1 document.

**Design**:
- Message response:
  ```json
  { "id": "...", "sender_type": "customer", "sequence_number": 3,
    "original": { "text": "...", "lang": "ja" },
    "translations": [ { "lang": "en", "text": "...", "confidence": 0.91,
                        "quality_flag": "ok", "provider": "deepl" } ] }
  ```
- FastAPI auto-generates `/openapi.json` (OpenAPI 3.1); a CI check exports and diff-tests it; tag descriptions reference relevant features.

**Testing**:
- `Integration: message response contains both original and translations blocks`.
- `Unit: generated OpenAPI is valid 3.1 (schema-validate the document)`.

### Definition of Done
Auth + RLS enforced on all endpoints; conversation/message flows translate correctly; OpenAPI 3.1 committed and validated; tests pass.

---

## Phase 5: Glossary Management & Self-Improving Learning

### Purpose
Deliver the brand-glossary differentiator and the self-improving learning loop. After this phase, tenants manage glossaries via API, agents submit corrections, and the system promotes recurring corrections into glossary terms automatically.

### Tasks

#### 5.1 — Glossary schema & CRUD (Domain 5)

**What**: Migration + models for `glossaries`, `glossary_terms`, `glossary_corrections`, `industry_term_packs`, `industry_pack_terms` (Data Model 1 §5), plus CRUD endpoints.

**Design**: DDL from §5 verbatim. Endpoints:
```
POST/GET/PATCH/DELETE /v1/glossaries
POST/GET/PATCH/DELETE /v1/glossaries/{id}/terms
POST /v1/glossaries/{id}/terms:import    (CSV / TBX upload)
POST /v1/glossaries/{id}:apply-pack      (clone an industry pack's terms)
```
- Provider-side glossary sync: on term changes, push to Google/Azure/Amazon glossary resources where supported; store the handle.

**Testing**:
- `Integration: create glossary + terms → persisted; duplicate (glossary,name) → IntegrityError`.
- `Integration: CSV import of 100 terms → 100 rows, source='imported'`.
- `Integration: apply industry pack → terms copied with source='industry_pack'`.

#### 5.2 — Agent corrections capture

**What**: Endpoint for agents to correct a machine translation, persisting to `glossary_corrections`.

**Design**:
```
POST /v1/translations/{message_translation_id}/corrections
  { original_segment, machine_translation, agent_correction, correction_type }
```
- Sets `message_translations.quality_flag='corrected'` and `reviewed_by_agent_id`.

**Testing**:
- `Integration: submit correction → glossary_corrections row, source translation flagged 'corrected'`.
- `Integration: correction on another tenant's translation → 404 (RLS)`.

#### 5.3 — Self-improving glossary learner

**What**: A Celery task that mines `glossary_corrections` to detect recurring terminology fixes and proposes/promotes them into `glossary_terms`.

**Design**:
- `glossary_engine.learn()` (scheduled nightly):
  1. Group corrections by `(source_lang, target_lang, normalized source segment span)`.
  2. Where the same source span maps to the same agent correction ≥ N times (config `MLCS_GLOSSARY_PROMOTE_THRESHOLD`, default 3), extract the term pair.
  3. Insert a `glossary_terms` row with `source='auto_learned'`, `confidence` ∝ agreement, `approved_at=NULL` (pending review) and link `glossary_corrections.promoted_to_term_id`.
- Auto-learned terms are *suggested* until an admin approves (sets `approved_by`), unless tenant opts into auto-approval.

**Testing**:
- `Unit: 3 identical corrections of "dashboard"→"tablero" → one auto_learned term proposed`.
- `Unit: below-threshold corrections → no promotion`.
- `Integration: promoted term links back via promoted_to_term_id`.

### Definition of Done
Glossary CRUD + import + packs work; corrections captured; learner promotes recurring corrections; tests pass.

---

## Phase 6: Helpdesk Connectors & Webhooks

### Purpose
Realize the middleware positioning: connect to real helpdesks, ingest inbound messages via webhooks, translate, and push translated content back. After this phase, a Zendesk/Salesforce/Freshdesk ticket flows through the translation pipeline automatically.

### Tasks

#### 6.1 — Integration & channel schema (Domain 3)

**What**: Migration + models for `integrations`, `channels`, `webhooks` (Data Model 1 §3).

**Design**: DDL from §3 verbatim. `credentials_vault_ref` stores only a secret-manager reference; `webhook_secret` used for signature verification. `integrations.config JSONB` (Suggestion 3) holds platform-specific settings.

**Testing**:
- `Integration: create integration with platform='zendesk' → persisted; platform='unknown' → CheckViolation`.
- `Integration: credentials never stored raw (only vault ref column present)`.

#### 6.2 — Connector protocol + webhook signature verification

**What**: `HelpdeskConnector` protocol and per-platform inbound webhook signature verification.

**Design**:
```python
class HelpdeskConnector(Protocol):
    platform: str
    def verify_signature(self, headers, body, secret) -> bool: ...
    def parse_inbound(self, payload: dict) -> InboundMessage: ...
    async def push_translation(self, integration, external_ticket_id, text, lang): ...
    async def fetch_oauth_token(self, integration) -> str: ...
```
- `InboundMessage(external_ticket_id, customer_external_id, sender_type, content, content_type, channel_type)`.
- Zendesk: HMAC-SHA256 over body with `webhook_secret`; Salesforce: OAuth + signed callbacks; Freshdesk: shared-secret header.

**Testing**:
- `Unit: Zendesk valid HMAC → True; tampered body → False`.
- `Unit: parse_inbound maps Zendesk ticket JSON → InboundMessage`.
- `Fixture-based: committed sample webhook payloads per platform parse correctly`.

#### 6.3 — Webhook ingestion endpoint + async translation pipeline

**What**: `POST /v1/webhooks/{integration_id}` that verifies, enqueues a Celery task, and returns 200 fast; the task translates and pushes back.

**Design**:
- Endpoint verifies signature (401 on failure), persists the inbound message, enqueues `translate_and_push.delay(...)`, returns `202 {"accepted": true}` within ~50ms.
- `tasks.translate_and_push`: load conversation context → `translate_message` → `connector.push_translation` (translated text back to customer / agent) → record metrics. Idempotent on `(integration_id, external_event_id)` to survive retries.

**Testing**:
- `Integration (mocked connector): valid webhook → 202, task enqueued, message persisted`.
- `Integration: invalid signature → 401, nothing enqueued`.
- `Integration: duplicate event id → processed once (idempotency)`.
- `E2E (mocked Zendesk API + real DB/Redis + eager Celery): inbound ticket → translated message pushed back via connector.push_translation`.

#### 6.4 — Connector implementations (Zendesk, Salesforce, Freshdesk)

**What**: Implement the three MVP connectors against the protocol with OAuth 2.0 where applicable.

**Design**: Each maps inbound payloads and implements `push_translation` against the platform REST API (Zendesk comments API, Salesforce Case feed, Freshdesk reply API). OAuth token refresh cached in Redis.

**Testing**:
- `Integration (mocked, per connector): push_translation issues correct authenticated API call`.
- `Integration: expired OAuth token → refresh then retry`.

### Definition of Done
Three connectors verify, parse, translate, and push; webhook ingestion is fast + idempotent; tests pass.

---

## Phase 7: Multilingual Knowledge Base

### Purpose
Add automated KB localization with XLIFF interchange, translation status tracking, and outdated-article detection. After this phase, tenants can translate help articles into many languages, track which translations are stale, and export/import jobs as XLIFF 2.2.

### Tasks

#### 7.1 — KB schema (Domain 6)

**What**: Migration + models for `kb_articles`, `kb_article_translations`, `kb_article_feedback` (Data Model 1 §6).

**Design**: DDL from §6 verbatim, including `last_modified_hash` (SHA-256 of body) for change detection, `is_outdated` partial index, and the GIN index on `tags`. Add `tsvector` columns for language-aware full-text search.

**Testing**:
- `Integration: article + translation; unique (article,language) enforced`.
- `Integration: full-text search on a translation finds it by localized term`.

#### 7.2 — Article translation pipeline + status lifecycle

**What**: Translate an article into target languages, tracking the status lifecycle.

**Design**:
- Status machine (per §6 enum): `draft → machine_translated → human_reviewed → published`, plus `outdated` and `archived`.
- `POST /v1/kb/articles/{id}/translate { target_langs: [...] }` enqueues per-language translation tasks producing `machine_translated` rows recording `source_version`.

**Testing**:
- `Integration: translate article into [es,ja] → two translations, status='machine_translated', source_version set`.
- `Unit: invalid transition published→draft rejected`.

#### 7.3 — Outdated detection & XLIFF 2.2 import/export

**What**: Nightly job flags translations whose source article changed; XLIFF endpoints for round-tripping with external CAT tools.

**Design**:
- Beat task compares each `kb_articles.last_modified_hash` to `kb_article_translations.source_version`; mismatch → `is_outdated=true`, status `outdated`, emits a feedback/alert hook.
- `GET /v1/kb/articles/{id}/xliff?target_lang=fr` exports XLIFF 2.2 (via `translate-toolkit`); `POST .../xliff` imports a reviewed file → `human_reviewed`.

**Testing**:
- `Integration: edit source article → translations marked outdated next run`.
- `Fixture-based: export XLIFF → valid XLIFF 2.2 (schema-validated); re-import → human_reviewed`.

### Definition of Done
KB translation lifecycle works; outdated detection fires; XLIFF round-trips validate; tests pass.

---

## Phase 8: Analytics & Quality Monitoring

### Purpose
Deliver the per-language analytics and the proactive quality-degradation alerting that incumbents lack. After this phase, operators see per-language CSAT/volume/quality trends and receive alerts before quality issues hit CSAT.

### Tasks

#### 8.1 — Analytics schema & nightly aggregation (Domain 7)

**What**: Migration + models for `translation_quality_daily`, `conversation_metrics_daily`, `provider_performance`; Celery Beat rollup jobs.

**Design**: DDL from Data Model 1 §7. `aggregation.py` rolls up the prior day's `message_translations` and `conversations` into the daily tables (avg/min confidence, low-confidence counts, corrections, latency, tokens, cost; CSAT, FCR, resolution time, escalations, sentiment per language). Upserts on the unique keys for idempotent re-runs.

**Testing**:
- `Integration: seed a day of translations → rollup produces correct avg_confidence, total_cost_usd, low_confidence_count`.
- `Integration: re-run rollup → idempotent (no duplicates)`.

#### 8.2 — Analytics API

**What**: Read endpoints for per-language dashboards.

**Design**:
```
GET /v1/analytics/quality?from&to&source_lang&target_lang&provider   → trend series
GET /v1/analytics/conversations?from&to&language                     → CSAT/FCR/volume by language
GET /v1/analytics/providers?from&to                                  → provider comparison
GET /v1/analytics/cost?from&to&group_by=language_pair|provider       → cost breakdown
```

**Testing**:
- `Integration: quality endpoint returns daily series within range`.
- `Integration: cost grouped by language_pair sums correctly`.

#### 8.3 — Quality degradation alerts

**What**: Detect trend degradation and write `quality_alerts`; expose ack endpoints.

**Design**: `alerts.py` (scheduled) computes rolling baselines per `(tenant, language pair, provider)` and fires alerts (Data Model 1 `quality_alerts` enum: `confidence_drop`, `correction_spike`, `latency_increase`, `cost_anomaly`, `csat_decline`, `volume_spike`) when the latest window deviates beyond a configurable z-score/threshold. `GET /v1/alerts`, `POST /v1/alerts/{id}:ack`.

**Testing**:
- `Unit: 7-day avg confidence 0.9 then 0.6 today → confidence_drop, severity='critical'`.
- `Unit: stable metrics → no alert`.
- `Integration: ack alert → is_acknowledged=true, acknowledged_by set`.

### Definition of Done
Daily rollups accurate + idempotent; analytics endpoints return correct series; alerts fire on degradation and can be acknowledged; tests pass.

---

## Phase 9: Translation Memory & MCP Server (AI-Native Layer)

### Purpose
Add the cost-saving translation-memory layer (from Data Model Suggestion 4) and the MCP server that exposes the platform's capabilities to AI agents (a `standards.md` priority). After this phase, repeated questions reuse approved translations, and external AI assistants can invoke translate/glossary/route tools natively.

### Tasks

#### 9.1 — Translation memory with pgvector

**What**: Reuse prior high-confidence, agent-approved translations for semantically equivalent source segments, scoped per tenant.

**Design**:
- Enable `pgvector`; add `translation_memory (id, tenant_id, source_lang, target_lang, source_text, target_text, embedding vector(1024), confidence, approved, created_at)` with an HNSW index.
- Embed source segments with a multilingual embedding model (`multilingual-e5-large` via a local/Hosted embedding service).
- In `translation_service.translate_message` step 2 (the Phase 3 hook): cosine-similarity search ≥ `MLCS_TM_SIMILARITY_THRESHOLD` (default 0.92) among approved entries → reuse target text, skip provider call, mark provider `internal`.

**Testing**:
- `Integration (real pgvector): store approved "How do I reset my password?"→ja; near-identical query reuses it (provider not called)`.
- `Integration: below-threshold similarity → falls through to provider`.

#### 9.2 — MCP server

**What**: Expose `translate`, `detect_language`, `glossary_lookup`, `route_conversation` as MCP tools.

**Design**: `api/mcp_server.py` using the `mcp` SDK; each tool wraps the corresponding `core/` service; auth via per-tenant API key passed in the MCP session; tool schemas mirror the Pydantic request models.

**Testing**:
- `Integration: MCP client lists tools → 4 tools with correct JSON schemas`.
- `Integration: invoke translate tool → same result as POST /v1/translate`.

### Definition of Done
Translation memory reuses approved translations and reduces provider calls; MCP server lists and executes tools; tests pass.

---

## Phase 10: Admin Dashboard, Compliance & Hardening

### Purpose
Ship the operator-facing dashboard and the compliance/audit features required for enterprise adoption (GDPR, SOC 2, ISO 42001 posture). After this phase, the product is demonstrable end-to-end and enterprise-evaluable.

### Tasks

#### 10.1 — Audit log & GDPR data-subject requests (Domain 8)

**What**: Migration + models for `audit_log` and `data_subject_requests` (Data Model 1 §8); audit middleware; DSAR endpoints.

**Design**: DDL from §8 (monthly-partitioned `audit_log`). Middleware writes audit entries for translation, glossary, integration, and admin actions. `POST /v1/dsr { customer_id, request_type }`; erasure follows FKs and CASCADE deletes; access/portability export the customer's conversations, messages, and translations as JSON.

**Testing**:
- `Integration: translate action → audit_log entry with actor/resource/action`.
- `Integration: erasure request → customer + conversations + messages + translations removed; request marked completed`.

#### 10.2 — Next.js admin/analytics dashboard

**What**: A thin Next.js console for integrations setup, glossary management, KB translation review, quality dashboards, and alerts.

**Design**: Next.js 15 App Router + TypeScript; calls the v1 REST API with a tenant API key; pages: Integrations, Glossaries (incl. pending auto-learned term approvals), Knowledge Base (translation status + outdated), Analytics (per-language charts), Alerts. Agent-facing dual-text translation remains injected into the helpdesk via connectors, not duplicated here.

**Testing**:
- `E2E (Playwright, mocked API): glossary CRUD flow; approve auto-learned term; acknowledge an alert`.
- `Component: analytics chart renders provided series`.

#### 10.3 — Production hardening

**What**: Rate limiting, secret-manager wiring, structured logging, OpenTelemetry traces, Prometheus metrics, and security review.

**Design**: Per-tenant + per-key rate limits; `credentials_vault_ref` resolved via HashiCorp Vault / cloud KMS in production; spans across the translate pipeline (detect→glossary→provider→score); metrics: `translation_latency_ms`, `provider_errors_total`, `queue_depth`, `cost_usd_total`; dependency and SAST scan in CI.

**Testing**:
- `Integration: traces emitted for a translate request with provider span`.
- `Integration: /metrics exposes translation + provider counters`.
- `Security: secret values never appear in logs or API responses (assert on captured logs)`.

### Definition of Done
Audit + DSAR functional; dashboard exercises all major flows; rate limiting, tracing, metrics, and secret management in place; security scan clean; tests pass.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (skeleton, DB, RLS, languages)   ─── required by everything
    │
Phase 2: Provider Abstraction                        ─── requires Phase 1
    │
Phase 3: Core Translation Engine                     ─── requires Phases 1, 2   ◀ core value ships
    │
Phase 4: API, Auth & Conversations                   ─── requires Phase 3
    │
    ├── Phase 5: Glossary & Self-Learning             ─── requires Phase 3   ┐
    ├── Phase 6: Helpdesk Connectors & Webhooks       ─── requires Phase 4   ├ can parallel
    └── Phase 7: Knowledge Base                       ─── requires Phase 3   ┘
         │
Phase 8: Analytics & Quality Monitoring              ─── requires Phases 3, 6 (volume of data)
    │
Phase 9: Translation Memory & MCP Server             ─── requires Phases 3, 5
    │
Phase 10: Dashboard, Compliance & Hardening          ─── requires Phases 5, 6, 7, 8
```

**Parallelism opportunities**: After Phase 4, Phases 5, 6, and 7 are independent and can be built concurrently by separate developers. Phase 8 needs Phase 6's data volume to be meaningful but can be scaffolded earlier. Phase 9's two tasks (TM, MCP) are independent of each other.

---

## Definition of Done (per phase)

Every phase must satisfy all of the following before it is considered complete:

1. All tasks in the phase implemented.
2. All unit and mocked-integration tests pass; real-integration tests pass where credentials are available (`@pytest.mark.real`).
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict` passes.
5. Alembic migrations apply cleanly in both directions (`upgrade head` / `downgrade`), and RLS policies are present on every new tenant-scoped table.
6. `docker compose up` builds and runs; `/health` is green.
7. The phase's feature works end-to-end against the compose stack (real Postgres/Redis, mocked external providers/helpdesks).
8. New/changed API endpoints appear in the generated OpenAPI 3.1 document, which validates.
9. New config options are documented in `README.md` / `.env.example` with defaults.
10. New external-facing behavior is reflected in audit logging where applicable, and no secrets appear in logs or responses.
```