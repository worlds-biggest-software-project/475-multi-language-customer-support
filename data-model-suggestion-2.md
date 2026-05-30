# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Multi-Language Customer Support (Candidate #475)
> Generated: 2026-05-26

## Overview

This model applies Event Sourcing and Command Query Responsibility Segregation (CQRS) to the multilingual customer support domain. Every state change -- a message arriving, a translation being produced, a glossary term being corrected, a quality alert firing -- is captured as an immutable event in an append-only event store. The current state of any aggregate (conversation, glossary, knowledge base article) is derived by replaying its event stream. Read models (projections) are materialized into denormalized views optimized for specific query patterns: agent inbox views, translation quality dashboards, per-language analytics.

This architecture is particularly well-suited to the translation middleware domain because:

1. **Complete translation audit trail.** Every translation request, result, agent correction, and quality score is an immutable event. Regulatory compliance (GDPR, HIPAA) and translation quality forensics are built into the storage model rather than bolted on as an afterthought.

2. **Temporal queries are natural.** "What was the translation quality for Japanese three months ago?" is answered by replaying events, not by hoping someone built the right aggregation table.

3. **Multiple read models from the same events.** The agent inbox, the quality dashboard, the cost reporting view, and the glossary learning pipeline all consume the same event stream but project it differently. Adding a new analytical view does not require schema changes to the write side.

4. **Decoupled translation pipeline.** Translation requests are commands; translation results are events. The translation provider (Google, DeepL, Azure, Amazon) is an implementation detail that does not affect the event schema.

---

## Event Store Schema

The event store itself is a PostgreSQL table (though it could also use EventStoreDB, Apache Kafka, or a similar system). PostgreSQL is chosen here for operational simplicity and because it can also host the read model projections.

```sql
-- =============================================================
-- EVENT STORE (Write Side)
-- =============================================================

-- The single append-only event store table
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(50) NOT NULL,      -- 'Conversation', 'Glossary', 'KBArticle', etc.
    aggregate_id    UUID NOT NULL,             -- ID of the aggregate this event belongs to
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,     -- 'MessageReceived', 'TranslationCompleted', etc.
    event_version   INTEGER NOT NULL,          -- sequence number within the aggregate
    payload         JSONB NOT NULL,            -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- correlation IDs, causation, actor info
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Optimistic concurrency: no two events can have the same version for the same aggregate
    UNIQUE (aggregate_id, event_version)
);

-- Primary query path: replay all events for an aggregate
CREATE INDEX idx_events_aggregate ON events (aggregate_id, event_version ASC);

-- Tenant-scoped queries for projections
CREATE INDEX idx_events_tenant_type ON events (tenant_id, event_type, created_at DESC);

-- Global ordering for projection rebuilds
CREATE INDEX idx_events_created ON events (created_at ASC);

-- Partitioning: partition by month for manageability at scale
-- (Declarative partitioning shown conceptually; implement with CREATE TABLE ... PARTITION BY RANGE)

-- Snapshot store for aggregates with long event histories
CREATE TABLE snapshots (
    aggregate_id    UUID NOT NULL,
    aggregate_type  VARCHAR(50) NOT NULL,
    tenant_id       UUID NOT NULL,
    version         INTEGER NOT NULL,           -- event_version this snapshot reflects
    state           JSONB NOT NULL,             -- serialized aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (aggregate_id, version)
);

-- Idempotency tracking to prevent duplicate command processing
CREATE TABLE processed_commands (
    command_id      UUID PRIMARY KEY,
    aggregate_id    UUID NOT NULL,
    command_type    VARCHAR(100) NOT NULL,
    processed_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_processed_commands_aggregate ON processed_commands (aggregate_id);
```

---

## Event Types

### Conversation Aggregate Events

The Conversation aggregate represents a customer support interaction from creation through resolution. Its event stream captures every message, translation, routing decision, and outcome.

