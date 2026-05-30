# Data Model Suggestion 4: Hybrid PostgreSQL + Vector Search (pgvector) + Time-Series Specialty Model

> Project: Multi-Language Customer Support (Candidate #475)
> Generated: 2026-05-26

## Overview

This model uses a domain-specific specialty approach that combines three storage paradigms, each chosen for the workload it serves best:

1. **PostgreSQL (relational core)** -- tenants, agents, conversations, glossaries, and the transactional backbone
2. **pgvector (vector similarity search)** -- multilingual semantic search across knowledge base articles, translation memory for segment reuse, and cross-lingual message similarity for intelligent routing and duplicate detection
3. **TimescaleDB (time-series)** -- translation quality metrics, per-language analytics, provider performance telemetry, and cost tracking with native continuous aggregation

This combination addresses three weaknesses of a purely relational model that are particularly acute in the multilingual customer support domain:

- **Cross-lingual search.** A customer searching for help in Japanese needs to find articles written in English and machine-translated. Keyword search fails across languages; vector embeddings from a multilingual model (e.g., `multilingual-e5-large` or `cohere-multilingual-v3`) place semantically equivalent text near each other regardless of language. pgvector makes this a PostgreSQL query.

- **Translation memory reuse.** When the same customer asks the same question ("How do I reset my password?") that was previously translated from English to Japanese with a high confidence score and agent approval, the system should reuse the prior translation rather than calling an external provider. Vector similarity over source segments, scoped to the tenant's translation history, implements translation memory without a separate TMS.

- **Time-series analytics at scale.** Translation quality metrics (confidence scores, latency, cost) arrive continuously and are queried primarily by time range. TimescaleDB's hypertables, continuous aggregates, and compression provide order-of-magnitude storage and query performance improvements over plain PostgreSQL tables for this workload.

---

## Architecture Diagram

```
+-------------------------------------------------------------+
|                     PostgreSQL Cluster                       |
|                                                              |
|  +------------------+  +------------------+  +-----------+   |
|  | Relational Core  |  | pgvector         |  | Timescale |   |
|  | (standard tables)|  | (vector indexes) |  | (hyper-   |   |
|  |                  |  |                  |  |  tables)  |   |
|  | tenants          |  | kb_embeddings    |  | tx_qual   |   |
|  | agents           |  | tm_embeddings    |  | conv_met  |   |
|  | conversations    |  | msg_embeddings   |  | prov_perf |   |
|  | messages         |  |                  |  | cost_log  |   |
|  | glossary_terms   |  |                  |  |           |   |
|  | integrations     |  |                  |  |           |   |
|  +------------------+  +------------------+  +-----------+   |
|                                                              |
+-------------------------------------------------------------+
              |                     |                |
       Transactional          Similarity         Time-range
       CRUD / JOINs          Search / KNN        Aggregation

All three paradigms live in the same PostgreSQL instance (pgvector and
TimescaleDB are PostgreSQL extensions). Single connection pool, single
backup strategy, single operational surface.
```

---

## Schema Definition

### 1. Relational Core (Standard PostgreSQL)

The relational tables are identical to those in the hybrid relational + JSONB model (Suggestion 3), so only the key tables are summarized here. The full definitions for tenants, agents, teams, languages, integrations, channels, customers, conversations, messages, glossaries, and glossary_terms are the same. The differences in this model are the addition of the vector and time-series tables below.

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS vector;       -- pgvector
CREATE EXTENSION IF NOT EXISTS timescaledb;  -- TimescaleDB

-- =====================================================
-- TENANCY (abbreviated -- same as Suggestion 3)
-- =====================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'starter',
    data_residency  VARCHAR(10) NOT NULL DEFAULT 'us',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- LANGUAGES (abbreviated -- same as Suggestion 3)
-- =====================================================

CREATE TABLE languages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    iso_639_1       CHAR(2),
    iso_639_3       CHAR(3),
    bcp47_tag       VARCHAR(35) NOT NULL UNIQUE,
    name_english    VARCHAR(255) NOT NULL,
    name_native     VARCHAR(255),
    script          VARCHAR(10),
    direction       VARCHAR(3) NOT NULL DEFAULT 'ltr',
    locale_data     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- AGENTS (abbreviated -- same as Suggestion 3)
-- =====================================================

CREATE TABLE agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    primary_language_id UUID REFERENCES languages(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'agent',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_agents_tenant ON agents (tenant_id);

-- =====================================================
-- INTEGRATIONS AND CHANNELS (abbreviated)
-- =====================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    platform        VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    integration_id  UUID REFERENCES integrations(id) ON DELETE SET NULL,
    channel_type    VARCHAR(30) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =====================================================
-- CUSTOMERS
-- =====================================================

CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    email           VARCHAR(320),
    phone           VARCHAR(50),
    display_name    VARCHAR(255),
    detected_language_id UUID REFERENCES languages(id),
    preferred_language_id UUID REFERENCES languages(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_tenant ON customers (tenant_id);
CREATE INDEX idx_customers_external ON customers (tenant_id, external_id);

-- =====================================================
-- CONVERSATIONS
-- =====================================================

CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    channel_id      UUID REFERENCES channels(id),
    customer_id     UUID REFERENCES customers(id),
    assigned_agent_id UUID REFERENCES agents(id),
    assigned_team_id UUID,
    integration_id  UUID REFERENCES integrations(id),
    external_ticket_id VARCHAR(255),
    subject         TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'open',
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal',
    customer_language_id UUID REFERENCES languages(id),
    agent_language_id UUID REFERENCES languages(id),
    requires_translation BOOLEAN NOT NULL DEFAULT true,
    sentiment_score NUMERIC(4,3),
    csat_score      SMALLINT,
    first_response_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_conv_tenant ON conversations (tenant_id);
CREATE INDEX idx_conv_status ON conversations (tenant_id, status);
CREATE INDEX idx_conv_customer ON conversations (customer_id);
CREATE INDEX idx_conv_agent ON conversations (assigned_agent_id);
CREATE INDEX idx_conv_created ON conversations (tenant_id, created_at DESC);

-- =====================================================
-- MESSAGES
-- =====================================================

CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    sender_type     VARCHAR(20) NOT NULL,
    sender_agent_id UUID REFERENCES agents(id),
    sender_customer_id UUID REFERENCES customers(id),
    channel_id      UUID REFERENCES channels(id),
    original_content TEXT NOT NULL,
    original_language_id UUID NOT NULL REFERENCES languages(id),
    content_type    VARCHAR(20) NOT NULL DEFAULT 'text',
    is_internal_note BOOLEAN NOT NULL DEFAULT false,
    sequence_number INTEGER NOT NULL,
    channel_metadata JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_messages_conv ON messages (conversation_id, sequence_number);
CREATE INDEX idx_messages_tenant ON messages (tenant_id);

-- =====================================================
-- MESSAGE TRANSLATIONS
-- =====================================================

CREATE TABLE message_translations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    target_language_id UUID NOT NULL REFERENCES languages(id),
    translated_content TEXT NOT NULL,
    translation_provider VARCHAR(50) NOT NULL,
    confidence_score NUMERIC(5,4),
    quality_flag    VARCHAR(20) NOT NULL DEFAULT 'ok',
    latency_ms      INTEGER,
    token_count     INTEGER,
    cost_usd        NUMERIC(10,6),
    reviewed_by_agent_id UUID REFERENCES agents(id),
    reviewed_at     TIMESTAMPTZ,
    provider_details JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (message_id, target_language_id)
);

CREATE INDEX idx_msg_trans_message ON message_translations (message_id);
CREATE INDEX idx_msg_trans_quality ON message_translations (quality_flag)
    WHERE quality_flag != 'ok';

-- =====================================================
-- GLOSSARIES AND TERMS (abbreviated -- same as Suggestion 3)
-- =====================================================

CREATE TABLE glossaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    industry        VARCHAR(100),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE glossary_terms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    glossary_id     UUID NOT NULL REFERENCES glossaries(id) ON DELETE CASCADE,
    source_language_id UUID NOT NULL REFERENCES languages(id),
    target_language_id UUID NOT NULL REFERENCES languages(id),
    source_term     VARCHAR(1000) NOT NULL,
    target_term     VARCHAR(1000) NOT NULL,
    case_sensitive  BOOLEAN NOT NULL DEFAULT false,
    match_type      VARCHAR(20) NOT NULL DEFAULT 'exact',
    context_hint    TEXT,
    is_do_not_translate BOOLEAN NOT NULL DEFAULT false,
    confidence      NUMERIC(5,4) NOT NULL DEFAULT 1.0,
    source          VARCHAR(20) NOT NULL DEFAULT 'manual',
    approved_by     UUID REFERENCES agents(id),
    usage_count     INTEGER NOT NULL DEFAULT 0,
    last_used_at    TIMESTAMPTZ,
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_gt_lookup ON glossary_terms (glossary_id, source_language_id,
    target_language_id, source_term);

-- =====================================================
-- KNOWLEDGE BASE ARTICLES
-- =====================================================

CREATE TABLE kb_articles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    source_language_id UUID NOT NULL REFERENCES languages(id),
    title           VARCHAR(1000) NOT NULL,
    slug            VARCHAR(500),
    body            TEXT NOT NULL,
    body_format     VARCHAR(20) NOT NULL DEFAULT 'html',
    category        VARCHAR(255),
    tags            TEXT[],
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    version         INTEGER NOT NULL DEFAULT 1,
    content_hash    VARCHAR(64),
    metadata        JSONB NOT NULL DEFAULT '{}',
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_kb_tenant ON kb_articles (tenant_id);
CREATE INDEX idx_kb_status ON kb_articles (tenant_id, status);

CREATE TABLE kb_article_translations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID NOT NULL REFERENCES kb_articles(id) ON DELETE CASCADE,
    language_id     UUID NOT NULL REFERENCES languages(id),
    title           VARCHAR(1000) NOT NULL,
    body            TEXT NOT NULL,
    translation_method VARCHAR(30) NOT NULL DEFAULT 'machine',
    translation_provider VARCHAR(50),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    source_version  INTEGER NOT NULL,
    is_outdated     BOOLEAN NOT NULL DEFAULT false,
    quality_score   NUMERIC(5,4),
    reviewed_by     UUID REFERENCES agents(id),
    translation_details JSONB NOT NULL DEFAULT '{}',
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (article_id, language_id)
);
```

### 2. Vector Search Tables (pgvector)

```sql
-- =====================================================
-- KNOWLEDGE BASE EMBEDDINGS
-- Multilingual vector embeddings for cross-lingual article search.
-- A customer searching in Japanese finds articles written in English
-- because the embeddings are language-agnostic.
-- =====================================================

CREATE TABLE kb_article_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    article_id      UUID NOT NULL REFERENCES kb_articles(id) ON DELETE CASCADE,
    -- This can be the source article or a specific translation
    article_translation_id UUID REFERENCES kb_article_translations(id) ON DELETE CASCADE,
    language_id     UUID NOT NULL REFERENCES languages(id),
    -- Embedding model identifier for versioning
    embedding_model VARCHAR(100) NOT NULL,  -- e.g., 'multilingual-e5-large-v2'
    -- Section of the article this embedding covers (articles are chunked)
    chunk_index     INTEGER NOT NULL DEFAULT 0,
    chunk_text      TEXT NOT NULL,           -- the text that was embedded
    -- 1024-dimension vector (multilingual-e5-large produces 1024-dim embeddings)
    embedding       vector(1024) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Compound unique: one embedding per chunk per article per language per model
    UNIQUE (article_id, language_id, embedding_model, chunk_index)
);

