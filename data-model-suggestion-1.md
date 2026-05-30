# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Multi-Language Customer Support (Candidate #475)
> Generated: 2026-05-26

## Overview

This model uses a fully normalized relational schema in PostgreSQL, with strict referential integrity, foreign key constraints, and a separation-of-concerns approach where each domain concept occupies its own table. The design is optimized for transactional consistency, complex analytical queries, and regulatory compliance (GDPR, HIPAA) where audit trails and data lineage are critical.

The schema is organized into seven logical domains:

1. **Tenancy and Organization** -- multi-tenant isolation, organizations, teams, agents
2. **Language and Locale** -- ISO 639 languages, BCP 47 locale tags, dialect variants
3. **Channels and Integrations** -- helpdesk connectors, channel types, webhook configurations
4. **Conversations and Messages** -- tickets, conversations, message threads, attachments
5. **Translation Pipeline** -- translation requests, results, quality scores, provider routing
6. **Glossary and Terminology** -- brand glossaries, term entries, corrections, learning feedback
7. **Knowledge Base** -- articles, article translations, translation status tracking
8. **Analytics and Quality Monitoring** -- per-language CSAT, translation quality trends, alerts

---

## Complete Schema Definition

### 1. Tenancy and Organization

```sql
-- Tenants represent distinct customer organizations using the platform
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'starter'
                    CHECK (plan_tier IN ('starter', 'professional', 'enterprise')),
    max_agents      INTEGER NOT NULL DEFAULT 10,
    max_languages   INTEGER NOT NULL DEFAULT 10,
    data_residency  VARCHAR(10) NOT NULL DEFAULT 'us'
                    CHECK (data_residency IN ('us', 'eu', 'ap', 'custom')),
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    suspended_at    TIMESTAMPTZ
);

CREATE INDEX idx_tenants_slug ON tenants (slug);
CREATE INDEX idx_tenants_plan ON tenants (plan_tier);

-- Agents are support staff who interact with translated conversations
CREATE TABLE agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),  -- ID in the connected helpdesk
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    primary_language_id UUID,      -- FK set after languages table is created
    role            VARCHAR(50) NOT NULL DEFAULT 'agent'
                    CHECK (role IN ('agent', 'supervisor', 'admin', 'owner')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_agents_tenant ON agents (tenant_id);
CREATE INDEX idx_agents_external ON agents (tenant_id, external_id);

-- Teams group agents for routing and assignment
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    agent_id        UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (team_id, agent_id)
);

-- Languages an agent is proficient in (for routing decisions)
CREATE TABLE agent_languages (
    agent_id        UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    language_id     UUID NOT NULL,  -- FK set after languages table
    proficiency     VARCHAR(20) NOT NULL DEFAULT 'fluent'
                    CHECK (proficiency IN ('basic', 'intermediate', 'fluent', 'native')),
    PRIMARY KEY (agent_id, language_id)
);
```

### 2. Language and Locale

```sql
-- Master language registry following ISO 639-1/639-3 and BCP 47
CREATE TABLE languages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    iso_639_1       CHAR(2),           -- e.g., 'en', 'es', 'zh'
    iso_639_3       CHAR(3),           -- e.g., 'eng', 'spa', 'zho'
    bcp47_tag       VARCHAR(35) NOT NULL UNIQUE,  -- e.g., 'en-US', 'es-419', 'zh-Hant-TW'
    name_english    VARCHAR(255) NOT NULL,
    name_native     VARCHAR(255),
    script          VARCHAR(10),       -- e.g., 'Latn', 'Hans', 'Hant', 'Arab'
    direction       VARCHAR(3) NOT NULL DEFAULT 'ltr'
                    CHECK (direction IN ('ltr', 'rtl')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_languages_iso1 ON languages (iso_639_1);
CREATE INDEX idx_languages_bcp47 ON languages (bcp47_tag);

-- Now add FK for agents.primary_language_id
ALTER TABLE agents
    ADD CONSTRAINT fk_agents_primary_language
    FOREIGN KEY (primary_language_id) REFERENCES languages(id);

ALTER TABLE agent_languages
    ADD CONSTRAINT fk_agent_languages_language
    FOREIGN KEY (language_id) REFERENCES languages(id);

-- Dialect/variant groupings (e.g., Castilian vs Latin American Spanish)
CREATE TABLE language_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_language_id UUID NOT NULL REFERENCES languages(id),
    variant_language_id UUID NOT NULL REFERENCES languages(id),
    relationship    VARCHAR(50) NOT NULL DEFAULT 'dialect'
                    CHECK (relationship IN ('dialect', 'script_variant', 'regional', 'formal_informal')),
    fallback_priority INTEGER NOT NULL DEFAULT 0,
    UNIQUE (parent_language_id, variant_language_id)
);

-- Languages enabled per tenant
CREATE TABLE tenant_languages (
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    language_id     UUID NOT NULL REFERENCES languages(id),
    is_source       BOOLEAN NOT NULL DEFAULT false,   -- can agents write in this language?
    is_target       BOOLEAN NOT NULL DEFAULT true,    -- can customers use this language?
    preferred_provider VARCHAR(50),                    -- translation provider preference
    enabled_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, language_id)
);

CREATE INDEX idx_tenant_languages_tenant ON tenant_languages (tenant_id);
```