```
ConversationStarted
    tenant_id: UUID
    conversation_id: UUID
    channel_type: string          -- 'email', 'chat', 'voice', 'sms', etc.
    channel_id: UUID
    integration_id: UUID
    external_ticket_id: string
    customer_id: UUID
    customer_language: string     -- BCP 47 tag, e.g., 'ja-JP'
    subject: string (optional)
    initial_message_content: string
    detected_language: string     -- BCP 47 tag
    detection_confidence: number

ConversationAssigned
    conversation_id: UUID
    agent_id: UUID
    team_id: UUID (optional)
    agent_language: string        -- BCP 47 tag
    requires_translation: boolean
    assignment_reason: string     -- 'auto_routed', 'manual', 'escalated'

ConversationReassigned
    conversation_id: UUID
    previous_agent_id: UUID
    new_agent_id: UUID
    reason: string

ConversationPriorityChanged
    conversation_id: UUID
    previous_priority: string
    new_priority: string
    reason: string

CustomerMessageReceived
    conversation_id: UUID
    message_id: UUID
    sender_customer_id: UUID
    channel_id: UUID
    content: string
    content_type: string          -- 'text', 'html', 'markdown'
    language: string              -- BCP 47 tag
    detection_confidence: number
    has_attachments: boolean
    attachment_ids: UUID[]

AgentMessageSent
    conversation_id: UUID
    message_id: UUID
    sender_agent_id: UUID
    content: string
    content_type: string
    language: string              -- agent's language
    target_language: string       -- customer's language (for translation)

TranslationRequested
    conversation_id: UUID
    message_id: UUID
    translation_request_id: UUID
    source_language: string
    target_language: string
    source_content: string
    glossary_id: UUID (optional)
    provider_preference: string (optional)

TranslationCompleted
    conversation_id: UUID
    message_id: UUID
    translation_request_id: UUID
    translated_content: string
    provider: string              -- 'google', 'deepl', 'azure', 'amazon', 'internal'
    model_version: string
    confidence_score: number
    latency_ms: integer
    token_count: integer
    cost_usd: number
    glossary_terms_applied: object[]  -- [{source: "...", target: "...", term_id: "..."}]

TranslationFailed
    conversation_id: UUID
    message_id: UUID
    translation_request_id: UUID
    provider: string
    error_code: string
    error_message: string
    will_retry: boolean
    retry_provider: string (optional)

TranslationCorrectedByAgent
    conversation_id: UUID
    message_id: UUID
    agent_id: UUID
    original_translation: string
    corrected_translation: string
    correction_type: string       -- 'terminology', 'grammar', 'tone', 'context', 'cultural'
    source_segment: string
    corrected_segment: string

TranslationQualityFlagged
    conversation_id: UUID
    message_id: UUID
    flag_reason: string           -- 'low_confidence', 'auto_detected', 'agent_flagged'
    confidence_score: number
    flagged_by: string            -- 'system' or agent UUID

SentimentAnalyzed
    conversation_id: UUID
    message_id: UUID
    sentiment_score: number       -- -1.0 to 1.0
    emotion: string               -- 'neutral', 'frustrated', 'satisfied', 'urgent', 'confused'
    cultural_context: string      -- notes on culture-specific interpretation
    language: string

CSATSubmitted
    conversation_id: UUID
    customer_id: UUID
    score: integer                -- 1-5
    comment: string (optional)
    language: string

ConversationResolved
    conversation_id: UUID
    resolved_by_agent_id: UUID
    resolution_type: string       -- 'agent', 'bot', 'auto_closed', 'customer_withdrew'
    first_contact_resolution: boolean
    total_messages: integer
    total_translations: integer
    resolution_duration_seconds: integer

ConversationReopened
    conversation_id: UUID
    reopened_by: string           -- 'customer', 'agent', 'system'
    reason: string

ConversationClosed
    conversation_id: UUID
    closed_by: string
```

### Glossary Aggregate Events

The Glossary aggregate tracks terminology management, including manual curation, auto-learning from agent corrections, and industry pack installations.

```
GlossaryCreated
    glossary_id: UUID
    tenant_id: UUID
    name: string
    description: string
    industry: string (optional)

GlossaryUpdated
    glossary_id: UUID
    changes: object               -- {name?: string, description?: string}

GlossaryActivated / GlossaryDeactivated
    glossary_id: UUID

TermAdded
    glossary_id: UUID
    term_id: UUID
    source_language: string
    target_language: string
    source_term: string
    target_term: string
    case_sensitive: boolean
    match_type: string
    context_hint: string
    source: string                -- 'manual', 'auto_learned', 'imported', 'industry_pack'

TermUpdated
    glossary_id: UUID
    term_id: UUID
    previous_target_term: string
    new_target_term: string
    updated_by: UUID

TermRemoved
    glossary_id: UUID
    term_id: UUID
    reason: string

TermAutoLearned
    glossary_id: UUID
    term_id: UUID
    source_correction_ids: UUID[]  -- corrections that triggered this learning
    confidence: number
    occurrence_count: integer
    needs_approval: boolean

TermApproved
    glossary_id: UUID
    term_id: UUID
    approved_by: UUID

TermRejected
    glossary_id: UUID
    term_id: UUID
    rejected_by: UUID
    reason: string

IndustryPackInstalled
    glossary_id: UUID
    pack_id: UUID
    industry: string
    terms_imported: integer

TermUsageRecorded
    glossary_id: UUID
    term_id: UUID
    conversation_id: UUID
    message_id: UUID
```

