# 11 — Database Schema

**Database:** PostgreSQL 16  
**Convention:** snake_case for all names; UUID primary keys; `client_id` on all data tables for multi-tenancy; `created_at` and `updated_at` on all tables.

---

## Schema Overview

```
core.*       — Mentions, Stories, Entities, Labels, Alerts, Campaigns
config.*     — ClientConfig, WatchTerms, Tiers, Outlets, Thresholds
accounts.*   — Users, Roles, Sessions
audit.*      — AuditLog, ChangeHistory
learning.*   — GoldSet, CapabilityMetrics, Corrections
```

---

## 1. `accounts` Schema

### `accounts.users`
```sql
CREATE TABLE accounts.users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    email           TEXT NOT NULL UNIQUE,
    display_name    TEXT NOT NULL,
    role            TEXT NOT NULL CHECK (role IN ('system_admin','war_room_admin','war_room_lead','team_member','cm_view')),
    language_pref   TEXT NOT NULL DEFAULT 'en' CHECK (language_pref IN ('en','ta')),
    whatsapp_number TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON accounts.users(client_id);
```

### `accounts.sessions`
```sql
CREATE TABLE accounts.sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES accounts.users(id),
    refresh_token   TEXT NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 2. `config` Schema

### `config.clients`
```sql
CREATE TABLE config.clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    -- UI vocabulary overrides
    vocab_party     TEXT NOT NULL DEFAULT 'party',
    vocab_ally      TEXT NOT NULL DEFAULT 'ally',
    vocab_summary   TEXT NOT NULL DEFAULT 'CM Summary',
    vocab_entity_lead TEXT NOT NULL DEFAULT 'CM',
    -- Co-worker identity
    coworker_name   TEXT NOT NULL DEFAULT 'Co-worker',
    coworker_avatar_url TEXT,
    -- Languages
    supported_languages TEXT[] NOT NULL DEFAULT ARRAY['en','ta'],
    -- Defaults
    min_sample_size INT NOT NULL DEFAULT 30,
    per_post_cap    NUMERIC(5,4) NOT NULL DEFAULT 0.05, -- 5%
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `config.entities`
```sql
CREATE TABLE config.entities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    name            TEXT NOT NULL,
    entity_type     TEXT NOT NULL CHECK (entity_type IN ('person','party','organisation')),
    -- For parties:
    relationship_tag TEXT CHECK (relationship_tag IN ('ally','supporting','opposition','neutral')),
    is_primary      BOOLEAN NOT NULL DEFAULT FALSE, -- TVK, CM Vijay, AadhavArjuna
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON config.entities(client_id);
```

### `config.entity_relationship_history`
```sql
-- Records relationship tag changes over time (historical tag preserved)
CREATE TABLE config.entity_relationship_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id       UUID NOT NULL REFERENCES config.entities(id),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    old_tag         TEXT,
    new_tag         TEXT NOT NULL,
    changed_by      UUID NOT NULL REFERENCES accounts.users(id),
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    effective_from  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `config.watch_terms`
```sql
CREATE TABLE config.watch_terms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    entity_id       UUID NOT NULL REFERENCES config.entities(id),
    term            TEXT NOT NULL,
    term_type       TEXT NOT NULL CHECK (term_type IN ('name','nickname','hashtag','tanglish','tamil')),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON config.watch_terms(client_id, entity_id);
```

### `config.standing_tiers`
```sql
CREATE TABLE config.standing_tiers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    tier_name       TEXT NOT NULL,
    tier_rank       INT NOT NULL, -- 1 = highest
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `config.tier_members`
```sql
CREATE TABLE config.tier_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tier_id         UUID NOT NULL REFERENCES config.standing_tiers(id),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    platform        TEXT NOT NULL CHECK (platform IN ('x','news','youtube')),
    handle_or_id    TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    added_by        UUID NOT NULL REFERENCES accounts.users(id),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON config.tier_members(client_id, tier_id);
```