### 3. Channels and Integrations

```sql
-- Helpdesk platform connections (Zendesk, Salesforce, Freshdesk, etc.)
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    platform        VARCHAR(50) NOT NULL
                    CHECK (platform IN ('zendesk', 'salesforce', 'freshdesk',
                                        'intercom', 'servicenow', 'custom')),
    name            VARCHAR(255) NOT NULL,
    base_url        VARCHAR(2048),
    auth_type       VARCHAR(20) NOT NULL DEFAULT 'oauth2'
                    CHECK (auth_type IN ('oauth2', 'api_key', 'basic', 'jwt')),
    credentials_vault_ref VARCHAR(512),   -- reference to secret manager, never store creds directly
    webhook_secret  VARCHAR(512),
    sync_direction  VARCHAR(20) NOT NULL DEFAULT 'bidirectional'
                    CHECK (sync_direction IN ('inbound', 'outbound', 'bidirectional')),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'paused', 'error', 'disconnected')),
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_integrations_tenant ON integrations (tenant_id);
CREATE INDEX idx_integrations_platform ON integrations (tenant_id, platform);

-- Communication channels through which messages arrive/depart
CREATE TABLE channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    integration_id  UUID REFERENCES integrations(id) ON DELETE SET NULL,
    channel_type    VARCHAR(30) NOT NULL
                    CHECK (channel_type IN ('email', 'chat', 'voice', 'sms',
                                            'social_facebook', 'social_twitter',
                                            'social_whatsapp', 'social_instagram',
                                            'web_form', 'api', 'other')),
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_channels_tenant ON channels (tenant_id);
CREATE INDEX idx_channels_type ON channels (tenant_id, channel_type);

-- Webhook configurations for receiving events from helpdesks
CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES integrations(id) ON DELETE CASCADE,
    event_type      VARCHAR(100) NOT NULL,  -- e.g., 'ticket.created', 'message.received'
    endpoint_url    VARCHAR(2048) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    retry_policy    JSONB NOT NULL DEFAULT '{"max_retries": 3, "backoff_ms": 1000}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 4. Conversations and Messages

```sql
-- Customers who interact through support channels
CREATE TABLE customers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),             -- ID in the connected helpdesk/CRM
    email           VARCHAR(320),
    phone           VARCHAR(50),
    display_name    VARCHAR(255),
    detected_language_id UUID REFERENCES languages(id),
    preferred_language_id UUID REFERENCES languages(id),
    metadata        JSONB NOT NULL DEFAULT '{}',
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_tenant ON customers (tenant_id);
CREATE INDEX idx_customers_external ON customers (tenant_id, external_id);
CREATE INDEX idx_customers_email ON customers (tenant_id, email);
CREATE INDEX idx_customers_language ON customers (detected_language_id);