### KBArticle Aggregate Events

```
ArticleCreated
    article_id: UUID
    tenant_id: UUID
    source_language: string
    title: string
    body: string
    body_format: string
    category: string
    tags: string[]

ArticleUpdated
    article_id: UUID
    version: integer
    title: string (optional)
    body: string (optional)
    content_hash: string          -- for change detection

ArticlePublished
    article_id: UUID
    version: integer

ArticleArchived
    article_id: UUID

ArticleTranslationRequested
    article_id: UUID
    target_language: string
    translation_method: string    -- 'machine', 'human', 'hybrid'
    source_version: integer

ArticleTranslationCompleted
    article_id: UUID
    language: string
    title: string
    body: string
    provider: string
    quality_score: number
    source_version: integer

ArticleTranslationOutdated
    article_id: UUID
    language: string
    reason: string                -- 'source_updated', 'quality_decline', 'manual'
    source_version_current: integer
    translation_source_version: integer

ArticleTranslationReviewed
    article_id: UUID
    language: string
    reviewed_by: UUID
    status: string                -- 'approved', 'needs_revision'
    comments: string

ArticleFeedbackReceived
    article_id: UUID
    language: string
    conversation_id: UUID (optional)
    feedback_type: string
    feedback_text: string (optional)
```

### Tenant Aggregate Events

```
TenantProvisioned
    tenant_id: UUID
    name: string
    slug: string
    plan_tier: string
    data_residency: string

TenantPlanChanged
    tenant_id: UUID
    previous_plan: string
    new_plan: string

TenantLanguageEnabled
    tenant_id: UUID
    language: string
    is_source: boolean
    is_target: boolean
    preferred_provider: string

TenantLanguageDisabled
    tenant_id: UUID
    language: string

IntegrationConnected
    tenant_id: UUID
    integration_id: UUID
    platform: string
    sync_direction: string

IntegrationDisconnected
    tenant_id: UUID
    integration_id: UUID
    reason: string
```

### Quality Monitoring Events

```
QualityAlertRaised
    tenant_id: UUID
    alert_id: UUID
    alert_type: string
    severity: string
    language: string (optional)
    source_language: string (optional)
    target_language: string (optional)
    metric_name: string
    metric_value: number
    threshold_value: number
    description: string

QualityAlertAcknowledged
    alert_id: UUID
    acknowledged_by: UUID

QualityAlertResolved
    alert_id: UUID
    resolved_by: UUID
    resolution_notes: string
```

---

## Read Model Projections

Read models are materialized views built by consuming the event stream. Each projection is optimized for a specific query pattern. They are fully rebuildable by replaying events from the beginning.

### Projection 1: Agent Inbox View

```sql
-- Denormalized view for the agent's inbox, showing conversations with
-- the latest message and translation status
CREATE TABLE rm_agent_inbox (
    conversation_id     UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    assigned_agent_id   UUID,
    assigned_team_id    UUID,
    customer_id         UUID,
    customer_name       VARCHAR(255),
    customer_email      VARCHAR(320),
    customer_language   VARCHAR(35),       -- BCP 47
    agent_language      VARCHAR(35),
    channel_type        VARCHAR(30),
    external_ticket_id  VARCHAR(255),
    subject             TEXT,
    status              VARCHAR(30),
    priority            VARCHAR(20),
    requires_translation BOOLEAN,
    last_message_content TEXT,              -- truncated to 500 chars
    last_message_sender  VARCHAR(20),       -- 'customer', 'agent'
    last_message_language VARCHAR(35),
    last_message_translated BOOLEAN,
    last_message_confidence NUMERIC(5,4),
    last_message_at     TIMESTAMPTZ,
    unread_count        INTEGER DEFAULT 0,
    total_messages      INTEGER DEFAULT 0,
    sentiment_score     NUMERIC(4,3),
    csat_score          SMALLINT,
    first_response_at   TIMESTAMPTZ,
    created_at          TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ
);

CREATE INDEX idx_rm_inbox_agent ON rm_agent_inbox (assigned_agent_id, status, updated_at DESC);
CREATE INDEX idx_rm_inbox_tenant ON rm_agent_inbox (tenant_id, status, updated_at DESC);
CREATE INDEX idx_rm_inbox_priority ON rm_agent_inbox (tenant_id, priority, updated_at DESC);
CREATE INDEX idx_rm_inbox_language ON rm_agent_inbox (customer_language);
```

