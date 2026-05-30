# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

> Project: Multi-Language Customer Support (Candidate #475)
> Generated: 2026-05-26

## Overview

This model takes a pragmatic middle path: use normalized relational tables with strict schema enforcement for stable, well-understood domain entities (tenants, languages, agents, glossary terms), and use PostgreSQL JSONB columns for data that is inherently variable, tenant-customizable, or integration-specific (helpdesk metadata, channel-specific message attributes, per-provider translation responses, custom analytics dimensions).

The hybrid approach is particularly well-suited to a translation middleware because:

1. **Helpdesk integrations have wildly different data shapes.** A Zendesk ticket has different metadata fields than a Salesforce case, an Intercom conversation, or a Freshdesk ticket. Normalizing every platform's field set into relational columns would require constant schema migrations as integrations are added. JSONB absorbs this variability naturally.

2. **Translation provider responses vary.** Google Cloud Translation, DeepL, Azure Translator, and Amazon Translate return different response structures (confidence formats, alternative translations, detected features). Storing the full provider response as JSONB preserves all data for debugging and analytics without forcing a lowest-common-denominator schema.

3. **Per-tenant customization is core to the product.** Tenants configure custom routing rules, language preferences, quality thresholds, and integration settings. These vary per tenant and evolve as the product grows. JSONB handles this without ALTER TABLE.

4. **Message content needs flexible metadata.** Messages arrive from different channels (email, chat, voice transcription, SMS) with channel-specific attributes (email headers, chat session data, voice call duration). A single `channel_metadata` JSONB column handles all of these.

The key principle: **if you JOIN on it, index it, or enforce constraints on it, it's a column. If you query it occasionally, display it, or pass it through, it's JSONB.**

---

## Complete Schema Definition

### 1. Core Relational Tables (Stable Schema)

```sql
-- =====================================================
-- TENANCY AND ORGANIZATION
-- =====================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'starter'
                    CHECK (plan_tier IN ('starter', 'professional', 'enterprise')),
    data_residency  VARCHAR(10) NOT NULL DEFAULT 'us',
    -- JSONB: plan limits, feature flags, branding, notification preferences
    -- Example: {"max_agents": 50, "max_languages": 25,
    --           "features": {"voice_translation": true, "auto_glossary": true},
    --           "branding": {"primary_color": "#1a73e8"},
    --           "notifications": {"quality_alerts": "email", "weekly_digest": true}}
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    suspended_at    TIMESTAMPTZ
);

CREATE TABLE agents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    external_id     VARCHAR(255),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    primary_language_id UUID REFERENCES languages(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'agent',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: spoken languages with proficiency, shift schedule, skills,
    --        per-agent translation preferences, notification settings
    -- Example: {"languages": [{"bcp47": "en-US", "proficiency": "native"},
    --                          {"bcp47": "es-MX", "proficiency": "fluent"}],
    --           "skills": ["billing", "technical"],
    --           "preferences": {"show_original_text": true, "auto_translate": true}}
    profile         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_agents_tenant ON agents (tenant_id);
CREATE INDEX idx_agents_external ON agents (tenant_id, external_id);
-- GIN index on profile for querying agent skills and language proficiencies
CREATE INDEX idx_agents_profile ON agents USING GIN (profile jsonb_path_ops);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    -- JSONB: routing rules, escalation policies, language specializations
    -- Example: {"routing": {"languages": ["ja", "ko", "zh"],
    --                        "channels": ["chat", "email"]},
    --           "escalation": {"timeout_minutes": 30, "escalate_to_team": "..."}}
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    agent_id        UUID NOT NULL REFERENCES agents(id) ON DELETE CASCADE,
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (team_id, agent_id)
);


-- =====================================================
-- LANGUAGE AND LOCALE (fully relational -- this is core domain data)
-- =====================================================

CREATE TABLE languages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    iso_639_1       CHAR(2),
    iso_639_3       CHAR(3),
    bcp47_tag       VARCHAR(35) NOT NULL UNIQUE,
    name_english    VARCHAR(255) NOT NULL,
    name_native     VARCHAR(255),
    script          VARCHAR(10),
    direction       VARCHAR(3) NOT NULL DEFAULT 'ltr'
                    CHECK (direction IN ('ltr', 'rtl')),
    -- JSONB: CLDR locale data, plural rules, number/date formats
    -- Example: {"cldr": {"decimal_separator": ",", "grouping_separator": ".",
    --                     "date_format": "dd/MM/yyyy", "currency_symbol": "EUR"},
    --           "plural_rules": {"one": "n = 1", "other": ""},
    --           "formality_levels": ["formal", "informal"]}
    locale_data     JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_languages_bcp47 ON languages (bcp47_tag);
CREATE INDEX idx_languages_iso1 ON languages (iso_639_1);

CREATE TABLE language_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    parent_language_id UUID NOT NULL REFERENCES languages(id),
    variant_language_id UUID NOT NULL REFERENCES languages(id),
    relationship    VARCHAR(50) NOT NULL DEFAULT 'dialect',
    fallback_priority INTEGER NOT NULL DEFAULT 0,
    UNIQUE (parent_language_id, variant_language_id)
);

CREATE TABLE tenant_languages (
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    language_id     UUID NOT NULL REFERENCES languages(id),
    is_source       BOOLEAN NOT NULL DEFAULT false,
    is_target       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: per-tenant, per-language provider preferences, quality thresholds,
    --        formality settings, glossary overrides
    -- Example: {"preferred_provider": "deepl",
    --           "fallback_provider": "google",
    --           "quality_threshold": 0.85,
    --           "formality": "formal",
    --           "auto_approve_above": 0.95}
    config          JSONB NOT NULL DEFAULT '{}',
    enabled_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, language_id)
);


-- =====================================================
-- INTEGRATIONS AND CHANNELS
-- =====================================================

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    platform        VARCHAR(50) NOT NULL
                    CHECK (platform IN ('zendesk', 'salesforce', 'freshdesk',
                                        'intercom', 'servicenow', 'custom')),
    name            VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    -- JSONB: platform-specific configuration (API endpoints, field mappings,
    --        sync rules, authentication references)
    -- This is the prime example of where JSONB shines: Zendesk config looks
    -- completely different from Salesforce config
    -- Example (Zendesk): {"base_url": "https://acme.zendesk.com",
    --                      "auth": {"type": "oauth2", "vault_ref": "vault://..."},
    --                      "field_mapping": {"ticket_id": "id", "subject": "subject",
    --                                        "language": "custom_field_12345"},
    --                      "sync": {"direction": "bidirectional",
    --                               "ticket_statuses": ["new", "open", "pending"]},
    --                      "webhooks": [{"event": "ticket.created", "active": true}]}
    -- Example (Salesforce): {"instance_url": "https://acme.my.salesforce.com",
    --                         "auth": {"type": "jwt_bearer", "vault_ref": "vault://..."},
    --                         "objects": {"case": {"language_field": "Language__c",
    --                                              "translation_field": "Translated_Body__c"}},
    --                         "sync": {"direction": "bidirectional",
    --                                  "use_cdc": true}}
    config          JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_integrations_tenant ON integrations (tenant_id);
CREATE INDEX idx_integrations_platform ON integrations (tenant_id, platform);

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
    -- JSONB: channel-specific settings (email templates, chat widget config, etc.)
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_channels_tenant ON channels (tenant_id);
```

### 2. Conversations and Messages (Relational Core + JSONB Periphery)

```sql
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
    -- JSONB: CRM data, custom fields from helpdesk, interaction history summary,
    --        language history (languages the customer has used over time)
    -- Example: {"crm": {"account_id": "A-12345", "tier": "enterprise"},
    --           "language_history": [{"bcp47": "ja-JP", "first_seen": "2026-01-15",
    --                                 "message_count": 47},
    --                                {"bcp47": "en-US", "first_seen": "2026-03-01",
    --                                 "message_count": 3}],
    --           "custom_fields": {"industry": "healthcare", "region": "APAC"}}
    metadata        JSONB NOT NULL DEFAULT '{}',
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_tenant ON customers (tenant_id);
CREATE INDEX idx_customers_external ON customers (tenant_id, external_id);
CREATE INDEX idx_customers_email ON customers (tenant_id, email);


-- =====================================================
-- CONVERSATIONS
-- =====================================================

CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    channel_id      UUID REFERENCES channels(id),
    customer_id     UUID REFERENCES customers(id),
    assigned_agent_id UUID REFERENCES agents(id),
    assigned_team_id UUID REFERENCES teams(id),
    integration_id  UUID REFERENCES integrations(id),
    external_ticket_id VARCHAR(255),
    subject         TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open', 'pending', 'waiting_on_customer',
                                      'waiting_on_third_party', 'resolved', 'closed')),
    priority        VARCHAR(20) NOT NULL DEFAULT 'normal'
                    CHECK (priority IN ('low', 'normal', 'high', 'urgent')),
    customer_language_id UUID REFERENCES languages(id),
    agent_language_id UUID REFERENCES languages(id),
    requires_translation BOOLEAN NOT NULL DEFAULT true,
    -- Relational columns for the most-queried metrics
    sentiment_score NUMERIC(4,3),
    csat_score      SMALLINT CHECK (csat_score BETWEEN 1 AND 5),
    first_response_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    -- JSONB: helpdesk-specific fields, routing metadata, SLA tracking,
    --        custom tags from the external system, translation summary stats
    -- Example: {"external": {"zendesk_ticket_type": "incident",
    --                         "zendesk_group_id": "123456",
    --                         "zendesk_tags": ["vip", "billing"],
    --                         "zendesk_custom_fields": {"satisfaction_reason": "speed"}},
    --           "routing": {"routed_by": "language_match", "routing_score": 0.95},
    --           "sla": {"first_response_target_min": 60, "resolution_target_min": 480},
    --           "translation_summary": {"total_translations": 12,
    --                                    "avg_confidence": 0.923,
    --                                    "corrections": 1,
    --                                    "providers_used": ["deepl", "google"]}}
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_conversations_tenant ON conversations (tenant_id);
CREATE INDEX idx_conversations_status ON conversations (tenant_id, status);
CREATE INDEX idx_conversations_agent ON conversations (assigned_agent_id);
CREATE INDEX idx_conversations_customer ON conversations (customer_id);
CREATE INDEX idx_conversations_external ON conversations (tenant_id, external_ticket_id);
CREATE INDEX idx_conversations_created ON conversations (tenant_id, created_at DESC);
-- GIN index for querying external helpdesk metadata
CREATE INDEX idx_conversations_metadata ON conversations USING GIN (metadata jsonb_path_ops);


-- =====================================================
-- MESSAGES
-- =====================================================

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
    content_type    VARCHAR(20) NOT NULL DEFAULT 'text'
                    CHECK (content_type IN ('text', 'html', 'markdown')),
    is_internal_note BOOLEAN NOT NULL DEFAULT false,
    sequence_number INTEGER NOT NULL,
    -- JSONB: channel-specific message attributes that vary by channel type
    -- Email example: {"email": {"from": "user@example.com", "to": "support@acme.com",
    --                            "cc": ["manager@acme.com"], "subject": "Re: Order #123",
    --                            "message_id": "<abc@mail.example.com>",
    --                            "in_reply_to": "<def@mail.example.com>",
    --                            "headers": {"x-mailer": "Outlook 16.0"}}}
    -- Chat example: {"chat": {"session_id": "sess_abc123",
    --                          "page_url": "https://acme.com/pricing",
    --                          "browser": "Chrome 126", "os": "macOS 16.0"}}
    -- Voice example: {"voice": {"call_id": "call_xyz789",
    --                            "duration_seconds": 245,
    --                            "transcription_provider": "whisper",
    --                            "transcription_confidence": 0.94,
    --                            "is_transcription": true}}
    -- SMS example: {"sms": {"from_number": "+1555123456",
    --                        "carrier": "Verizon"}}
    channel_metadata JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_messages_conversation ON messages (conversation_id, sequence_number);
CREATE INDEX idx_messages_tenant ON messages (tenant_id);
CREATE INDEX idx_messages_created ON messages (conversation_id, created_at);


-- =====================================================
-- TRANSLATIONS (the critical pipeline output)
-- =====================================================

CREATE TABLE message_translations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    target_language_id UUID NOT NULL REFERENCES languages(id),
    translated_content TEXT NOT NULL,
    translation_provider VARCHAR(50) NOT NULL,
    confidence_score NUMERIC(5,4),
    quality_flag    VARCHAR(20) NOT NULL DEFAULT 'ok'
                    CHECK (quality_flag IN ('ok', 'low_confidence', 'needs_review',
                                            'reviewed', 'corrected')),
    latency_ms      INTEGER,
    token_count     INTEGER,
    cost_usd        NUMERIC(10,6),
    reviewed_by_agent_id UUID REFERENCES agents(id),
    reviewed_at     TIMESTAMPTZ,
    -- JSONB: full provider response for debugging, alternative translations,
    --        glossary terms applied, model-specific metadata
    -- Example: {"provider_response": {"detected_language": "ja",
    --                                   "model": "nmt-2026-03",
    --                                   "alternatives": [{"text": "...", "score": 0.87}]},
    --           "glossary_terms_applied": [{"source": "CloudSync Pro",
    --                                       "target": "CloudSync Pro",
    --                                       "term_id": "...",
    --                                       "type": "do_not_translate"}],
    --           "segments": [{"source": "...", "target": "...", "confidence": 0.95},
    --                        {"source": "...", "target": "...", "confidence": 0.78}],
    --           "quality_details": {"bleu_score": 0.82,
    --                               "flagged_segments": [2],
    --                               "cultural_warnings": ["formality_mismatch"]}}
    provider_details JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (message_id, target_language_id)
);

CREATE INDEX idx_msg_trans_message ON message_translations (message_id);
CREATE INDEX idx_msg_trans_quality ON message_translations (quality_flag)
    WHERE quality_flag != 'ok';
CREATE INDEX idx_msg_trans_provider ON message_translations (translation_provider);
-- GIN index for querying glossary terms applied or specific provider response fields
CREATE INDEX idx_msg_trans_details ON message_translations USING GIN (provider_details jsonb_path_ops);


-- =====================================================
-- AGENT CORRECTIONS
-- =====================================================

CREATE TABLE translation_corrections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    message_translation_id UUID NOT NULL REFERENCES message_translations(id),
    agent_id        UUID NOT NULL REFERENCES agents(id),
    source_language_id UUID NOT NULL REFERENCES languages(id),
    target_language_id UUID NOT NULL REFERENCES languages(id),
    original_segment TEXT NOT NULL,
    machine_translation TEXT NOT NULL,
    agent_correction TEXT NOT NULL,
    correction_type VARCHAR(30) NOT NULL DEFAULT 'terminology'
                    CHECK (correction_type IN ('terminology', 'grammar', 'tone',
                                               'context', 'cultural', 'factual')),
    -- JSONB: correction context, suggested glossary entry, agent notes
    -- Example: {"agent_notes": "This is our brand name, should not be translated",
    --           "suggested_term": {"source": "CloudSync", "target": "CloudSync",
    --                              "do_not_translate": true},
    --           "context": {"preceding_segment": "...", "following_segment": "..."}}
    details         JSONB NOT NULL DEFAULT '{}',
    promoted_to_term_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_corrections_tenant ON translation_corrections (tenant_id);
CREATE INDEX idx_corrections_langs ON translation_corrections (source_language_id,
    target_language_id);


-- =====================================================
-- ATTACHMENTS
-- =====================================================

CREATE TABLE message_attachments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    file_name       VARCHAR(512) NOT NULL,
    file_type       VARCHAR(100) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    storage_url     VARCHAR(2048) NOT NULL,
    is_translatable BOOLEAN NOT NULL DEFAULT false,
    -- JSONB: OCR results, document translation status, extracted text
    -- Example: {"ocr": {"provider": "google_vision", "text": "...", "confidence": 0.92},
    --           "translation": {"status": "completed", "translated_url": "..."}}
    processing      JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 3. Glossary and Terminology

```sql
-- =====================================================
-- GLOSSARIES
-- =====================================================

CREATE TABLE glossaries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    industry        VARCHAR(100),
    is_default      BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    -- JSONB: glossary-level settings (matching rules, auto-learn config,
    --        export format preferences)
    -- Example: {"matching": {"default_match_type": "exact",
    --                         "fuzzy_threshold": 0.85},
    --           "auto_learn": {"enabled": true,
    --                          "min_corrections": 3,
    --                          "min_distinct_agents": 2,
    --                          "auto_approve": false},
    --           "export": {"format": "tbx", "include_context": true}}
    config          JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_glossaries_tenant ON glossaries (tenant_id);

-- Glossary terms: fully relational because they are queried on every translation
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
    context_hint    TEXT,
    is_do_not_translate BOOLEAN NOT NULL DEFAULT false,
    confidence      NUMERIC(5,4) NOT NULL DEFAULT 1.0,
    source          VARCHAR(20) NOT NULL DEFAULT 'manual'
                    CHECK (source IN ('manual', 'auto_learned', 'imported', 'industry_pack')),
    approved_by     UUID REFERENCES agents(id),
    approved_at     TIMESTAMPTZ,
    usage_count     INTEGER NOT NULL DEFAULT 0,
    last_used_at    TIMESTAMPTZ,
    -- JSONB: additional context, related terms, usage examples, notes
    -- Example: {"examples": ["Use CloudSync Pro to backup your files",
    --                         "CloudSync Pro is available on all platforms"],
    --           "related_terms": ["CloudSync", "CloudSync Enterprise"],
    --           "notes": "Always capitalize 'Pro'",
    --           "regional_variants": {"es-ES": "CloudSync Pro",
    --                                  "es-419": "CloudSync Pro"}}
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_glossary_terms_lookup ON glossary_terms (glossary_id, source_language_id,
    target_language_id, source_term);
CREATE INDEX idx_glossary_terms_source ON glossary_terms (source_language_id, source_term);

-- Industry terminology packs (pre-built, tenant-installable)
CREATE TABLE industry_term_packs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    industry        VARCHAR(100) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    version         VARCHAR(20) NOT NULL DEFAULT '1.0',
    -- JSONB: pack contents stored as a document for bulk loading
    -- Example: {"terms": [{"source_lang": "en", "target_lang": "es",
    --                       "source": "deductible", "target": "deducible",
    --                       "context": "insurance terminology"},
    --                      {"source_lang": "en", "target_lang": "es",
    --                       "source": "copayment", "target": "copago"}]}
    terms_data      JSONB NOT NULL DEFAULT '{"terms": []}',
    term_count      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 4. Knowledge Base

```sql
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
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'archived')),
    version         INTEGER NOT NULL DEFAULT 1,
    content_hash    VARCHAR(64),
    -- JSONB: SEO metadata, custom fields, external system metadata
    -- Example: {"seo": {"meta_description": "...", "keywords": ["...", "..."]},
    --           "external": {"zendesk_section_id": "123", "zendesk_position": 5}}
    metadata        JSONB NOT NULL DEFAULT '{}',
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_kb_articles_tenant ON kb_articles (tenant_id);
CREATE INDEX idx_kb_articles_status ON kb_articles (tenant_id, status);
CREATE INDEX idx_kb_articles_tags ON kb_articles USING GIN (tags);

CREATE TABLE kb_article_translations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID NOT NULL REFERENCES kb_articles(id) ON DELETE CASCADE,
    language_id     UUID NOT NULL REFERENCES languages(id),
    title           VARCHAR(1000) NOT NULL,
    body            TEXT NOT NULL,
    translation_method VARCHAR(30) NOT NULL DEFAULT 'machine',
    translation_provider VARCHAR(50),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'machine_translated', 'human_reviewed',
                                      'published', 'outdated', 'archived')),
    source_version  INTEGER NOT NULL,
    is_outdated     BOOLEAN NOT NULL DEFAULT false,
    quality_score   NUMERIC(5,4),
    reviewed_by     UUID REFERENCES agents(id),
    reviewed_at     TIMESTAMPTZ,
    -- JSONB: translation metadata, segment-level quality, provider response
    -- Example: {"provider_response": {"model": "deepl-2026-q1"},
    --           "segment_scores": [{"segment": 1, "confidence": 0.97},
    --                              {"segment": 2, "confidence": 0.72}],
    --           "review_notes": "Adjusted technical terminology in section 3"}
    translation_details JSONB NOT NULL DEFAULT '{}',
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (article_id, language_id)
);

CREATE INDEX idx_kb_trans_article ON kb_article_translations (article_id);
CREATE INDEX idx_kb_trans_outdated ON kb_article_translations (is_outdated) WHERE is_outdated = true;

-- Knowledge gap detection: which articles cause follow-up questions per language
CREATE TABLE kb_article_feedback (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    article_id      UUID NOT NULL REFERENCES kb_articles(id) ON DELETE CASCADE,
    language_id     UUID NOT NULL REFERENCES languages(id),
    conversation_id UUID REFERENCES conversations(id),
    feedback_type   VARCHAR(30) NOT NULL
                    CHECK (feedback_type IN ('helpful', 'not_helpful', 'followup_question',
                                             'wrong_language', 'outdated', 'unclear')),
    -- JSONB: feedback context, the follow-up question text, sentiment
    -- Example: {"question_text": "How do I configure SSO?",
    --           "sentiment": "confused",
    --           "page_section": "authentication"}
    details         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_kb_feedback ON kb_article_feedback (article_id, language_id);
```

### 5. Analytics and Monitoring

```sql
-- Daily translation quality metrics (relational for fast aggregation)
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
    p95_latency_ms  NUMERIC(10,2),
    total_tokens    BIGINT NOT NULL DEFAULT 0,
    total_cost_usd  NUMERIC(12,4) NOT NULL DEFAULT 0,
    -- JSONB: histogram data, percentile distributions, provider-specific metrics
    -- Example: {"confidence_histogram": {"0.5-0.6": 2, "0.6-0.7": 5,
    --                                     "0.7-0.8": 15, "0.8-0.9": 45,
    --                                     "0.9-1.0": 133},
    --           "latency_percentiles": {"p50": 120, "p75": 180, "p95": 340, "p99": 890},
    --           "error_breakdown": {"timeout": 2, "rate_limit": 1, "invalid_input": 0}}
    details         JSONB NOT NULL DEFAULT '{}',
    UNIQUE (tenant_id, date, source_language_id, target_language_id,
            translation_provider, channel_type)
);

CREATE INDEX idx_tq_daily ON translation_quality_daily (tenant_id, date DESC);

-- Daily conversation metrics per language (relational core)
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
    -- JSONB: detailed breakdowns, sentiment distribution, CSAT distribution
    -- Example: {"csat_distribution": {"1": 2, "2": 3, "3": 8, "4": 25, "5": 62},
    --           "sentiment_distribution": {"frustrated": 5, "neutral": 40,
    --                                       "satisfied": 45, "delighted": 10},
    --           "escalation_reasons": {"language_mismatch": 2, "complex_issue": 5}}
    details         JSONB NOT NULL DEFAULT '{}',
    UNIQUE (tenant_id, date, language_id, channel_type)
);

CREATE INDEX idx_cm_daily ON conversation_metrics_daily (tenant_id, date DESC);

-- Quality alerts
CREATE TABLE quality_alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    alert_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL DEFAULT 'warning',
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
    -- JSONB: alert context, supporting data points, recommended actions
    -- Example: {"trend_data": [{"date": "2026-05-20", "value": 0.92},
    --                           {"date": "2026-05-21", "value": 0.89},
    --                           {"date": "2026-05-22", "value": 0.85}],
    --           "affected_conversations": ["conv_123", "conv_456"],
    --           "recommended_action": "Review glossary terms for medical terminology"}
    context         JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_alerts_tenant ON quality_alerts (tenant_id, created_at DESC);
CREATE INDEX idx_alerts_unack ON quality_alerts (tenant_id) WHERE is_acknowledged = false;
```

### 6. Audit Trail

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    actor_type      VARCHAR(20) NOT NULL,
    actor_id        UUID,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    -- JSONB: action-specific details (what changed, before/after values)
    -- Example: {"before": {"priority": "normal"}, "after": {"priority": "high"},
    --           "reason": "Customer escalated via phone"}
    details         JSONB,
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log (resource_type, resource_id);

-- GDPR data subject requests
CREATE TABLE data_subject_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    customer_id     UUID NOT NULL REFERENCES customers(id),
    request_type    VARCHAR(20) NOT NULL
                    CHECK (request_type IN ('access', 'erasure', 'portability', 'rectification')),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending',
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at    TIMESTAMPTZ,
    -- JSONB: processing log, what was exported/deleted
    processing_log  JSONB NOT NULL DEFAULT '[]',
    notes           TEXT
);
```

---

## JSONB Query Patterns

The hybrid model relies on PostgreSQL's JSONB operators being efficient and well-indexed. Here are the key query patterns used in this schema:

### Querying Integration Config

```sql
-- Find all Zendesk integrations that sync bidirectionally
SELECT id, name, config
FROM integrations
WHERE tenant_id = $1
  AND platform = 'zendesk'
  AND config @> '{"sync": {"direction": "bidirectional"}}';