-- Conversations (tickets/cases) are the top-level container
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    channel_id      UUID REFERENCES channels(id),
    customer_id     UUID REFERENCES customers(id),
    assigned_agent_id UUID REFERENCES agents(id),
    assigned_team_id UUID REFERENCES teams(id),
    external_ticket_id VARCHAR(255),         -- ticket ID in connected helpdesk
    integration_id  UUID REFERENCES integrations(id),
    subject         TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'pending', 'waiting_on_customer',
                                      'waiting_on_third_party', 'resolved', 'closed')),
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal'
                    CHECK (priority IN ('low', 'normal', 'high', 'urgent')),
    customer_language_id UUID REFERENCES languages(id),
    agent_language_id UUID REFERENCES languages(id),
    requires_translation BOOLEAN NOT NULL DEFAULT true,
    sentiment_score NUMERIC(4,3),            -- -1.000 to 1.000
    csat_score      SMALLINT CHECK (csat_score BETWEEN 1 AND 5),
    csat_submitted_at TIMESTAMPTZ,
    first_response_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_conversations_tenant ON conversations (tenant_id);
CREATE INDEX idx_conversations_customer ON conversations (customer_id);
CREATE INDEX idx_conversations_agent ON conversations (assigned_agent_id);
CREATE INDEX idx_conversations_status ON conversations (tenant_id, status);
CREATE INDEX idx_conversations_external ON conversations (tenant_id, external_ticket_id);
CREATE INDEX idx_conversations_created ON conversations (tenant_id, created_at DESC);
CREATE INDEX idx_conversations_lang ON conversations (customer_language_id);

-- Individual messages within a conversation
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    sender_type     VARCHAR(20) NOT NULL
                    CHECK (sender_type IN ('customer', 'agent', 'bot', 'system')),
    sender_agent_id UUID REFERENCES agents(id),
    sender_customer_id UUID REFERENCES customers(id),
    channel_id      UUID REFERENCES channels(id),
    original_content TEXT NOT NULL,
    original_language_id UUID NOT NULL REFERENCES languages(id),
    original_content_type VARCHAR(20) NOT NULL DEFAULT 'text'
                    CHECK (original_content_type IN ('text', 'html', 'markdown')),
    is_translated   BOOLEAN NOT NULL DEFAULT false,
    is_internal_note BOOLEAN NOT NULL DEFAULT false,
    sequence_number INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation ON messages (conversation_id, sequence_number);
CREATE INDEX idx_messages_tenant ON messages (tenant_id);
CREATE INDEX idx_messages_created ON messages (conversation_id, created_at);

-- Translated versions of messages
CREATE TABLE message_translations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    target_language_id UUID NOT NULL REFERENCES languages(id),
    translated_content TEXT NOT NULL,
    translation_provider VARCHAR(50) NOT NULL,      -- 'google', 'deepl', 'azure', 'amazon', 'internal'
    model_version   VARCHAR(100),
    confidence_score NUMERIC(5,4),                  -- 0.0000 to 1.0000
    quality_flag    VARCHAR(20) NOT NULL DEFAULT 'ok'
                    CHECK (quality_flag IN ('ok', 'low_confidence', 'needs_review',
                                            'reviewed', 'corrected')),
    latency_ms      INTEGER,                        -- translation round-trip time
    token_count     INTEGER,                        -- for cost tracking
    cost_usd        NUMERIC(10,6),                  -- per-translation cost
    reviewed_by_agent_id UUID REFERENCES agents(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (message_id, target_language_id)
);

CREATE INDEX idx_msg_translations_message ON message_translations (message_id);
CREATE INDEX idx_msg_translations_quality ON message_translations (quality_flag)
    WHERE quality_flag != 'ok';
CREATE INDEX idx_msg_translations_provider ON message_translations (translation_provider);

-- Attachments on messages
CREATE TABLE message_attachments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    file_name       VARCHAR(512) NOT NULL,
    file_type       VARCHAR(100) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    storage_url     VARCHAR(2048) NOT NULL,
    is_translatable BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 5. Glossary and Terminology Management