### Projection 2: Conversation Detail View

```sql
-- Full conversation with all messages and translations, for rendering
-- the agent's conversation view with side-by-side original/translated text
CREATE TABLE rm_conversation_messages (
    message_id          UUID PRIMARY KEY,
    conversation_id     UUID NOT NULL,
    tenant_id           UUID NOT NULL,
    sequence_number     INTEGER NOT NULL,
    sender_type         VARCHAR(20),
    sender_id           UUID,
    sender_name         VARCHAR(255),
    original_content    TEXT,
    original_language   VARCHAR(35),
    translated_content  TEXT,
    target_language     VARCHAR(35),
    translation_provider VARCHAR(50),
    confidence_score    NUMERIC(5,4),
    quality_flag        VARCHAR(20),
    was_corrected       BOOLEAN DEFAULT false,
    corrected_content   TEXT,
    corrected_by_name   VARCHAR(255),
    sentiment_score     NUMERIC(4,3),
    emotion             VARCHAR(30),
    glossary_terms_applied JSONB,          -- [{source, target, term_id}]
    has_attachments     BOOLEAN DEFAULT false,
    attachment_urls     JSONB,
    created_at          TIMESTAMPTZ
);

CREATE INDEX idx_rm_conv_msgs ON rm_conversation_messages (conversation_id, sequence_number);
CREATE INDEX idx_rm_conv_msgs_tenant ON rm_conversation_messages (tenant_id);
```

### Projection 3: Translation Quality Dashboard

```sql
-- Hourly aggregated translation quality metrics for the monitoring dashboard
CREATE TABLE rm_translation_quality (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    hour                TIMESTAMPTZ NOT NULL,    -- truncated to hour
    source_language     VARCHAR(35) NOT NULL,
    target_language     VARCHAR(35) NOT NULL,
    provider            VARCHAR(50) NOT NULL,
    channel_type        VARCHAR(30),
    translation_count   INTEGER DEFAULT 0,
    avg_confidence      NUMERIC(5,4),
    min_confidence      NUMERIC(5,4),
    max_confidence      NUMERIC(5,4),
    low_confidence_count INTEGER DEFAULT 0,
    correction_count    INTEGER DEFAULT 0,
    failure_count       INTEGER DEFAULT 0,
    avg_latency_ms      NUMERIC(10,2),
    p95_latency_ms      NUMERIC(10,2),
    total_tokens        BIGINT DEFAULT 0,
    total_cost_usd      NUMERIC(12,4) DEFAULT 0,
    UNIQUE (tenant_id, hour, source_language, target_language, provider, channel_type)
);

CREATE INDEX idx_rm_tq_tenant_hour ON rm_translation_quality (tenant_id, hour DESC);
CREATE INDEX idx_rm_tq_langs ON rm_translation_quality (source_language, target_language);
```

### Projection 4: Per-Language Analytics

```sql
-- Daily per-language customer satisfaction and performance metrics
CREATE TABLE rm_language_analytics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    date                DATE NOT NULL,
    language            VARCHAR(35) NOT NULL,     -- BCP 47
    channel_type        VARCHAR(30),
    conversation_count  INTEGER DEFAULT 0,
    resolved_count      INTEGER DEFAULT 0,
    fcr_count           INTEGER DEFAULT 0,        -- first-contact resolution
    escalation_count    INTEGER DEFAULT 0,
    avg_csat            NUMERIC(4,2),
    avg_first_response_sec NUMERIC(12,2),
    avg_resolution_sec  NUMERIC(12,2),
    avg_sentiment       NUMERIC(4,3),
    translation_count   INTEGER DEFAULT 0,
    avg_translation_confidence NUMERIC(5,4),
    correction_count    INTEGER DEFAULT 0,
    total_translation_cost_usd NUMERIC(12,4) DEFAULT 0,
    UNIQUE (tenant_id, date, language, channel_type)
);

CREATE INDEX idx_rm_lang_analytics ON rm_language_analytics (tenant_id, date DESC);
CREATE INDEX idx_rm_lang_analytics_lang ON rm_language_analytics (language);
```