-- HNSW index for approximate nearest neighbor search
-- HNSW gives better recall than IVFFlat at similar query speeds
CREATE INDEX idx_kb_embeddings_hnsw ON kb_article_embeddings
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 256);

-- Filter index for tenant-scoped searches
CREATE INDEX idx_kb_embeddings_tenant ON kb_article_embeddings (tenant_id);
CREATE INDEX idx_kb_embeddings_article ON kb_article_embeddings (article_id);


-- =====================================================
-- TRANSLATION MEMORY EMBEDDINGS
-- Vector representations of previously translated segments.
-- When a new message arrives for translation, search for similar
-- previously-translated segments to reuse high-quality translations.
-- =====================================================

CREATE TABLE translation_memory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    source_language_id UUID NOT NULL REFERENCES languages(id),
    target_language_id UUID NOT NULL REFERENCES languages(id),
    source_text     TEXT NOT NULL,
    target_text     TEXT NOT NULL,
    -- Quality indicators for ranking TM matches
    confidence_score NUMERIC(5,4) NOT NULL,
    quality_level   VARCHAR(20) NOT NULL DEFAULT 'machine'
                    CHECK (quality_level IN ('machine', 'reviewed', 'corrected', 'human')),
    -- Source of this TM entry
    source_message_id UUID REFERENCES messages(id),
    source_message_translation_id UUID REFERENCES message_translations(id),
    -- Usage tracking
    reuse_count     INTEGER NOT NULL DEFAULT 0,
    last_reused_at  TIMESTAMPTZ,
    -- Embedding of the SOURCE text (for similarity matching)
    embedding_model VARCHAR(100) NOT NULL,
    source_embedding vector(1024) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- HNSW index for translation memory lookup