```sql
-- Brand glossaries are collections of terms for a tenant
CREATE TABLE glossaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    industry        VARCHAR(100),  -- 'healthcare', 'finance', 'ecommerce', 'technology', etc.
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_glossaries_tenant ON glossaries (tenant_id);

-- Individual glossary terms with source-target language pairs
CREATE TABLE glossary_terms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    glossary_id     UUID NOT NULL REFERENCES glossaries(id) ON DELETE CASCADE,
    source_language_id UUID NOT NULL REFERENCES languages(id),
    target_language_id UUID NOT NULL REFERENCES languages(id),
    source_term     VARCHAR(1000) NOT NULL,
    target_term     VARCHAR(1000) NOT NULL,
    case_sensitive  BOOLEAN NOT NULL DEFAULT false,
    match_type      VARCHAR(20) NOT NULL DEFAULT 'exact'
                    CHECK (match_type IN ('exact', 'prefix', 'fuzzy', 'regex')),
    context_hint    TEXT,                          -- usage context for disambiguation
    is_do_not_translate BOOLEAN NOT NULL DEFAULT false,  -- e.g., brand names
    confidence      NUMERIC(5,4) NOT NULL DEFAULT 1.0,
    source          VARCHAR(20) NOT NULL DEFAULT 'manual'
                    CHECK (source IN ('manual', 'auto_learned', 'imported', 'industry_pack')),
    approved_by     UUID REFERENCES agents(id),
    approved_at     TIMESTAMPTZ,
    usage_count     INTEGER NOT NULL DEFAULT 0,
    last_used_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_glossary_terms_glossary ON glossary_terms (glossary_id);
CREATE INDEX idx_glossary_terms_source ON glossary_terms (source_language_id, source_term);
CREATE INDEX idx_glossary_terms_lookup ON glossary_terms (glossary_id, source_language_id,
    target_language_id, source_term);

-- Agent corrections that feed the self-improving glossary
CREATE TABLE glossary_corrections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    message_translation_id UUID REFERENCES message_translations(id),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    source_language_id UUID NOT NULL REFERENCES languages(id),
    target_language_id UUID NOT NULL REFERENCES languages(id),
    original_segment TEXT NOT NULL,
    machine_translation TEXT NOT NULL,
    agent_correction TEXT NOT NULL,
    correction_type VARCHAR(30) NOT NULL DEFAULT 'terminology'
                    CHECK (correction_type IN ('terminology', 'grammar', 'tone',
                                               'context', 'cultural', 'factual')),
    promoted_to_term_id UUID REFERENCES glossary_terms(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_glossary_corrections_tenant ON glossary_corrections (tenant_id);
CREATE INDEX idx_glossary_corrections_agent ON glossary_corrections (agent_id);
CREATE INDEX idx_glossary_corrections_langs ON glossary_corrections (source_language_id,
    target_language_id);

-- Pre-built industry terminology packs
CREATE TABLE industry_term_packs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    industry        VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    term_count      INTEGER NOT NULL DEFAULT 0,
    version         VARCHAR(20) NOT NULL DEFAULT '1.0',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE industry_pack_terms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pack_id         UUID NOT NULL REFERENCES industry_term_packs(id) ON DELETE CASCADE,
    source_language_id UUID NOT NULL REFERENCES languages(id),
    target_language_id UUID NOT NULL REFERENCES languages(id),
    source_term     VARCHAR(1000) NOT NULL,
    target_term     VARCHAR(1000) NOT NULL,
    context_hint    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 6. Knowledge Base

```sql
-- Knowledge base articles (master/source language)
CREATE TABLE kb_articles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),         -- article ID in connected helpdesk
    source_language_id UUID NOT NULL REFERENCES languages(id),
    title           VARCHAR(1000) NOT NULL,
    slug            VARCHAR(500),
    body            TEXT NOT NULL,
    body_format     VARCHAR(20) NOT NULL DEFAULT 'html'
                    CHECK (body_format IN ('html', 'markdown', 'plain')),
    category        VARCHAR(255),
    tags            TEXT[],
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'archived')),
    version         INTEGER NOT NULL DEFAULT 1,
    last_modified_hash VARCHAR(64),       -- SHA-256 of body for change detection
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_kb_articles_tenant ON kb_articles (tenant_id);
CREATE INDEX idx_kb_articles_status ON kb_articles (tenant_id, status);
CREATE INDEX idx_kb_articles_category ON kb_articles (tenant_id, category);
CREATE INDEX idx_kb_articles_tags ON kb_articles USING GIN (tags);