### Projection 5: Glossary Lookup View

```sql
-- Optimized for fast term lookup during translation pipeline processing
CREATE TABLE rm_glossary_terms (
    term_id             UUID PRIMARY KEY,
    glossary_id         UUID NOT NULL,
    tenant_id           UUID NOT NULL,
    glossary_name       VARCHAR(255),
    source_language     VARCHAR(35) NOT NULL,
    target_language     VARCHAR(35) NOT NULL,
    source_term         VARCHAR(1000) NOT NULL,
    target_term         VARCHAR(1000) NOT NULL,
    case_sensitive      BOOLEAN DEFAULT false,
    match_type          VARCHAR(20),
    context_hint        TEXT,
    is_do_not_translate BOOLEAN DEFAULT false,
    source              VARCHAR(20),
    is_approved         BOOLEAN DEFAULT true,
    usage_count         INTEGER DEFAULT 0,
    last_used_at        TIMESTAMPTZ,
    created_at          TIMESTAMPTZ,
    updated_at          TIMESTAMPTZ
);

CREATE INDEX idx_rm_glossary_lookup ON rm_glossary_terms (tenant_id, source_language,
    target_language, source_term);
CREATE INDEX idx_rm_glossary_glossary ON rm_glossary_terms (glossary_id);

-- Glossary learning candidates: corrections not yet promoted to terms
CREATE TABLE rm_glossary_candidates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    source_language     VARCHAR(35) NOT NULL,
    target_language     VARCHAR(35) NOT NULL,
    source_segment      VARCHAR(1000) NOT NULL,
    suggested_translation VARCHAR(1000) NOT NULL,
    occurrence_count    INTEGER DEFAULT 1,
    distinct_agents     INTEGER DEFAULT 1,
    first_seen_at       TIMESTAMPTZ,
    last_seen_at        TIMESTAMPTZ,
    confidence          NUMERIC(5,4),
    status              VARCHAR(20) DEFAULT 'candidate',  -- 'candidate', 'promoted', 'rejected'
    UNIQUE (tenant_id, source_language, target_language, source_segment)
);

CREATE INDEX idx_rm_candidates ON rm_glossary_candidates (tenant_id, status, occurrence_count DESC);
```

### Projection 6: Knowledge Base Translation Status

```sql
-- Shows the translation status of every article across all enabled languages
CREATE TABLE rm_kb_translation_status (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    article_id          UUID NOT NULL,
    article_title       VARCHAR(1000),
    article_category    VARCHAR(255),
    source_language     VARCHAR(35),
    source_version      INTEGER,
    target_language     VARCHAR(35) NOT NULL,
    translation_status  VARCHAR(30),       -- 'not_started', 'machine_translated', 'reviewed', 'outdated'
    translated_version  INTEGER,           -- source version the translation was based on
    is_outdated         BOOLEAN DEFAULT false,
    quality_score       NUMERIC(5,4),
    feedback_count      INTEGER DEFAULT 0,
    followup_question_count INTEGER DEFAULT 0,
    last_translated_at  TIMESTAMPTZ,
    last_reviewed_at    TIMESTAMPTZ,
    UNIQUE (article_id, target_language)
);

CREATE INDEX idx_rm_kb_status ON rm_kb_translation_status (tenant_id, translation_status);
CREATE INDEX idx_rm_kb_outdated ON rm_kb_translation_status (tenant_id)
    WHERE is_outdated = true;
```

### Projection 7: Active Quality Alerts

```sql
CREATE TABLE rm_active_alerts (
    alert_id            UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    alert_type          VARCHAR(50),
    severity            VARCHAR(20),
    source_language     VARCHAR(35),
    target_language     VARCHAR(35),
    description         TEXT,
    metric_name         VARCHAR(100),
    metric_value        NUMERIC(12,4),
    threshold_value     NUMERIC(12,4),
    is_acknowledged     BOOLEAN DEFAULT false,
    acknowledged_by_name VARCHAR(255),
    acknowledged_at     TIMESTAMPTZ,
    created_at          TIMESTAMPTZ
);

CREATE INDEX idx_rm_alerts_tenant ON rm_active_alerts (tenant_id, severity, created_at DESC);
CREATE INDEX idx_rm_alerts_unack ON rm_active_alerts (tenant_id)
    WHERE is_acknowledged = false;
```