-- Get the language field mapping for a Salesforce integration
SELECT config #>> '{objects,case,language_field}' AS language_field
FROM integrations
WHERE id = $1;
```

### Querying Message Channel Metadata

```sql
-- Find all voice messages longer than 5 minutes in a conversation
SELECT id, original_content, channel_metadata
FROM messages
WHERE conversation_id = $1
  AND (channel_metadata -> 'voice' ->> 'duration_seconds')::integer > 300;

-- Get email thread for a conversation
SELECT id, channel_metadata -> 'email' ->> 'subject' AS subject,
       channel_metadata -> 'email' ->> 'message_id' AS email_message_id
FROM messages
WHERE conversation_id = $1
  AND channel_metadata ? 'email'
ORDER BY sequence_number;
```

### Querying Translation Provider Details

```sql
-- Find translations where glossary terms were applied
SELECT mt.id, mt.translated_content, mt.provider_details -> 'glossary_terms_applied'
FROM message_translations mt
WHERE mt.message_id = $1
  AND jsonb_array_length(mt.provider_details -> 'glossary_terms_applied') > 0;

-- Find translations with cultural warnings
SELECT mt.id, mt.provider_details -> 'quality_details' -> 'cultural_warnings'
FROM message_translations mt
JOIN messages m ON m.id = mt.message_id
WHERE m.conversation_id = $1
  AND mt.provider_details @> '{"quality_details": {"cultural_warnings": []}}';