-- Translated versions of knowledge base articles
CREATE TABLE kb_article_translations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID NOT NULL REFERENCES kb_articles(id) ON DELETE CASCADE,
    language_id     UUID NOT NULL REFERENCES languages(id),
    title           VARCHAR(1000) NOT NULL,
    body            TEXT NOT NULL,
    translation_method VARCHAR(30) NOT NULL DEFAULT 'machine'
                    CHECK (translation_method IN ('machine', 'human', 'hybrid', 'imported')),
    translation_provider VARCHAR(50),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'machine_translated', 'human_reviewed',
                                      'published', 'outdated', 'archived')),
    source_version  INTEGER NOT NULL,      -- article version this was translated from
    is_outdated     BOOLEAN NOT NULL DEFAULT false,
    quality_score   NUMERIC(5,4),
    reviewed_by     UUID REFERENCES agents(id),
    reviewed_at     TIMESTAMPTZ,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (article_id, language_id)
);

CREATE INDEX idx_kb_translations_article ON kb_article_translations (article_id);
CREATE INDEX idx_kb_translations_status ON kb_article_translations (status);
CREATE INDEX idx_kb_translations_outdated ON kb_article_translations (is_outdated)
    WHERE is_outdated = true;

-- Track which articles generate follow-up questions per language (knowledge gap detection)
CREATE TABLE kb_article_feedback (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID NOT NULL REFERENCES kb_articles(id) ON DELETE CASCADE,
    language_id     UUID NOT NULL REFERENCES languages(id),
    conversation_id UUID REFERENCES conversations(id),
    feedback_type   VARCHAR(30) NOT NULL
                    CHECK (feedback_type IN ('helpful', 'not_helpful', 'followup_question',
                                             'wrong_language', 'outdated', 'unclear')),
    feedback_text   TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_kb_feedback_article_lang ON kb_article_feedback (article_id, language_id);
```

### 7. Analytics and Quality Monitoring

```sql
-- Daily aggregated translation quality metrics per language pair
CREATE TABLE translation_quality_daily (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    date            DATE NOT NULL,
    source_language_id UUID NOT NULL REFERENCES languages(id),
    target_language_id UUID NOT NULL REFERENCES languages(id),
    translation_provider VARCHAR(50) NOT NULL,
    channel_type    VARCHAR(30),
    total_translations INTEGER NOT NULL DEFAULT 0,
    avg_confidence  NUMERIC(5,4),
    min_confidence  NUMERIC(5,4),
    low_confidence_count INTEGER NOT NULL DEFAULT 0,
    corrections_count INTEGER NOT NULL DEFAULT 0,
    avg_latency_ms  NUMERIC(10,2),
    total_tokens    BIGINT NOT NULL DEFAULT 0,
    total_cost_usd  NUMERIC(12,4) NOT NULL DEFAULT 0,
    UNIQUE (tenant_id, date, source_language_id, target_language_id,
            translation_provider, channel_type)
);

CREATE INDEX idx_tq_daily_tenant_date ON translation_quality_daily (tenant_id, date DESC);
CREATE INDEX idx_tq_daily_langs ON translation_quality_daily (source_language_id,
    target_language_id);

-- Daily aggregated conversation metrics per language
CREATE TABLE conversation_metrics_daily (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    date            DATE NOT NULL,
    language_id     UUID NOT NULL REFERENCES languages(id),
    channel_type    VARCHAR(30),
    total_conversations INTEGER NOT NULL DEFAULT 0,
    resolved_conversations INTEGER NOT NULL DEFAULT 0,
    avg_csat        NUMERIC(4,2),
    avg_first_response_seconds NUMERIC(12,2),
    avg_resolution_seconds NUMERIC(12,2),
    first_contact_resolution_count INTEGER NOT NULL DEFAULT 0,
    escalation_count INTEGER NOT NULL DEFAULT 0,
    avg_sentiment   NUMERIC(4,3),
    UNIQUE (tenant_id, date, language_id, channel_type)
);

CREATE INDEX idx_cm_daily_tenant_date ON conversation_metrics_daily (tenant_id, date DESC);
CREATE INDEX idx_cm_daily_language ON conversation_metrics_daily (language_id);

-- Quality degradation alerts
CREATE TABLE quality_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    alert_type      VARCHAR(50) NOT NULL
                    CHECK (alert_type IN ('confidence_drop', 'correction_spike',
                                          'latency_increase', 'cost_anomaly',
                                          'csat_decline', 'volume_spike')),
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('info', 'warning', 'critical')),
    language_id     UUID REFERENCES languages(id),
    source_language_id UUID REFERENCES languages(id),
    target_language_id UUID REFERENCES languages(id),
    description     TEXT NOT NULL,
    metric_name     VARCHAR(100),
    metric_value    NUMERIC(12,4),
    threshold_value NUMERIC(12,4),
    is_acknowledged BOOLEAN NOT NULL DEFAULT false,
    acknowledged_by UUID REFERENCES agents(id),
    acknowledged_at TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_quality_alerts_tenant ON quality_alerts (tenant_id, created_at DESC);