### `config.outlets`
```sql
CREATE TABLE config.outlets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    name            TEXT NOT NULL,
    outlet_type     TEXT NOT NULL CHECK (outlet_type IN ('tamil_daily','english_daily','political_weekly','tv_news','national','youtube')),
    language        TEXT NOT NULL CHECK (language IN ('ta','en','both')),
    is_partisan     BOOLEAN NOT NULL DEFAULT FALSE,
    partisan_note   TEXT,
    feed_url        TEXT,
    crawl_url       TEXT,
    youtube_channel_id TEXT,
    crawl_enabled   BOOLEAN NOT NULL DEFAULT TRUE,
    robots_compliant BOOLEAN NOT NULL DEFAULT TRUE,
    tier_id         UUID REFERENCES config.standing_tiers(id),
    reach_estimate  TEXT, -- e.g. "high", "medium", "low" or numeric if known
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    last_crawled_at TIMESTAMPTZ,
    health_status   TEXT NOT NULL DEFAULT 'healthy' CHECK (health_status IN ('healthy','delayed','down')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `config.alert_thresholds`
```sql
CREATE TABLE config.alert_thresholds (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    threshold_key   TEXT NOT NULL,
    value_numeric   NUMERIC,
    value_int       INT,
    value_text      TEXT,
    description     TEXT,
    default_value   TEXT,
    updated_by      UUID REFERENCES accounts.users(id),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(client_id, threshold_key)
);
-- Example keys: spike_z_score, spike_min_mentions, spike_window_minutes,
-- spike_min_multiplier, big_screen_max_alerts, min_sample_size,
-- per_post_cap, ally_threshold, response_trigger_threshold
```

### `config.alert_recipients`
```sql
CREATE TABLE config.alert_recipients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    user_id         UUID NOT NULL REFERENCES accounts.users(id),
    alert_type      TEXT NOT NULL, -- 'tier1_all', 'response_opportunity', 'campaign', 'ally_mood'
    entity_id       UUID REFERENCES config.entities(id), -- NULL = all entities
    channel         TEXT NOT NULL CHECK (channel IN ('whatsapp','in_app')),
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `config.events_calendar`
```sql
CREATE TABLE config.events_calendar (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    event_name      TEXT NOT NULL,
    event_type      TEXT NOT NULL CHECK (event_type IN ('rally','assembly_session','results_day','other')),
    starts_at       TIMESTAMPTZ NOT NULL,
    ends_at         TIMESTAMPTZ NOT NULL,
    entity_id       UUID REFERENCES config.entities(id),
    custom_baseline_note TEXT,
    created_by      UUID NOT NULL REFERENCES accounts.users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 3. `core` Schema

### `core.mentions`
```sql
CREATE TABLE core.mentions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    source_type     TEXT NOT NULL CHECK (source_type IN ('x_post','news_article','epaper','clipping','youtube')),
    external_id     TEXT NOT NULL, -- X post ID, article URL, etc.
    outlet_id       UUID REFERENCES config.outlets(id),
    -- Content
    headline        TEXT,
    snippet         TEXT,
    full_text       TEXT,  -- Only if licensed
    language        TEXT NOT NULL CHECK (language IN ('ta','en','tanglish','mixed')),
    language_confidence NUMERIC(5,4),
    -- Author / account
    author_external_id  TEXT,
    author_name         TEXT,
    author_handle       TEXT,
    author_verified     BOOLEAN,
    author_follower_count INT,
    -- Engagement (X-specific)
    repost_count    INT,
    like_count      INT,
    reply_count     INT,
    view_count      INT,
    -- Source metadata
    published_at    TIMESTAMPTZ NOT NULL,
    source_url      TEXT,
    -- For e-paper/clipping
    epaper_page     INT,
    epaper_edition_date DATE,
    page_image_url  TEXT,
    -- For clipping
    uploaded_by     UUID REFERENCES accounts.users(id),
    -- OCR fields
    ocr_confidence  NUMERIC(5,4),
    ocr_flagged     BOOLEAN NOT NULL DEFAULT FALSE,
    -- Processing
    filter_matched  BOOLEAN NOT NULL DEFAULT FALSE,
    processing_stage TEXT NOT NULL DEFAULT 'raw', -- raw, filtered, labelled, weighed, story_matched
    -- Timing
    ingested_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    labelled_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(client_id, source_type, external_id)
);
CREATE INDEX ON core.mentions(client_id, published_at DESC);
CREATE INDEX ON core.mentions(client_id, source_type, published_at DESC);
CREATE INDEX ON core.mentions(client_id, filter_matched, processing_stage);
```

### `core.mention_labels`
```sql
CREATE TABLE core.mention_labels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mention_id      UUID NOT NULL REFERENCES core.mentions(id),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    entity_id       UUID NOT NULL REFERENCES config.entities(id),
    -- Labels
    stance          TEXT NOT NULL CHECK (stance IN ('positive','negative','neutral','sarcastic')),
    claim_type      TEXT NOT NULL CHECK (claim_type IN ('on_record','named_source','anonymous_insinuation','satire','other')),
    -- Provenance
    labelled_by     TEXT NOT NULL CHECK (labelled_by IN ('model_cheap','model_strong','human')),
    model_version   TEXT,
    prompt_version  TEXT,
    confidence      NUMERIC(5,4),
    reason          TEXT NOT NULL,
    -- State
    is_current      BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON core.mention_labels(mention_id, is_current);