```

### Querying Analytics Details

```sql
-- Get confidence histogram for a language pair on a specific date
SELECT details -> 'confidence_histogram' AS histogram
FROM translation_quality_daily
WHERE tenant_id = $1
  AND date = '2026-05-25'
  AND source_language_id = $2
  AND target_language_id = $3;

-- Get CSAT distribution for a language
SELECT details -> 'csat_distribution' AS csat_dist
FROM conversation_metrics_daily
WHERE tenant_id = $1
  AND date = '2026-05-25'
  AND language_id = $2;
```

---

## Pros and Cons

### Pros

1. **Best of both worlds for integration variability.** The core domain (conversations, messages, glossary terms, languages) is fully relational with foreign keys, constraints, and indexes. Integration-specific metadata (Zendesk custom fields, Salesforce objects, Freshdesk solution categories) lives in JSONB and never requires schema migrations when a new helpdesk connector is added.

2. **Single database technology.** PostgreSQL handles both the relational and document aspects, avoiding the operational complexity of running a separate document database (MongoDB, DynamoDB) alongside a relational store. One backup strategy, one monitoring setup, one connection pool.

3. **JSONB is queryable and indexable.** GIN indexes on JSONB columns support containment queries (@>) and existence checks (?) efficiently. Partial indexes on specific JSONB paths further optimize hot queries. This is not a "dump bucket" -- it is structured, queryable data with a different schema enforcement model.

4. **Graceful schema evolution.** When a new translation provider is added, its unique response fields are stored in the `provider_details` JSONB column. No ALTER TABLE, no migration, no downtime. When those fields become important enough to query frequently, they can be promoted to dedicated columns in a later migration.

5. **Rich analytics without separate data warehouse (initially).** The `details` JSONB columns on analytics tables store histogram data, percentile distributions, and breakdown details that would otherwise require separate dimension tables. This keeps the analytics schema simple while still supporting drill-down queries.

6. **Per-tenant customization without schema changes.** Tenants can have custom routing rules, quality thresholds, and feature flags in the `settings` JSONB. Agents can have custom preferences and skill tags in their `profile` JSONB. No tenant-specific columns or extension tables needed.

7. **GDPR compliance with audit trail.** The audit log stores before/after snapshots in JSONB `details`, capturing the full context of every change without requiring a separate audit table per entity.

### Cons

1. **No schema enforcement on JSONB.** Unlike relational columns with CHECK constraints and NOT NULL, JSONB data can contain any structure. A bug in the API layer could write malformed integration config or incorrect provider response data. Mitigation: application-level validation (JSON Schema) and database CHECK constraints on JSONB columns using `jsonb_typeof` and `jsonb_path_exists`.

2. **JSONB query performance is lower than columnar access.** Querying `config ->> 'preferred_provider'` is slower than querying a dedicated `preferred_provider` column. For hot-path queries (glossary term lookup during translation), the critical fields are relational columns. JSONB is used for less-frequently-queried metadata.

3. **Temptation to over-use JSONB.** Without discipline, developers may store data in JSONB that belongs in columns (e.g., putting `confidence_score` in JSONB instead of a dedicated column). The principle "if you JOIN on it or index it, it's a column" must be enforced through code review and documentation.

4. **Backup and restore of JSONB is opaque.** Unlike relational columns that can be restored selectively, JSONB blobs are all-or-nothing. If a bug corrupts the `provider_details` JSONB for a batch of translations, restoring just that field requires JSONB manipulation, not a simple column-level restore.

5. **Reporting tools may struggle with JSONB.** Business intelligence tools (Metabase, Looker, Tableau) are optimized for flat relational schemas. Extracting data from JSONB columns requires SQL expressions (`->>`, `#>>`, `jsonb_array_elements`) that some BI tools handle poorly. Mitigation: create materialized views that flatten JSONB into columns for BI consumption.