CREATE INDEX idx_tm_hnsw ON translation_memory
    USING hnsw (source_embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 256);

-- Scoped lookups: find TM matches for a specific tenant and language pair
CREATE INDEX idx_tm_tenant_langs ON translation_memory (tenant_id, source_language_id,
    target_language_id);
CREATE INDEX idx_tm_quality ON translation_memory (quality_level);


-- =====================================================
-- MESSAGE EMBEDDINGS
-- Vector representations of customer messages for:
-- 1. Similar ticket detection (deduplicate repeat issues)
-- 2. Intelligent routing (match to agents who resolved similar issues)
-- 3. Sentiment clustering (find groups of frustrated customers)
-- =====================================================

CREATE TABLE message_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    -- Embedding of the message content (in its original language)
    -- Multilingual model ensures cross-lingual similarity works
    embedding_model VARCHAR(100) NOT NULL,
    embedding       vector(1024) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (message_id, embedding_model)
);

-- HNSW index for message similarity search
CREATE INDEX idx_msg_emb_hnsw ON message_embeddings
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 256);

CREATE INDEX idx_msg_emb_tenant ON message_embeddings (tenant_id);
CREATE INDEX idx_msg_emb_conv ON message_embeddings (conversation_id);


-- =====================================================
-- GLOSSARY TERM EMBEDDINGS
-- Vector representations of glossary terms for fuzzy matching.
-- When the exact term "CloudSync Pro" doesn't appear but "cloud sync
-- professional edition" does, vector similarity finds the match.
-- =====================================================