CREATE INDEX ON core.mention_labels(client_id, entity_id, created_at DESC);
```

### `core.label_corrections`
```sql
CREATE TABLE core.label_corrections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    original_label_id UUID NOT NULL REFERENCES core.mention_labels(id),
    corrected_by    UUID NOT NULL REFERENCES accounts.users(id),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    -- New values
    new_stance      TEXT CHECK (stance IN ('positive','negative','neutral','sarcastic')),
    new_claim_type  TEXT,
    new_entity_id   UUID REFERENCES config.entities(id),
    -- Reason
    reason          TEXT NOT NULL,
    -- Second review
    requires_second_review BOOLEAN NOT NULL DEFAULT FALSE,
    second_review_status TEXT CHECK (second_review_status IN ('pending','approved','rejected')),
    second_reviewed_by UUID REFERENCES accounts.users(id),
    second_reviewed_at TIMESTAMPTZ,
    second_review_reason TEXT,
    -- State
    applied_at      TIMESTAMPTZ, -- NULL until second review approved (if required)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON core.label_corrections(client_id, created_at DESC);
```

### `core.mention_weights`
```sql
CREATE TABLE core.mention_weights (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    mention_id      UUID NOT NULL UNIQUE REFERENCES core.mentions(id),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    -- Weight components
    tier_id         UUID REFERENCES config.standing_tiers(id),
    tier_weight     NUMERIC(10,6),
    follower_weight NUMERIC(10,6),
    engagement_velocity NUMERIC(10,6),
    bot_suspicion   BOOLEAN NOT NULL DEFAULT FALSE,
    bot_down_weight NUMERIC(5,4),
    -- Final
    influence_score NUMERIC(10,6) NOT NULL,
    capped_contribution NUMERIC(5,4), -- after per-post cap
    weight_version  TEXT NOT NULL,
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `core.stories`
```sql
CREATE TABLE core.stories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    title           TEXT,
    auto_title      TEXT NOT NULL,
    -- Reach
    total_x_posts   INT NOT NULL DEFAULT 0,
    total_outlets   INT NOT NULL DEFAULT 0,
    total_mentions  INT NOT NULL DEFAULT 0,
    -- Status
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    merged_into     UUID REFERENCES core.stories(id),
    -- Response
    response_queue  BOOLEAN NOT NULL DEFAULT FALSE,
    suggested_response_type TEXT CHECK (suggested_response_type IN ('condemn','rebut','congratulate','condole','other')),
    response_status TEXT NOT NULL DEFAULT 'none' CHECK (response_status IN ('none','acted_on','not_needed')),
    response_marked_by UUID REFERENCES accounts.users(id),
    response_marked_at TIMESTAMPTZ,
    -- Campaign link
    campaign_id     UUID REFERENCES core.campaigns(id),
    -- Timestamps
    first_mention_at TIMESTAMPTZ NOT NULL,
    last_mention_at TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON core.stories(client_id, last_mention_at DESC);
CREATE INDEX ON core.stories(client_id, response_queue, response_status);
```

### `core.story_mentions`
```sql
CREATE TABLE core.story_mentions (
    story_id        UUID NOT NULL REFERENCES core.stories(id),
    mention_id      UUID NOT NULL REFERENCES core.mentions(id),
    added_by        TEXT NOT NULL DEFAULT 'auto' CHECK (added_by IN ('auto','manual')),
    added_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (story_id, mention_id)
);
```

### `core.alerts`
```sql
CREATE TABLE core.alerts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    alert_type      TEXT NOT NULL CHECK (alert_type IN ('volume_spike','influence_trigger','campaign','response_opportunity','ally_mood')),
    entity_id       UUID REFERENCES config.entities(id),
    story_id        UUID REFERENCES core.stories(id),
    campaign_id     UUID REFERENCES core.campaigns(id),
    -- Severity
    severity_tier   INT NOT NULL CHECK (severity_tier IN (1,2,3)),
    severity_score  NUMERIC(10,4),
    -- Co-worker explanation
    explanation     TEXT,
    -- State
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','acknowledged','resolved')),
    acknowledged_by UUID REFERENCES accounts.users(id),
    acknowledged_at TIMESTAMPTZ,
    assigned_to     UUID REFERENCES accounts.users(id),
    assigned_by     UUID REFERENCES accounts.users(id),
    assigned_at     TIMESTAMPTZ,
    -- Delivery
    whatsapp_sent   BOOLEAN NOT NULL DEFAULT FALSE,
    whatsapp_delivered BOOLEAN,
    whatsapp_failed_at TIMESTAMPTZ,
    whatsapp_retry_count INT NOT NULL DEFAULT 0,
    -- Metadata
    fired_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON core.alerts(client_id, severity_tier, status, fired_at DESC);
```

### `core.campaigns`
```sql
CREATE TABLE core.campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    entity_id       UUID NOT NULL REFERENCES config.entities(id),
    -- Detection
    confidence_score NUMERIC(5,4) NOT NULL,
    shared_phrasing TEXT[],
    account_count   INT NOT NULL,
    new_account_ratio NUMERIC(5,4), -- share of accounts under 3 months old
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Attribution
    attribution_status TEXT NOT NULL DEFAULT 'possible' CHECK (attribution_status IN ('possible','confirmed','dismissed')),
    attributed_to_entity_id UUID REFERENCES config.entities(id),
    confirmed_by    UUID REFERENCES accounts.users(id),
    confirmed_at    TIMESTAMPTZ,
    confirmation_reason TEXT,
    -- State
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### `core.campaign_accounts`
```sql
CREATE TABLE core.campaign_accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES core.campaigns(id),
    external_account_id TEXT NOT NULL,
    handle          TEXT,
    account_age_days INT,
    follower_count  INT,
    is_bot_suspected BOOLEAN NOT NULL DEFAULT FALSE,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON core.campaign_accounts(campaign_id);
```

---

## 4. `learning` Schema

### `learning.gold_set`
```sql
CREATE TABLE learning.gold_set (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    mention_id      UUID NOT NULL REFERENCES core.mentions(id),
    source          TEXT NOT NULL CHECK (source IN ('correction','manual_label','import')),
    -- Verified labels
    entity_id       UUID NOT NULL REFERENCES config.entities(id),
    stance          TEXT NOT NULL,
    claim_type      TEXT NOT NULL,
    language        TEXT NOT NULL,
    -- Provenance
    labelled_by     UUID NOT NULL REFERENCES accounts.users(id),
    labelled_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_included     BOOLEAN NOT NULL DEFAULT TRUE
);
```

### `learning.capability_metrics`
```sql
CREATE TABLE learning.capability_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    capability      TEXT NOT NULL, -- 'label_en','label_ta','label_tanglish','spike_alert','influence_alert','campaign_flag','response_suggestion','brief'
    week_start      DATE NOT NULL,
    stage           TEXT NOT NULL CHECK (stage IN ('assist','trusted','coworker')),
    f1_model_only   NUMERIC(5,4),
    f1_with_corrections NUMERIC(5,4),
    correction_rate NUMERIC(10,4), -- corrections per 1,000 labels
    accuracy_bar    NUMERIC(5,4),
    is_above_bar    BOOLEAN,
    calculated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(client_id, capability, week_start)
);
```

### `learning.coworker_interactions`
```sql
CREATE TABLE learning.coworker_interactions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    user_id         UUID NOT NULL REFERENCES accounts.users(id),
    question        TEXT NOT NULL,
    question_language TEXT NOT NULL,
    answer          TEXT NOT NULL,
    answer_language TEXT NOT NULL,
    model_version   TEXT NOT NULL,
    prompt_version  TEXT NOT NULL,
    source_mention_ids UUID[],
    latency_ms      INT,
    useful_rating   TEXT CHECK (useful_rating IN ('useful','not_useful')),
    rated_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON learning.coworker_interactions(client_id, created_at DESC);