CREATE INDEX idx_quality_alerts_unack ON quality_alerts (tenant_id, is_acknowledged)
    WHERE is_acknowledged = false;

-- Translation provider performance tracking
CREATE TABLE provider_performance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    provider        VARCHAR(50) NOT NULL,
    language_pair   VARCHAR(10) NOT NULL,     -- e.g., 'en-es', 'en-ja'
    date            DATE NOT NULL,
    request_count   INTEGER NOT NULL DEFAULT 0,
    success_count   INTEGER NOT NULL DEFAULT 0,
    error_count     INTEGER NOT NULL DEFAULT 0,
    avg_latency_ms  NUMERIC(10,2),
    p95_latency_ms  NUMERIC(10,2),
    p99_latency_ms  NUMERIC(10,2),
    avg_confidence  NUMERIC(5,4),
    total_cost_usd  NUMERIC(12,4),
    UNIQUE (tenant_id, provider, language_pair, date)
);

CREATE INDEX idx_provider_perf_tenant ON provider_performance (tenant_id, date DESC);
```

### 8. Audit Trail

```sql
-- Comprehensive audit log for GDPR/compliance
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    actor_type      VARCHAR(20) NOT NULL
                    CHECK (actor_type IN ('agent', 'customer', 'system', 'integration', 'admin')),
    actor_id        UUID,
    action          VARCHAR(100) NOT NULL,     -- 'message.translated', 'glossary.updated', etc.
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    details         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Partition audit_log by month for performance
CREATE INDEX idx_audit_log_tenant ON audit_log (tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_resource ON audit_log (resource_type, resource_id);
CREATE INDEX idx_audit_log_action ON audit_log (action);

-- GDPR data subject access / erasure tracking
CREATE TABLE data_subject_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    request_type    VARCHAR(20) NOT NULL
                    CHECK (request_type IN ('access', 'erasure', 'portability', 'rectification')),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'processing', 'completed', 'denied')),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    notes           TEXT
);
```

---

## Pros and Cons

### Pros

1. **Referential integrity guarantees.** Foreign keys prevent orphaned translations, dangling glossary references, and broken conversation-message relationships. In a translation platform where data lineage is critical (which message was translated, by which provider, with what confidence), relational constraints ensure every record is traceable.

2. **Complex analytical queries.** Per-language CSAT breakdowns, translation quality trend analysis, cost-per-language-pair reporting, and provider performance comparisons are all natural SQL aggregations over well-indexed tables. No denormalization or materialized views are required for most reporting.

3. **GDPR and compliance.** The normalized structure makes data subject access requests straightforward: follow the foreign keys from a customer to all their conversations, messages, translations, and feedback. Erasure is similarly clean with CASCADE deletes. Audit trails live in a dedicated table with clear provenance.

4. **Multi-tenant isolation.** The tenant_id column on every table, combined with row-level security policies, provides robust logical isolation without the operational complexity of schema-per-tenant or database-per-tenant approaches.

5. **Glossary and terminology integrity.** The glossary_terms table enforces unique source-target pairs per glossary, prevents duplicate entries, and tracks provenance (manual vs. auto-learned). The correction-to-term promotion pipeline is traceable through foreign keys.

6. **Mature ecosystem.** PostgreSQL has extensive support for full-text search, JSONB for semi-structured data where needed (tenant settings, webhook configs), array types for tags, and extensions like pg_trgm for fuzzy glossary matching.

### Cons

1. **Schema rigidity.** Adding new per-tenant custom fields, channel-specific metadata, or translation provider-specific response attributes requires ALTER TABLE migrations. In a middleware that integrates with many different helpdesks, each with unique data shapes, the normalized model forces either frequent migrations or a catch-all JSONB column that undermines normalization.

2. **Write contention on hot tables.** The messages and message_translations tables will see very high write volumes during peak support hours. PostgreSQL's MVCC model handles this well, but extremely high throughput (thousands of translations per second) may require connection pooling, partitioning, or read replicas.

3. **Join-heavy queries.** Retrieving a complete conversation with all messages, their translations, glossary terms applied, and quality scores requires joining 4-6 tables. While PostgreSQL handles this efficiently with proper indexing, response times for deep conversation retrieval can grow as conversations get longer.

4. **Analytics at scale.** The daily aggregation tables help, but real-time analytics over millions of messages (e.g., "show me translation quality trends for the last 90 days across all language pairs") may need materialized views or a separate analytics data warehouse.

5. **Translation memory limitations.** A normalized relational model is not the most natural fit for translation memory (segment-to-segment lookup with fuzzy matching). The glossary_terms table handles exact and prefix matches well but fuzzy/semantic matching is better served by vector search or dedicated TM systems.

6. **Partitioning complexity.** The messages, message_translations, and audit_log tables will grow large. PostgreSQL declarative partitioning (by tenant_id or created_at) is the answer, but it adds operational complexity for maintenance, backups, and migrations.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with logical replication for read replicas |
| **Connection Pooling** | PgBouncer in transaction mode for high-concurrency translation requests |
| **Partitioning** | Declarative range partitioning on `messages`, `message_translations`, and `audit_log` by `created_at` (monthly) |
| **Full-text Search** | PostgreSQL tsvector/tsquery for knowledge base article search; pg_trgm for fuzzy glossary matching |
| **Caching** | Redis for hot glossary term lookups and recently translated segments (translation memory cache) |
| **Migration Tool** | Flyway or golang-migrate for version-controlled schema migrations |
| **Row-Level Security** | PostgreSQL RLS policies on all tables using `current_setting('app.tenant_id')` for multi-tenant isolation |
| **Monitoring** | pg_stat_statements for query performance; custom metrics for translation latency and queue depth |

---

## Migration and Scaling Considerations

### Initial Deployment (0-100 tenants)

- Single PostgreSQL primary with one read replica
- PgBouncer for connection pooling
- Monthly partitioning on messages and translations tables
- Redis cache for glossary lookups with 5-minute TTL

### Growth Phase (100-1,000 tenants)

- Add read replicas for analytics queries (separate from transactional reads)
- Implement table partitioning on audit_log and message_translations
- Consider Citus extension for horizontal sharding by tenant_id if single-node capacity is approached
- Move daily aggregation jobs to async workers (pg_cron or external scheduler)
- Introduce a write-ahead buffer (e.g., Kafka) for high-volume message ingestion to smooth write spikes

### Scale Phase (1,000+ tenants)

- Citus distributed PostgreSQL for transparent sharding by tenant_id
- Dedicated analytics database (ClickHouse or TimescaleDB) fed by CDC (Change Data Capture) from the primary
- Separate read replicas per geographic region for data residency compliance
- Consider moving translation memory (fuzzy segment matching) to a dedicated vector search system (pgvector extension or standalone Qdrant) while keeping the relational model as the system of record
- Archive old conversations and translations to cold storage (S3 + Parquet) with metadata remaining in PostgreSQL for reference

### Data Residency

- For EU data residency requirements, deploy a separate PostgreSQL cluster in an EU region
- Use logical replication to sync tenant configuration and glossary data (non-PII) between regions
- Route tenant traffic to the appropriate regional cluster based on the tenant's `data_residency` setting

### Backup and Recovery

- Continuous WAL archiving to object storage (S3/GCS)
- Point-in-time recovery capability
- Daily logical backups of glossary and configuration tables (small, critical data)
- Test restore procedures monthly -- translation quality data and glossary terms are irreplaceable business assets
