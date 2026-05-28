# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Influencer Marketing Platform · Created: 2026-05-19

## Philosophy

This model treats every state change as an immutable event appended to a central event store. The event stream is the single source of truth; all queryable state is derived by projecting (replaying) events into materialised read models. This is the Command Query Responsibility Segregation (CQRS) pattern: writes go to the event store, reads come from purpose-built projections.

Event sourcing is used in financial systems, compliance-heavy platforms, and any domain where "what happened and when" matters as much as "what is true now." For an influencer marketing platform subject to FTC audit requirements, GDPR data subject access requests, and financial payment records, having a complete, tamper-evident history of every change is a natural fit.

The trade-off is operational complexity. Projections must be maintained, event schemas must be versioned, and developers must think in terms of events rather than CRUD operations. However, the payoff is extraordinary audit capability, the ability to answer temporal queries ("what was this creator's fraud score on January 15th?"), and the flexibility to build new read models from existing events without migrating data.

**Best for:** Teams building for regulated environments where full audit trails, temporal queries, and compliance forensics are primary requirements.

**Trade-offs:**
- Pro: Complete, immutable audit trail — every change recorded with timestamp, actor, and context
- Pro: Temporal queries are trivial — replay events to any point in time
- Pro: New read models can be built from existing events without data migration
- Pro: Natural fit for FTC compliance auditing and GDPR subject access requests
- Con: Higher operational complexity — projections must be maintained and monitored
- Con: Event schema evolution requires careful versioning (upcasting)
- Con: Eventually consistent read models — slight delay between write and read
- Con: Larger storage footprint — events are never deleted, only compacted
- Con: Steeper learning curve for developers unfamiliar with event-sourced systems

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IAB Creator Economy Taxonomy (April 2025) | Event payloads include IAB-standard creator tier and campaign type classifications |
| ISO 3166-1 alpha-2 | Country codes in location-related events |
| ISO 4217 | Currency codes in all financial events |
| ISO 8601 | All event timestamps in UTC with timezone offset |
| FTC 16 CFR Part 255 | Compliance events create tamper-evident audit chain for regulatory review |
| GDPR (EU) 2016/679 | Subject access requests answered by replaying all events for a given creator |
| OWASP API Security | Event store access controlled via application-layer authorization |

---

## Core Event Store

### Events Table (Append-Only)

```sql
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,              -- aggregate root ID
    stream_type     VARCHAR(50) NOT NULL,        -- creator, campaign, content_post, payment, outreach
    event_type      VARCHAR(100) NOT NULL,       -- e.g. creator_profile_synced, campaign_created
    event_version   INTEGER NOT NULL,            -- schema version for this event type
    sequence_number BIGINT NOT NULL,             -- ordering within a stream
    payload         JSONB NOT NULL,              -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}', -- actor, ip, correlation_id, causation_id
    organization_id UUID NOT NULL,               -- tenant isolation
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, sequence_number)
);

-- Partitioned by month for performance on large event stores
-- CREATE TABLE events_2026_05 PARTITION OF events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_events_stream ON events(stream_id, sequence_number);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_org ON events(organization_id);
CREATE INDEX idx_events_occurred ON events(occurred_at);
CREATE INDEX idx_events_stream_type ON events(stream_type, occurred_at);
```

### Event Type Examples