---

## Event Processing Pipeline

```
                    +------------------+
                    |   Commands       |
                    | (API requests)   |
                    +--------+---------+
                             |
                    +--------v---------+
                    | Command Handlers |
                    | (validate,       |
                    |  load aggregate, |
                    |  apply rules)    |
                    +--------+---------+
                             |
                    +--------v---------+
                    |   Event Store    |
                    |   (PostgreSQL)   |
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v-----+ +-----v------+ +-----v--------+
     | Projection   | | Projection | | Projection   |
     | Builder:     | | Builder:   | | Builder:     |
     | Agent Inbox  | | Quality    | | Glossary     |
     |              | | Dashboard  | | Learning     |
     +--------------+ +------------+ +--------------+
              |              |              |
     +--------v-----+ +-----v------+ +-----v--------+
     | rm_agent_    | | rm_trans   | | rm_glossary  |
     | inbox        | | _quality   | | _candidates  |
     +--------------+ +------------+ +--------------+
```

### Event Dispatching

Events are dispatched to projection builders using one of two strategies:

1. **In-process subscriptions** (simpler, for early stage): After appending events to the store, the command handler notifies in-process subscribers that build projections synchronously or via an in-memory queue.

2. **Change Data Capture** (scalable, for growth stage): Use PostgreSQL logical replication or a CDC tool (Debezium) to stream new events to Apache Kafka topics, from which independent consumer groups build their projections. This allows projections to be rebuilt independently and at their own pace.

### Projection Rebuilding

Any projection can be torn down and rebuilt by replaying the entire event stream (or from a snapshot). This is a key advantage: if a new analytical view is needed (e.g., "cost per language pair per customer"), a new projection builder is created and pointed at the event stream. No schema migration is required on the write side.

---

## Pros and Cons

### Pros

1. **Immutable audit trail by default.** Every translation, correction, routing decision, and quality score is permanently recorded. GDPR data subject access requests can produce a complete history of all translations performed on a customer's data. HIPAA compliance for healthcare support conversations has a built-in event log without any additional auditing infrastructure.

2. **Temporal queries without pre-aggregation.** "What was the average translation confidence for Japanese-to-English in March?" can be answered by querying the event stream directly, even if no one thought to build that specific aggregation at the time. This is critical for a translation quality monitoring platform where new analytical questions emerge as the product matures.

3. **Decoupled translation pipeline.** The translation provider is behind a command/event boundary. Switching from Google Cloud Translation to DeepL for a specific language pair requires changing only the command handler, not the event schema or any read model. A/B testing translation providers is a natural extension: emit both results as events, let the quality monitoring projection compare them.