CREATE TABLE glossary_term_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    term_id         UUID NOT NULL REFERENCES glossary_terms(id) ON DELETE CASCADE,
    glossary_id     UUID NOT NULL REFERENCES glossaries(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    -- Embedding of the source term + context hint
    embedding_model VARCHAR(100) NOT NULL,
    source_embedding vector(1024) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (term_id, embedding_model)
);

CREATE INDEX idx_gte_hnsw ON glossary_term_embeddings
    USING hnsw (source_embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 256);

CREATE INDEX idx_gte_tenant ON glossary_term_embeddings (tenant_id);
CREATE INDEX idx_gte_glossary ON glossary_term_embeddings (glossary_id);
```

### 3. Time-Series Tables (TimescaleDB)

```sql
-- =====================================================
-- TRANSLATION TELEMETRY
-- Every translation request generates a telemetry row.
-- This is the raw event stream for translation pipeline monitoring.
-- TimescaleDB hypertable for time-range queries and compression.
-- =====================================================

CREATE TABLE translation_telemetry (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    message_id      UUID,
    conversation_id UUID,
    source_language VARCHAR(35) NOT NULL,     -- BCP 47 tag
    target_language VARCHAR(35) NOT NULL,
    provider        VARCHAR(50) NOT NULL,
    channel_type    VARCHAR(30),
    -- Metrics
    confidence_score NUMERIC(5,4),
    latency_ms      INTEGER NOT NULL,
    token_count     INTEGER NOT NULL DEFAULT 0,
    cost_usd        NUMERIC(10,6) NOT NULL DEFAULT 0,
    -- Outcome
    status          VARCHAR(20) NOT NULL DEFAULT 'success'
                    CHECK (status IN ('success', 'low_confidence', 'failed', 'cached', 'tm_reuse')),
    error_code      VARCHAR(50),
    -- Was a glossary applied? Was a TM match reused?
    glossary_terms_applied INTEGER NOT NULL DEFAULT 0,
    tm_match_used   BOOLEAN NOT NULL DEFAULT false,
    tm_match_score  NUMERIC(5,4)
);