```sql
-- Example event payloads (stored in the 'payload' JSONB column):

-- creator_registered
-- {
--   "creator_id": "uuid",
--   "full_name": "Jane Doe",
--   "primary_email": "jane@example.com",
--   "country_code": "US",
--   "tier": "micro",
--   "source": "manual_import"
-- }

-- creator_profile_synced
-- {
--   "creator_id": "uuid",
--   "platform": "instagram",
--   "username": "janedoe",
--   "follower_count": 45000,
--   "avg_engagement_rate": 3.52,
--   "fraud_score": 12.5,
--   "authenticity_score": 87.3,
--   "audience_demographics": {
--     "age_18_24_pct": 35.2,
--     "age_25_34_pct": 42.1,
--     "gender_female_pct": 68.0,
--     "top_countries": [{"code": "US", "pct": 45.0}, {"code": "GB", "pct": 12.3}]
--   }
-- }

-- campaign_created
-- {
--   "campaign_id": "uuid",
--   "name": "Summer Beauty Launch",
--   "campaign_type": "sponsored_post",
--   "budget_total": 50000.00,
--   "budget_currency": "USD",
--   "target_platforms": ["instagram", "tiktok"]
-- }

-- creator_invited_to_campaign
-- {
--   "campaign_id": "uuid",
--   "creator_id": "uuid",
--   "agreed_rate": 2500.00,
--   "rate_currency": "USD",
--   "rate_type": "per_post"
-- }

-- content_published
-- {
--   "content_post_id": "uuid",
--   "campaign_id": "uuid",
--   "creator_id": "uuid",
--   "platform": "instagram",
--   "post_type": "reel",
--   "post_url": "https://instagram.com/p/...",
--   "published_at": "2026-05-15T14:30:00Z"
-- }

-- compliance_check_completed
-- {
--   "content_post_id": "uuid",
--   "check_type": "ftc_disclosure",
--   "result": "fail",
--   "details": "Missing #ad or #sponsored hashtag in caption",
--   "checked_by": "ai"
-- }

-- payment_processed
-- {
--   "payment_id": "uuid",
--   "campaign_creator_id": "uuid",
--   "amount": 2500.00,
--   "currency": "USD",
--   "payment_method": "stripe",
--   "external_payment_id": "pi_abc123"
-- }
```

---

## Materialised Read Models (Projections)

These tables are derived from events and can be rebuilt at any time by replaying the event store.

### Creator Read Model

```sql
CREATE TABLE rm_creators (
    id                  UUID PRIMARY KEY,
    full_name           VARCHAR(255),
    primary_email       VARCHAR(255),
    country_code        CHAR(2),
    tier                VARCHAR(20),
    fraud_score         NUMERIC(5,2),
    authenticity_score  NUMERIC(5,2),
    is_synthetic        BOOLEAN NOT NULL DEFAULT false,
    total_campaigns     INTEGER NOT NULL DEFAULT 0,
    total_earnings      NUMERIC(12,2) NOT NULL DEFAULT 0,
    avg_campaign_rating NUMERIC(3,2),
    last_event_at       TIMESTAMPTZ,
    projection_version  BIGINT NOT NULL DEFAULT 0  -- last processed event sequence
);

CREATE INDEX idx_rm_creators_tier ON rm_creators(tier);
CREATE INDEX idx_rm_creators_fraud ON rm_creators(fraud_score);
```

### Creator Profile Read Model

```sql
CREATE TABLE rm_creator_profiles (
    id                  UUID PRIMARY KEY,
    creator_id          UUID NOT NULL,
    platform            VARCHAR(50) NOT NULL,
    username            VARCHAR(255),
    follower_count      BIGINT,
    avg_engagement_rate NUMERIC(7,4),
    avg_views           BIGINT,
    growth_rate_30d     NUMERIC(7,4),
    audience_age_json   JSONB,          -- {"18-24": 35.2, "25-34": 42.1, ...}
    audience_gender_json JSONB,         -- {"male": 32.0, "female": 68.0}
    audience_geo_json   JSONB,          -- [{"code": "US", "pct": 45.0}, ...]
    last_synced_at      TIMESTAMPTZ,
    projection_version  BIGINT NOT NULL DEFAULT 0,
    UNIQUE(creator_id, platform)
);

CREATE INDEX idx_rm_profiles_creator ON rm_creator_profiles(creator_id);
CREATE INDEX idx_rm_profiles_platform ON rm_creator_profiles(platform);
CREATE INDEX idx_rm_profiles_followers ON rm_creator_profiles(follower_count);
```

### Campaign Read Model