4. **Self-improving glossary is a natural event stream consumer.** Agent corrections emit `TranslationCorrectedByAgent` events. A glossary learning projection consumes these events, accumulates correction patterns, and when a threshold is reached (e.g., three agents corrected "cloud computing" to "computacion en la nube" instead of the machine's "informatica en la nube"), it generates a `TermAutoLearned` event. This is a textbook event-driven workflow.

5. **Independent scaling of reads and writes.** Translation requests (writes) scale independently from agent inbox queries (reads). During a support surge, the event store absorbs writes at high throughput while read projections update asynchronously. Agents see near-real-time updates without write-side contention.

6. **Safe schema evolution.** New event types can be added without affecting existing projections. Existing event payloads can be extended (new optional fields) without breaking existing consumers. This is critical for a middleware that must adapt to evolving helpdesk APIs and new translation providers.

### Cons

1. **Operational complexity.** An event-sourced system requires infrastructure for event dispatching, projection building, snapshot management, and projection rebuild orchestration. For a startup building an MVP, this is significant overhead compared to a simple CRUD model. The team needs experience with event-sourcing patterns to avoid common pitfalls (event schema versioning, projection lag, eventual consistency).

2. **Eventual consistency.** Read models are updated asynchronously after events are stored. An agent sending a message may not see the translation appear in their inbox for a few hundred milliseconds. For live chat with sub-second translation requirements, this lag must be carefully managed -- either by returning the translation in the command response (before the projection updates) or by using a synchronous in-process projection for the critical path.

3. **Event schema versioning is non-trivial.** As the product evolves, event schemas change. A `TranslationCompleted` event from 2026 may have different fields than one from 2028 (e.g., a new `cultural_tone_score` field). The system needs an upcasting mechanism to transform old events into the current schema during replay. This is solvable but requires discipline and tooling.

4. **Storage growth.** Every message, translation, and correction generates multiple events. A busy support operation with 10,000 conversations per day, averaging 8 messages per conversation, each with 2 translations, generates ~160,000+ events per day per tenant. At scale (1,000 tenants), this is 160 million events per day. Snapshots and archival strategies are essential.

5. **Debugging is harder.** When an agent reports "the translation was wrong," diagnosing the issue requires replaying the event stream for that conversation to understand what happened. Tooling for event stream inspection is necessary but often immature in early-stage projects.

6. **Projection rebuild time.** Rebuilding a projection from scratch (e.g., after a bug fix in the projection logic) requires replaying potentially billions of events. Snapshots help, but the rebuild for a large tenant can take hours. During this time, the projection is stale.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| **Event Store** | PostgreSQL 16+ (early stage) or EventStoreDB (growth stage) |
| **Event Bus** | Apache Kafka or Amazon Kinesis for cross-service event distribution |
| **CDC** | Debezium for PostgreSQL WAL-based change data capture |
| **Projection Storage** | PostgreSQL for relational projections; Redis for hot lookup caches (glossary terms, agent inbox counts) |
| **Snapshot Storage** | PostgreSQL JSONB (same database as event store) |
| **Command Processing** | Application-level command handlers in the API service |
| **Event Serialization** | JSON with schema registry (Confluent Schema Registry or custom) for versioning |
| **Monitoring** | Track projection lag (time between event creation and projection update); alert if lag exceeds thresholds |
| **Framework** | Axon Framework (Java), Marten (C#/.NET), or custom implementation with PostgreSQL advisory locks for aggregate consistency |

---

## Migration and Scaling Considerations

### Phase 1: MVP (PostgreSQL-Only Event Store)

- Single PostgreSQL database hosts both the event store and all read model projections
- In-process event dispatching (no Kafka needed yet)
- Synchronous projection updates for critical-path projections (agent inbox, conversation detail)
- Asynchronous projection updates for analytics (quality dashboard, language analytics)
- Snapshots every 100 events per aggregate to keep replay time manageable

### Phase 2: Growth (Introduce Event Bus)

- Debezium CDC streams new events from PostgreSQL to Kafka
- Projection builders become independent Kafka consumers
- Each projection can be rebuilt independently by resetting its Kafka consumer offset
- Introduce a separate PostgreSQL instance for read model projections (read/write split)
- Add Redis caching layer for glossary term lookups (hot path in translation pipeline)

### Phase 3: Scale (Distributed Event Processing)

- Partition Kafka topics by tenant_id for horizontal scaling
- Deploy projection builders as independent microservices that can be scaled per workload
- Move high-volume projections (translation quality metrics) to ClickHouse or TimescaleDB
- Archive old events to S3/GCS in Parquet format; keep recent events (e.g., 90 days) in PostgreSQL
- Implement event store sharding by tenant_id using Citus or application-level routing
- Consider EventStoreDB as a dedicated event store if PostgreSQL becomes a bottleneck for append-only writes

### Event Archival Strategy

- Events older than the retention window (configurable per tenant, default 1 year) are archived to object storage
- Snapshots are kept for all aggregates so that current state can be loaded without replaying archived events
- Archived events remain queryable via a cold-path query service (Athena/Presto over Parquet files)
- GDPR erasure: instead of deleting events (which breaks the append-only contract), use crypto-shredding -- encrypt event payloads with a per-customer key and destroy the key when erasure is requested

### GDPR Compliance with Event Sourcing

Event sourcing and GDPR erasure are inherently in tension (immutable events vs. right to be forgotten). The recommended approach:

1. **Crypto-shredding**: Encrypt PII fields in event payloads with a per-customer encryption key stored separately. When a customer exercises their right to erasure, destroy the encryption key. The events remain in the store but their PII is irrecoverable.

2. **Separate PII store**: Store PII (customer name, email, phone) in a mutable reference data store, not in event payloads. Events reference customers by ID only. Erasure deletes from the PII store; event integrity is preserved.

3. **Hybrid approach**: Use crypto-shredding for event payloads that contain message content (which may include customer PII in freeform text), and the separate PII store for structured customer data.