-- Convert to TimescaleDB hypertable (partitioned by time, 1-day chunks)
SELECT create_hypertable('translation_telemetry', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Indexes for common query patterns
CREATE INDEX idx_tt_tenant_time ON translation_telemetry (tenant_id, time DESC);
CREATE INDEX idx_tt_langs ON translation_telemetry (source_language, target_language, time DESC);
CREATE INDEX idx_tt_provider ON translation_telemetry (provider, time DESC);
CREATE INDEX idx_tt_status ON translation_telemetry (status, time DESC)
    WHERE status != 'success';


-- =====================================================
-- CONVERSATION TELEMETRY
-- Per-conversation lifecycle events for CSAT, resolution time,
-- and per-language performance analytics.
-- =====================================================

CREATE TABLE conversation_telemetry (
    time            TIMESTAMPTZ NOT NULL,    -- event timestamp
    tenant_id       UUID NOT NULL,
    conversation_id UUID NOT NULL,
    language        VARCHAR(35) NOT NULL,    -- customer language
    channel_type    VARCHAR(30),
    event_type      VARCHAR(50) NOT NULL,    -- 'created', 'first_response', 'resolved',
                                             -- 'csat_submitted', 'escalated', 'reopened'
    -- Metrics (nullable; populated based on event_type)
    csat_score      SMALLINT,
    sentiment_score NUMERIC(4,3),
    first_response_seconds NUMERIC(12,2),
    resolution_seconds NUMERIC(12,2),
    message_count   INTEGER,
    translation_count INTEGER,
    is_fcr          BOOLEAN                  -- first-contact resolution
);

SELECT create_hypertable('conversation_telemetry', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_ct_tenant_time ON conversation_telemetry (tenant_id, time DESC);
CREATE INDEX idx_ct_language ON conversation_telemetry (language, time DESC);
CREATE INDEX idx_ct_event ON conversation_telemetry (event_type, time DESC);


-- =====================================================
-- PROVIDER HEALTH TELEMETRY
-- Track external translation provider availability, latency, and errors.
-- Used for provider routing decisions and SLA monitoring.
-- =====================================================

CREATE TABLE provider_telemetry (
    time            TIMESTAMPTZ NOT NULL,
    provider        VARCHAR(50) NOT NULL,
    language_pair   VARCHAR(75) NOT NULL,   -- 'en-US -> ja-JP'
    -- Health metrics
    request_count   INTEGER NOT NULL DEFAULT 0,
    success_count   INTEGER NOT NULL DEFAULT 0,
    error_count     INTEGER NOT NULL DEFAULT 0,
    timeout_count   INTEGER NOT NULL DEFAULT 0,
    rate_limit_count INTEGER NOT NULL DEFAULT 0,
    -- Latency percentiles
    avg_latency_ms  NUMERIC(10,2),
    p50_latency_ms  NUMERIC(10,2),
    p95_latency_ms  NUMERIC(10,2),
    p99_latency_ms  NUMERIC(10,2),
    -- Quality
    avg_confidence  NUMERIC(5,4),
    -- Cost
    total_cost_usd  NUMERIC(12,4) NOT NULL DEFAULT 0
);

SELECT create_hypertable('provider_telemetry', 'time',
    chunk_time_interval => INTERVAL '1 hour');

CREATE INDEX idx_pt_provider ON provider_telemetry (provider, time DESC);
CREATE INDEX idx_pt_pair ON provider_telemetry (language_pair, time DESC);


-- =====================================================
-- COST TRACKING
-- Granular cost tracking for translation spend analysis.
-- Supports the cost optimization recommendation engine.
-- =====================================================

CREATE TABLE translation_costs (
    time            TIMESTAMPTZ NOT NULL,
    tenant_id       UUID NOT NULL,
    source_language VARCHAR(35) NOT NULL,
    target_language VARCHAR(35) NOT NULL,
    provider        VARCHAR(50) NOT NULL,
    channel_type    VARCHAR(30),
    -- Cost details
    token_count     INTEGER NOT NULL,
    cost_usd        NUMERIC(10,6) NOT NULL,
    -- Context
    message_count   INTEGER NOT NULL DEFAULT 1,
    was_cached      BOOLEAN NOT NULL DEFAULT false,
    was_tm_reuse    BOOLEAN NOT NULL DEFAULT false
);

SELECT create_hypertable('translation_costs', 'time',
    chunk_time_interval => INTERVAL '1 day');

CREATE INDEX idx_tc_tenant ON translation_costs (tenant_id, time DESC);
CREATE INDEX idx_tc_langs ON translation_costs (source_language, target_language, time DESC);


-- =====================================================
-- CONTINUOUS AGGREGATES (TimescaleDB materialised views)
-- Pre-computed rollups that update automatically as new data arrives.
-- =====================================================

-- Hourly translation quality rollup
CREATE MATERIALIZED VIEW translation_quality_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    tenant_id,
    source_language,
    target_language,
    provider,
    channel_type,
    COUNT(*)                                    AS total_translations,
    AVG(confidence_score)                       AS avg_confidence,
    MIN(confidence_score)                       AS min_confidence,
    COUNT(*) FILTER (WHERE confidence_score < 0.8) AS low_confidence_count,
    COUNT(*) FILTER (WHERE status = 'failed')   AS failure_count,
    AVG(latency_ms)                             AS avg_latency_ms,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY latency_ms) AS p95_latency_ms,
    SUM(token_count)                            AS total_tokens,
    SUM(cost_usd)                               AS total_cost_usd,
    COUNT(*) FILTER (WHERE tm_match_used)       AS tm_reuse_count,
    COUNT(*) FILTER (WHERE glossary_terms_applied > 0) AS glossary_applied_count
FROM translation_telemetry
GROUP BY hour, tenant_id, source_language, target_language, provider, channel_type
WITH NO DATA;

-- Refresh policy: continuously update as new data arrives
SELECT add_continuous_aggregate_policy('translation_quality_hourly',
    start_offset    => INTERVAL '3 hours',
    end_offset      => INTERVAL '1 hour',
    schedule_interval => INTERVAL '30 minutes');

-- Daily conversation metrics rollup
CREATE MATERIALIZED VIEW conversation_metrics_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    tenant_id,
    language,
    channel_type,
    COUNT(*) FILTER (WHERE event_type = 'created')        AS conversations_created,
    COUNT(*) FILTER (WHERE event_type = 'resolved')       AS conversations_resolved,
    AVG(csat_score) FILTER (WHERE event_type = 'csat_submitted') AS avg_csat,
    AVG(first_response_seconds) FILTER (WHERE event_type = 'first_response') AS avg_first_response_sec,
    AVG(resolution_seconds) FILTER (WHERE event_type = 'resolved') AS avg_resolution_sec,
    COUNT(*) FILTER (WHERE event_type = 'resolved' AND is_fcr = true) AS fcr_count,
    COUNT(*) FILTER (WHERE event_type = 'escalated')      AS escalation_count,
    AVG(sentiment_score) FILTER (WHERE sentiment_score IS NOT NULL) AS avg_sentiment
FROM conversation_telemetry
GROUP BY day, tenant_id, language, channel_type
WITH NO DATA;

SELECT add_continuous_aggregate_policy('conversation_metrics_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 hour');

-- Daily cost rollup per tenant per language pair
CREATE MATERIALIZED VIEW translation_cost_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    tenant_id,
    source_language,
    target_language,
    provider,
    SUM(token_count)    AS total_tokens,
    SUM(cost_usd)       AS total_cost_usd,
    SUM(message_count)  AS total_messages,
    COUNT(*) FILTER (WHERE was_cached)   AS cached_count,
    COUNT(*) FILTER (WHERE was_tm_reuse) AS tm_reuse_count
FROM translation_costs
GROUP BY day, tenant_id, source_language, target_language, provider
WITH NO DATA;

SELECT add_continuous_aggregate_policy('translation_cost_daily',
    start_offset    => INTERVAL '3 days',
    end_offset      => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 hour');


-- =====================================================
-- COMPRESSION POLICIES
-- Compress old time-series data for storage efficiency
-- =====================================================

-- Compress translation telemetry older than 7 days
ALTER TABLE translation_telemetry SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, provider',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('translation_telemetry', INTERVAL '7 days');

-- Compress conversation telemetry older than 7 days
ALTER TABLE conversation_telemetry SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, language',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('conversation_telemetry', INTERVAL '7 days');

-- Compress provider telemetry older than 3 days
ALTER TABLE provider_telemetry SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'provider',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('provider_telemetry', INTERVAL '3 days');

-- Compress cost data older than 30 days
ALTER TABLE translation_costs SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'tenant_id, provider',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('translation_costs', INTERVAL '30 days');


-- =====================================================
-- RETENTION POLICIES
-- Automatically drop very old raw data (aggregates are kept)
-- =====================================================

SELECT add_retention_policy('translation_telemetry', INTERVAL '90 days');
SELECT add_retention_policy('conversation_telemetry', INTERVAL '180 days');
SELECT add_retention_policy('provider_telemetry', INTERVAL '30 days');
-- Cost data kept longer for budgeting/forecasting
SELECT add_retention_policy('translation_costs', INTERVAL '365 days');
```

---

## Key Query Patterns

### Cross-Lingual Knowledge Base Search

```sql
-- Customer searches for "パスワードをリセットする方法" (Japanese: "How to reset password")
-- The embedding model maps this to the same vector space as the English article
-- about password reset, enabling cross-lingual search.

WITH query_embedding AS (
    -- In practice, the application generates this embedding via the ML model
    SELECT $1::vector(1024) AS embedding
)
SELECT
    ka.id AS article_id,
    ka.title,
    ka.category,
    kat.title AS translated_title,
    kat.language_id,
    1 - (kae.embedding <=> qe.embedding) AS similarity_score
FROM kb_article_embeddings kae
JOIN kb_articles ka ON ka.id = kae.article_id
LEFT JOIN kb_article_translations kat ON kat.article_id = ka.id
    AND kat.language_id = $2  -- customer's language
CROSS JOIN query_embedding qe
WHERE kae.tenant_id = $3
  AND ka.status = 'published'
ORDER BY kae.embedding <=> qe.embedding
LIMIT 10;
```

### Translation Memory Lookup

```sql
-- Before calling an external translation provider, check if a similar
-- segment was previously translated with high quality.

WITH query_embedding AS (
    SELECT $1::vector(1024) AS embedding  -- embedding of source text
)
SELECT
    tm.source_text,
    tm.target_text,
    tm.confidence_score,
    tm.quality_level,
    tm.reuse_count,
    1 - (tm.source_embedding <=> qe.embedding) AS similarity_score
FROM translation_memory tm
CROSS JOIN query_embedding qe
WHERE tm.tenant_id = $2
  AND tm.source_language_id = $3
  AND tm.target_language_id = $4
  AND tm.confidence_score >= 0.85
  -- Only consider matches above 0.92 similarity threshold
  AND (1 - (tm.source_embedding <=> qe.embedding)) >= 0.92
ORDER BY tm.source_embedding <=> qe.embedding
LIMIT 5;
```

### Similar Ticket Detection

```sql
-- When a new customer message arrives, find similar previous messages
-- for deduplication, routing, or suggesting resolutions.

WITH query_embedding AS (
    SELECT $1::vector(1024) AS embedding
)
SELECT
    me.conversation_id,
    c.subject,
    c.status,
    c.resolved_at,
    c.customer_language_id,
    m.original_content,
    1 - (me.embedding <=> qe.embedding) AS similarity_score
FROM message_embeddings me
JOIN messages m ON m.id = me.message_id
JOIN conversations c ON c.id = me.conversation_id
CROSS JOIN query_embedding qe
WHERE me.tenant_id = $2
  AND c.status IN ('resolved', 'closed')
  AND (1 - (me.embedding <=> qe.embedding)) >= 0.85
ORDER BY me.embedding <=> qe.embedding
LIMIT 10;
```

### Translation Quality Trend Analysis

```sql
-- Show hourly translation quality trends for Japanese->English over the last 7 days
-- This query hits the continuous aggregate, not the raw telemetry
SELECT
    hour,
    total_translations,
    avg_confidence,
    low_confidence_count,
    avg_latency_ms,
    p95_latency_ms,
    total_cost_usd,
    tm_reuse_count
FROM translation_quality_hourly
WHERE tenant_id = $1
  AND source_language = 'ja-JP'
  AND target_language = 'en-US'
  AND hour >= NOW() - INTERVAL '7 days'
ORDER BY hour ASC;
```

### Cost Optimization Analysis

```sql
-- Monthly cost per language pair with TM reuse savings
SELECT
    source_language,
    target_language,
    provider,
    SUM(total_cost_usd) AS total_cost,
    SUM(total_tokens) AS total_tokens,
    SUM(total_messages) AS total_messages,
    SUM(tm_reuse_count) AS tm_reuses,
    SUM(cached_count) AS cached_translations,
    -- Estimated savings from TM reuse (assumes average cost per translation)
    CASE WHEN SUM(total_messages) > 0
         THEN (SUM(tm_reuse_count)::NUMERIC / SUM(total_messages)) * 100
         ELSE 0
    END AS tm_reuse_pct
FROM translation_cost_daily
WHERE tenant_id = $1
  AND day >= date_trunc('month', NOW())
GROUP BY source_language, target_language, provider
ORDER BY total_cost DESC;
```

### Quality Degradation Detection

```sql
-- Detect languages where translation quality is declining
-- Compare last 7 days to the preceding 7 days
WITH recent AS (
    SELECT source_language, target_language,
           AVG(avg_confidence) AS recent_avg_confidence,
           SUM(low_confidence_count)::NUMERIC / NULLIF(SUM(total_translations), 0) AS recent_low_pct
    FROM translation_quality_hourly
    WHERE tenant_id = $1
      AND hour >= NOW() - INTERVAL '7 days'
    GROUP BY source_language, target_language
),
previous AS (
    SELECT source_language, target_language,
           AVG(avg_confidence) AS prev_avg_confidence,
           SUM(low_confidence_count)::NUMERIC / NULLIF(SUM(total_translations), 0) AS prev_low_pct
    FROM translation_quality_hourly
    WHERE tenant_id = $1
      AND hour >= NOW() - INTERVAL '14 days'
      AND hour < NOW() - INTERVAL '7 days'
    GROUP BY source_language, target_language
)
SELECT
    r.source_language,
    r.target_language,
    r.recent_avg_confidence,
    p.prev_avg_confidence,
    r.recent_avg_confidence - p.prev_avg_confidence AS confidence_delta,
    r.recent_low_pct,
    p.prev_low_pct,
    r.recent_low_pct - p.prev_low_pct AS low_pct_delta
FROM recent r
JOIN previous p ON r.source_language = p.source_language
                AND r.target_language = p.target_language
WHERE r.recent_avg_confidence < p.prev_avg_confidence - 0.02  -- 2% decline threshold
   OR r.recent_low_pct > p.prev_low_pct + 0.05               -- 5% increase in low-quality
ORDER BY confidence_delta ASC;
```

---

## Pros and Cons

### Pros

1. **Cross-lingual search solves a real product problem.** The README identifies "language-aware search so customers find answers in their own language" as a key feature. Vector embeddings from a multilingual model are the state-of-the-art solution for this. A Japanese customer's query finds English articles because both are embedded in the same vector space. No keyword translation pipeline is needed.

2. **Translation memory reduces cost and improves consistency.** By embedding previously-translated segments and searching for similar ones before calling an external provider, the system builds an organic translation memory. High-confidence, agent-reviewed translations are reused, reducing API costs and ensuring consistency. This directly implements the "self-improving glossary that learns from agent corrections" feature from the README.

3. **Time-series analytics are a natural fit.** Translation quality metrics, provider performance, and cost tracking are inherently time-series data. TimescaleDB's continuous aggregates provide pre-computed hourly and daily rollups without ETL pipelines. The "proactive translation quality trend monitoring with degradation alerts" feature from the README is implemented directly as a SQL query over the continuous aggregate.

4. **Single database cluster.** pgvector and TimescaleDB are both PostgreSQL extensions. They run in the same PostgreSQL instance, sharing the same connection pool, backup infrastructure, and operational tooling. There is no separate vector database (Pinecone, Qdrant) or time-series database (InfluxDB) to manage.

5. **Compression and retention are built in.** TimescaleDB's compression (10-20x for time-series data) and retention policies automatically manage data lifecycle. Raw telemetry is compressed after 7 days and dropped after 90 days, but continuous aggregates retain the rollup data indefinitely.

6. **Fuzzy glossary matching.** Vector similarity over glossary term embeddings enables fuzzy matching that goes beyond substring and regex patterns. A customer writing "cloud synchronization professional version" can be matched to the glossary term "CloudSync Pro" through semantic similarity.

7. **Similar ticket detection and intelligent routing.** Message embeddings enable finding conversations about similar issues regardless of the language they were originally written in. An agent who successfully resolved a Japanese customer's billing issue can be routed similar billing issues from Korean customers.

### Cons

1. **Embedding generation adds latency to the pipeline.** Every incoming message needs to be embedded before it can be used for TM lookup, similar ticket detection, or KB search. Embedding generation takes 10-50ms per request (batch inference) or 50-200ms (single request). For sub-second live chat translation, this latency may be unacceptable on the critical path. Mitigation: generate embeddings asynchronously and use keyword-based fallback for the first message in a conversation.

2. **Embedding model dependency.** The system depends on a specific embedding model (e.g., `multilingual-e5-large`). Changing models requires re-embedding all existing data (KB articles, TM entries, messages). For a tenant with millions of TM entries, this is a multi-hour batch job. The `embedding_model` column enables gradual migration but adds query complexity.

3. **Vector index memory requirements.** HNSW indexes are memory-intensive. A 1024-dimension vector consumes 4 KB. With 10 million TM entries, the vector data alone is ~40 GB, and the HNSW index adds 2-3x overhead. At scale, this requires significant RAM allocation to PostgreSQL.

4. **pgvector is less mature than dedicated vector databases.** While pgvector handles millions of vectors well, dedicated vector databases (Qdrant, Pinecone, Weaviate) offer more advanced features: multi-vector search, filtered HNSW, streaming updates, and horizontal scaling. If vector search becomes a primary workload (not just supplementary), a dedicated vector database may be needed.

5. **TimescaleDB license considerations.** TimescaleDB Community Edition is open source (Apache 2.0 for core features), but some features (continuous aggregates, compression, retention policies) require the Timescale License which allows free use but restricts offering TimescaleDB as a managed service. For a self-hosted or cloud-hosted deployment this is not an issue, but it constrains business model options.

6. **Operational complexity of extensions.** Running two PostgreSQL extensions (pgvector + TimescaleDB) in the same instance requires careful version management, especially during PostgreSQL major version upgrades. Both extensions must be compatible with the PostgreSQL version and with each other.

7. **Cost of embedding model inference.** Generating embeddings for every message, KB article chunk, and glossary term requires either self-hosted model inference (GPU infrastructure) or API calls to an embedding provider. This adds a per-translation cost beyond the translation provider's charge. Mitigation: batch embedding generation, cache embeddings aggressively, and use lighter models (e5-small vs. e5-large) for less critical applications (message embeddings) while reserving the large model for KB and TM.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with pgvector 0.7+ and TimescaleDB 2.15+ |
| **Embedding Model** | `intfloat/multilingual-e5-large` (1024-dim, 100+ languages) for KB and TM; `intfloat/multilingual-e5-small` (384-dim) for message embeddings if latency is critical |
| **Embedding Inference** | Self-hosted with ONNX Runtime or vLLM on GPU instances; fallback to API (Cohere Embed, OpenAI Embeddings) for burst capacity |
| **Vector Index** | HNSW (pgvector) with m=16, ef_construction=256 for production; tune ef_search per query (lower for speed, higher for recall) |
| **Connection Pooling** | PgBouncer in transaction mode |
| **Caching** | Redis for hot glossary term lookups; embedding cache for recently-embedded segments |
| **Monitoring** | pg_stat_statements for query performance; custom metrics for embedding generation latency, TM hit rate, and vector index recall |
| **Backup** | Standard PostgreSQL WAL archiving; vector and time-series data are included in regular backups |

---

## Migration and Scaling Considerations

### Phase 1: MVP (All-in-One PostgreSQL)

- Single PostgreSQL instance with pgvector and TimescaleDB extensions
- Use `multilingual-e5-small` (384-dim vectors) to reduce memory footprint
- Embed only KB articles and high-quality TM entries (agent-reviewed translations)
- Message embedding is deferred to post-MVP
- TimescaleDB hypertables for translation telemetry with 7-day raw retention

### Phase 2: Growth (Optimize Vector Search)

- Upgrade to `multilingual-e5-large` (1024-dim) for KB and TM as quality demands increase
- Add message embeddings for similar ticket detection and intelligent routing
- Deploy dedicated GPU instances for embedding inference (batch processing)
- Increase HNSW index parameters for better recall as the TM grows
- Add continuous aggregates for cost tracking and provider health monitoring

### Phase 3: Scale (Consider Specialization)

- If vector search becomes a bottleneck (>50M vectors), evaluate migrating to a dedicated vector database (Qdrant or Milvus) while keeping PostgreSQL as the relational core
- If time-series volume exceeds TimescaleDB capacity, evaluate ClickHouse for analytics while keeping TimescaleDB for real-time telemetry
- Partition translation_memory by tenant_id for large tenants
- Archive old embeddings (messages from resolved conversations older than 6 months) to cold storage
- Consider HNSW index partitioning by tenant for multi-tenant isolation of search results

### Embedding Model Migration Strategy

When upgrading embedding models (inevitable as better multilingual models are released):

1. Add new embeddings with the new model alongside old embeddings (both rows exist)
2. Update queries to prefer the new model when available, fall back to old
3. Run a background job to re-embed all existing content with the new model
4. Once migration is complete, drop old embedding rows and update unique constraints
5. The `embedding_model` column on all embedding tables enables this gradual migration

### Data Residency

- Same regional PostgreSQL cluster approach as other models
- Embeddings contain encoded content and are subject to the same data residency rules as the source content
- Embedding model inference should run in the same region as the data to avoid cross-region PII transfer