```sql
CREATE TABLE rm_campaigns (
    id                  UUID PRIMARY KEY,
    organization_id     UUID NOT NULL,
    name                VARCHAR(255),
    status              VARCHAR(30),
    campaign_type       VARCHAR(50),
    budget_total        NUMERIC(12,2),
    budget_spent        NUMERIC(12,2) NOT NULL DEFAULT 0,
    budget_currency     CHAR(3),
    start_date          DATE,
    end_date            DATE,
    creator_count       INTEGER NOT NULL DEFAULT 0,
    content_count       INTEGER NOT NULL DEFAULT 0,
    total_impressions   BIGINT NOT NULL DEFAULT 0,
    total_engagement    BIGINT NOT NULL DEFAULT 0,
    compliance_issues   INTEGER NOT NULL DEFAULT 0,
    projection_version  BIGINT NOT NULL DEFAULT 0
);

ALTER TABLE rm_campaigns ENABLE ROW LEVEL SECURITY;
CREATE POLICY rm_campaigns_tenant ON rm_campaigns
    USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE INDEX idx_rm_campaigns_org ON rm_campaigns(organization_id);
CREATE INDEX idx_rm_campaigns_status ON rm_campaigns(status);
```

### Compliance Audit Read Model

```sql
CREATE TABLE rm_compliance_audit (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_post_id     UUID NOT NULL,
    campaign_id         UUID NOT NULL,
    creator_id          UUID NOT NULL,
    organization_id     UUID NOT NULL,
    platform            VARCHAR(50),
    post_url            VARCHAR(500),
    check_type          VARCHAR(50),
    result              VARCHAR(20),
    details             TEXT,
    checked_by          VARCHAR(20),
    checked_at          TIMESTAMPTZ,
    resolved_at         TIMESTAMPTZ,
    resolution_notes    TEXT,
    projection_version  BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_rm_compliance_org ON rm_compliance_audit(organization_id);
CREATE INDEX idx_rm_compliance_result ON rm_compliance_audit(result);
CREATE INDEX idx_rm_compliance_campaign ON rm_compliance_audit(campaign_id);
```

### Payment Ledger Read Model

```sql
CREATE TABLE rm_payment_ledger (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    campaign_id         UUID NOT NULL,
    creator_id          UUID NOT NULL,
    payment_type        VARCHAR(30),
    amount              NUMERIC(12,2),
    currency            CHAR(3),
    status              VARCHAR(20),
    payment_method      VARCHAR(30),
    external_ref        VARCHAR(255),
    occurred_at         TIMESTAMPTZ,
    projection_version  BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_rm_payments_org ON rm_payment_ledger(organization_id);
CREATE INDEX idx_rm_payments_campaign ON rm_payment_ledger(campaign_id);
```

### Outreach Activity Read Model

```sql
CREATE TABLE rm_outreach_activity (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    campaign_id         UUID,
    creator_id          UUID NOT NULL,
    message_type        VARCHAR(20),
    status              VARCHAR(20),
    is_ai_generated     BOOLEAN,
    sent_at             TIMESTAMPTZ,
    opened_at           TIMESTAMPTZ,
    replied_at          TIMESTAMPTZ,
    projection_version  BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_rm_outreach_org ON rm_outreach_activity(organization_id);
CREATE INDEX idx_rm_outreach_creator ON rm_outreach_activity(creator_id);
```

---

## Projection Management

### Projection Checkpoints

```sql
CREATE TABLE projection_checkpoints (
    projection_name     VARCHAR(100) PRIMARY KEY,
    last_event_id       UUID,
    last_sequence        BIGINT NOT NULL DEFAULT 0,
    last_processed_at   TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'running',
        -- running, paused, rebuilding, error
    error_message       TEXT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example rows:
-- ('rm_creators', 'uuid', 154302, '2026-05-19T10:30:00Z', 'running', null)
-- ('rm_campaigns', 'uuid', 154301, '2026-05-19T10:30:00Z', 'running', null)
```

### Snapshot Store (for large aggregates)