```

---

## 5. `audit` Schema

### `audit.audit_log`
```sql
CREATE TABLE audit.audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES config.clients(id),
    actor_id        UUID REFERENCES accounts.users(id),
    actor_type      TEXT NOT NULL CHECK (actor_type IN ('user','system','coworker')),
    action          TEXT NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID,
    old_value       JSONB,
    new_value       JSONB,
    reason          TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX ON audit.audit_log(client_id, created_at DESC);
CREATE INDEX ON audit.audit_log(client_id, entity_type, entity_id);
```

---

## 6. Key Indexes and Constraints Summary

| Table | Key Indexes |
|-------|------------|
| `core.mentions` | `(client_id, published_at DESC)`, `(client_id, source_type, published_at DESC)`, `(client_id, filter_matched, processing_stage)` |
| `core.mention_labels` | `(mention_id, is_current)`, `(client_id, entity_id, created_at DESC)` |
| `core.alerts` | `(client_id, severity_tier, status, fired_at DESC)` |
| `core.stories` | `(client_id, last_mention_at DESC)`, `(client_id, response_queue, response_status)` |
| `config.watch_terms` | `(client_id, entity_id)` |
| `audit.audit_log` | `(client_id, created_at DESC)`, `(client_id, entity_type, entity_id)` |

**All foreign keys enforced with appropriate ON DELETE behaviour (mostly RESTRICT to prevent accidental data loss).**  
**Row-level security or client_id WHERE clause enforced at API layer for all queries.**  
**`updated_at` maintained via trigger on all core tables.**