6. **Migration path from JSONB to columns is manual.** Promoting a frequently-queried JSONB field to a dedicated column requires a data migration (extract from JSONB, populate column, update application code). This is straightforward but must be planned.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Database** | PostgreSQL 16+ with JSONB support (mature since PostgreSQL 9.4, continuously improved) |
| **JSONB Validation** | Application-level JSON Schema validation before writes; `CHECK` constraints using `jsonb_path_exists` for critical JSONB structures |
| **Indexing Strategy** | GIN indexes with `jsonb_path_ops` on JSONB columns that are queried; partial indexes on specific JSONB paths for hot queries |
| **ORM** | Prisma, Drizzle, or SQLAlchemy with native JSONB support; avoid ORMs that treat JSONB as opaque strings |
| **Connection Pooling** | PgBouncer in transaction mode |
| **Caching** | Redis for glossary term lookups and tenant settings (JSONB parsed once, cached as structured objects) |
| **BI/Reporting** | Materialized views that flatten JSONB into columnar format for BI tool consumption; refresh on schedule |
| **Full-Text Search** | PostgreSQL tsvector for KB articles; pg_trgm for fuzzy glossary term matching |
| **Monitoring** | Track JSONB column sizes (pg_column_size) to detect bloat; monitor GIN index size and rebuild frequency |