```sql
CREATE TABLE snapshots (
    stream_id           UUID NOT NULL,
    stream_type         VARCHAR(50) NOT NULL,
    sequence_number     BIGINT NOT NULL,
    state               JSONB NOT NULL,          -- serialised aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, sequence_number)
);

CREATE INDEX idx_snapshots_stream ON snapshots(stream_id, sequence_number DESC);
```

---

## Example Queries

### Temporal Query: Creator fraud score on a specific date

```sql
-- Replay all creator_profile_synced events up to a target date
SELECT payload->>'fraud_score' AS fraud_score,
       occurred_at
FROM events
WHERE stream_id = '{{creator_id}}'
  AND event_type = 'creator_profile_synced'
  AND occurred_at <= '2026-01-15T23:59:59Z'
ORDER BY sequence_number DESC
LIMIT 1;
```

### GDPR Subject Access Request: All events for a creator

```sql
-- Return every event involving a specific creator
SELECT event_type, payload, occurred_at, metadata
FROM events
WHERE stream_id = '{{creator_id}}'
   OR payload->>'creator_id' = '{{creator_id}}'
ORDER BY occurred_at ASC;
```

### Compliance Audit Trail: All checks for a campaign

```sql
SELECT *
FROM rm_compliance_audit
WHERE campaign_id = '{{campaign_id}}'
ORDER BY checked_at ASC;
```

### Rebuild a read model from scratch

```sql
-- 1. Truncate the projection table
TRUNCATE rm_creators;

-- 2. Reset the checkpoint
UPDATE projection_checkpoints
SET last_sequence = 0, status = 'rebuilding'
WHERE projection_name = 'rm_creators';

-- 3. The projection worker replays all events from sequence 0
-- (application code handles this)
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Append-only, partitioned by month |
| Snapshots | 1 | Optional, for large aggregate streams |
| Projection Management | 1 | Checkpoint tracking per projection |
| Read Models — Creators | 2 | Creator summary + per-platform profiles |
| Read Models — Campaigns | 1 | Denormalized campaign dashboard |
| Read Models — Compliance | 1 | Flattened compliance audit trail |
| Read Models — Payments | 1 | Payment ledger view |
| Read Models — Outreach | 1 | Outreach activity summary |
| **Total** | **9** | Plus additional read models as needed |

---

## Key Design Decisions

1. **Single events table with JSONB payload** — all event types share one table, with `event_type` and `event_version` fields controlling deserialization. This simplifies the event store and avoids table-per-event-type explosion.

2. **Partition events by month** — PostgreSQL declarative partitioning keeps the active partition small for fast writes while older partitions can be moved to cheaper storage. A platform processing millions of events per month benefits significantly.

3. **Metadata captures actor and correlation** — every event records who triggered it (`metadata->>'actor_id'`), the originating request (`metadata->>'correlation_id'`), and the causing event (`metadata->>'causation_id'`). This enables full traceability for compliance audits.

4. **Projection checkpoints prevent duplicate processing** — each read model tracks the last processed event sequence number, enabling idempotent replay and crash recovery.

5. **Snapshots for performance** — aggregates with long event histories (e.g., a creator with thousands of profile sync events) use periodic snapshots to avoid replaying the entire stream.

6. **Read models are disposable** — any `rm_*` table can be dropped and rebuilt from events. This means new analytics dimensions can be added retroactively without data migration.

7. **Multi-tenancy on both layers** — the event store includes `organization_id` for efficient tenant-scoped queries. Read models inherit the same RLS policies as the normalized model.

8. **Event versioning enables schema evolution** — the `event_version` field allows the application to handle multiple versions of the same event type, applying upcasting functions to transform old events into the current schema.

9. **Compliance is an event, not a side effect** — every compliance check, consent grant, and deletion request is an event in the store. This means the compliance audit trail is as robust as the rest of the system — it cannot be accidentally deleted or modified.

10. **Financial events are immutable** — payment events are never updated; corrections are modeled as new events (e.g., `payment_refunded`, `payment_adjusted`). This matches double-entry accounting principles and provides a clean financial audit trail.