---

## Migration and Scaling Considerations

### JSONB Column Size Management

JSONB columns can grow large if unconstrained. Recommended limits:

| Table | JSONB Column | Expected Size | Hard Limit |
|-------|-------------|---------------|------------|
| `tenants.settings` | Tenant config | 2-10 KB | 100 KB |
| `agents.profile` | Agent preferences | 1-5 KB | 50 KB |
| `integrations.config` | Platform config | 5-20 KB | 200 KB |
| `messages.channel_metadata` | Channel-specific data | 0.5-5 KB | 50 KB |
| `message_translations.provider_details` | Provider response | 2-20 KB | 100 KB |
| `conversations.metadata` | Helpdesk metadata | 1-10 KB | 100 KB |

Enforce limits with CHECK constraints:

```sql
ALTER TABLE messages
    ADD CONSTRAINT chk_channel_metadata_size
    CHECK (pg_column_size(channel_metadata) <= 51200);  -- 50 KB
```

### Promoting JSONB Fields to Columns

When a JSONB field becomes a frequent query target, promote it to a dedicated column:

```sql
-- Example: promoting voice call duration from JSONB to a column
ALTER TABLE messages ADD COLUMN voice_duration_seconds INTEGER;

UPDATE messages
SET voice_duration_seconds = (channel_metadata -> 'voice' ->> 'duration_seconds')::integer
WHERE channel_metadata ? 'voice';

CREATE INDEX idx_messages_voice_duration ON messages (voice_duration_seconds)
    WHERE voice_duration_seconds IS NOT NULL;
```

### Scaling Path

1. **0-100 tenants**: Single PostgreSQL instance with read replica
2. **100-500 tenants**: Add PgBouncer, materialized views for analytics, Redis for hot lookups
3. **500-2000 tenants**: Citus for sharding by tenant_id (JSONB columns shard transparently)
4. **2000+ tenants**: Separate analytics database (ClickHouse) fed by CDC; archive old JSONB-heavy rows to object storage; consider promoting the most-queried JSONB fields to columns based on usage patterns

### Data Residency

- Same approach as the normalized model: regional PostgreSQL clusters with logical replication for non-PII configuration data
- JSONB columns containing customer data (message content, CRM metadata) stay in the tenant's designated region
- JSONB columns containing system configuration (integration config, glossary settings) can be replicated across regions
