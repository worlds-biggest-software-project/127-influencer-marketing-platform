# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Influencer Marketing Platform · Created: 2026-05-19

## Philosophy

This model uses a relational backbone for core entities and relationships but stores platform-specific, variable, and rapidly evolving data in PostgreSQL JSONB columns. The key insight is that influencer marketing data is inherently multi-platform — each social network (Instagram, TikTok, YouTube, LinkedIn, Substack) returns different metrics, audience structures, and content types. Rather than creating platform-specific columns or tables that require migrations whenever a platform changes its API, JSONB columns absorb the variability.

Core business logic fields (IDs, statuses, amounts, dates, foreign keys) remain fully relational with strong types and constraints. Variable fields (platform-specific metrics, audience breakdowns, AI scoring details, content metadata) live in typed JSONB columns with GIN indexes for efficient querying. This is the approach used by many modern SaaS platforms that must integrate with multiple third-party APIs whose schemas evolve independently.

The result is fewer tables than a fully normalized model, faster feature iteration (adding a new platform metric means adding a JSON key, not running ALTER TABLE), and the same transactional guarantees PostgreSQL provides for relational data.

**Best for:** Teams building an MVP that must support multiple social platforms quickly and adapt to frequent API changes without downtime migrations.

**Trade-offs:**
- Pro: Fewer tables (~15 vs ~21) — simpler schema, fewer joins
- Pro: New platform metrics require zero schema migrations — just add JSON keys
- Pro: GIN-indexed JSONB queries perform well for containment and existence checks
- Pro: Natural fit for storing heterogeneous social platform API responses
- Pro: Fastest path from design to working MVP
- Con: JSONB fields lack column-level constraints — validation must happen at application layer
- Con: JSONB queries are slower than native column queries for range scans and aggregations
- Con: Schema-on-read means bugs can introduce malformed data that is hard to detect
- Con: Reporting queries on JSONB fields are more verbose than column-based queries
- Con: No foreign key relationships within JSONB — referential integrity for nested data is application-managed

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IAB Creator Economy Taxonomy (April 2025) | Creator tiers and campaign types stored as relational enum-like fields for reliable querying |
| ISO 3166-1 alpha-2 | Country codes in relational columns and within JSONB audience geography arrays |
| ISO 4217 | Currency codes as relational columns alongside financial amounts |
| OAuth 2.0 (RFC 6749) | Platform credentials stored in JSONB within integration_configs |
| FTC 16 CFR Part 255 | Compliance results tracked relationally; check details in JSONB for flexibility |
| GDPR (EU) 2016/679 | Consent flags as relational booleans; consent details and provenance in JSONB |
| JSON Schema Draft 2020-12 | Application-layer validation of JSONB columns using JSON Schema definitions |

---

## Organization & Users

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "billing_email": "billing@brand.com",
    --   "industry": "beauty",
    --   "country_code": "US",
    --   "default_currency": "USD",
    --   "compliance_regions": ["us_ftc", "uk_asa", "eu_gdpr"],
    --   "integrations": {
    --     "shopify": {"store_url": "...", "enabled": true},
    --     "stripe": {"account_id": "acct_...", "enabled": true}
    --   },
    --   "branding": {"logo_url": "...", "primary_color": "#FF5722"}
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_slug ON organizations(slug);
CREATE INDEX idx_org_settings ON organizations USING GIN (settings);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- preferences example:
    -- {
    --   "notification_channels": ["email", "slack"],
    --   "default_view": "campaigns",
    --   "timezone": "America/New_York"
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY users_tenant ON users
    USING (organization_id = current_setting('app.current_org_id')::uuid);
```

---

## Creators

```sql
CREATE TABLE creators (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name           VARCHAR(255),
    primary_email       VARCHAR(255),
    country_code        CHAR(2),               -- ISO 3166-1
    tier                VARCHAR(20) NOT NULL DEFAULT 'unknown',
        -- IAB: nano, micro, mid, macro, mega
    fraud_score         NUMERIC(5,2),
    authenticity_score  NUMERIC(5,2),
    is_synthetic        BOOLEAN NOT NULL DEFAULT false,
    -- All platform profiles in one JSONB column
    profiles            JSONB NOT NULL DEFAULT '{}',
    -- profiles example:
    -- {
    --   "instagram": {
    --     "platform_user_id": "12345",
    --     "username": "janedoe",
    --     "profile_url": "https://instagram.com/janedoe",
    --     "follower_count": 45000,
    --     "following_count": 1200,
    --     "post_count": 340,
    --     "avg_engagement_rate": 3.52,
    --     "avg_likes": 1580,
    --     "avg_comments": 95,
    --     "avg_views": null,
    --     "growth_rate_30d": 2.1,
    --     "is_verified": true,
    --     "last_synced_at": "2026-05-19T10:00:00Z"
    --   },
    --   "tiktok": {
    --     "platform_user_id": "67890",
    --     "username": "janedoe_tt",
    --     "follower_count": 120000,
    --     "avg_engagement_rate": 8.7,
    --     "avg_views": 25000,
    --     "growth_rate_30d": 5.3,
    --     "last_synced_at": "2026-05-19T09:30:00Z"
    --   }
    -- }
    audience            JSONB NOT NULL DEFAULT '{}',
    -- audience example:
    -- {
    --   "instagram": {
    --     "snapshot_date": "2026-05-19",
    --     "age": {"13-17": 2.1, "18-24": 35.2, "25-34": 42.1, "35-44": 12.3, "45-54": 5.8, "55+": 2.5},
    --     "gender": {"male": 32.0, "female": 66.5, "other": 1.5},
    --     "countries": [{"code": "US", "pct": 45.0}, {"code": "GB", "pct": 12.3}, {"code": "CA", "pct": 8.1}],
    --     "interests": ["beauty", "fashion", "lifestyle"],
    --     "languages": [{"code": "en", "pct": 78.0}, {"code": "es", "pct": 12.0}]
    --   },
    --   "tiktok": {
    --     "snapshot_date": "2026-05-19",
    --     "age": {"13-17": 15.0, "18-24": 45.0, "25-34": 30.0, "35+": 10.0},
    --     "gender": {"male": 40.0, "female": 58.0, "other": 2.0},
    --     "countries": [{"code": "US", "pct": 55.0}, {"code": "MX", "pct": 10.0}]
    --   }
    -- }
    ai_scores           JSONB NOT NULL DEFAULT '{}',
    -- ai_scores example:
    -- {
    --   "brand_safety": 92.5,
    --   "content_quality": 78.3,
    --   "audience_purchasing_intent": 65.0,
    --   "fatigue_risk": 15.0,
    --   "niche_authority": {"beauty": 88.0, "fashion": 72.0},
    --   "sentiment": {"positive": 78.0, "neutral": 18.0, "negative": 4.0},
    --   "scored_at": "2026-05-19T08:00:00Z"
    -- }
    contact_info        JSONB NOT NULL DEFAULT '{}',
    -- {"phone": "+1...", "secondary_email": "...", "agent_name": "...", "agent_email": "..."}
    categories          VARCHAR(100)[] NOT NULL DEFAULT '{}',
    -- PostgreSQL array: {"beauty", "fashion", "lifestyle"}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_creators_tier ON creators(tier);
CREATE INDEX idx_creators_country ON creators(country_code);
CREATE INDEX idx_creators_fraud ON creators(fraud_score);
CREATE INDEX idx_creators_categories ON creators USING GIN (categories);
CREATE INDEX idx_creators_profiles ON creators USING GIN (profiles);
CREATE INDEX idx_creators_audience ON creators USING GIN (audience);
CREATE INDEX idx_creators_ai_scores ON creators USING GIN (ai_scores);
```

### Example JSONB Queries on Creators

```sql
-- Find creators with Instagram engagement rate > 3.0
SELECT id, full_name,
       (profiles->'instagram'->>'avg_engagement_rate')::numeric AS ig_engagement
FROM creators
WHERE (profiles->'instagram'->>'avg_engagement_rate')::numeric > 3.0;

-- Find creators whose Instagram audience is >40% female aged 18-24
SELECT id, full_name
FROM creators
WHERE (audience->'instagram'->'age'->>'18-24')::numeric > 40.0
  AND (audience->'instagram'->'gender'->>'female')::numeric > 40.0;

-- Find creators in 'beauty' category with fraud score under 20
SELECT id, full_name, fraud_score
FROM creators
WHERE 'beauty' = ANY(categories)
  AND fraud_score < 20.0;

-- Find creators who have a TikTok profile
SELECT id, full_name
FROM creators
WHERE profiles ? 'tiktok';
```

---

## Campaigns

```sql
CREATE TABLE campaigns (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    campaign_type       VARCHAR(50) NOT NULL,       -- IAB taxonomy
    objective           VARCHAR(50),
    budget_total        NUMERIC(12,2),
    budget_spent        NUMERIC(12,2) NOT NULL DEFAULT 0,
    budget_currency     CHAR(3) NOT NULL DEFAULT 'USD',
    start_date          DATE,
    end_date            DATE,
    target_platforms    VARCHAR(50)[] NOT NULL DEFAULT '{}',
    brief               JSONB NOT NULL DEFAULT '{}',
    -- brief example:
    -- {
    --   "title": "Summer Beauty Launch",
    --   "content": "## Campaign Brief\n...",
    --   "deliverables": ["1 Instagram Reel", "2 TikTok videos"],
    --   "hashtags_required": ["#ad", "#SummerGlow"],
    --   "disclosure_text": "Paid partnership with BrandX",
    --   "dos": ["Show product in natural light", "Tag @brandx"],
    --   "donts": ["No competitor products visible", "No alcohol"],
    --   "version": 3
    -- }
    metrics             JSONB NOT NULL DEFAULT '{}',
    -- metrics example (denormalized rollup, updated periodically):
    -- {
    --   "total_impressions": 1250000,
    --   "total_reach": 890000,
    --   "total_engagement": 45000,
    --   "total_clicks": 12000,
    --   "total_conversions": 340,
    --   "total_revenue": 15200.00,
    --   "roi": 3.04,
    --   "by_platform": {
    --     "instagram": {"impressions": 750000, "engagement": 28000},
    --     "tiktok": {"impressions": 500000, "engagement": 17000}
    --   },
    --   "last_calculated_at": "2026-05-19T12:00:00Z"
    -- }
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE campaigns ENABLE ROW LEVEL SECURITY;
CREATE POLICY campaigns_tenant ON campaigns
    USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE INDEX idx_campaigns_org ON campaigns(organization_id);
CREATE INDEX idx_campaigns_status ON campaigns(status);
CREATE INDEX idx_campaigns_dates ON campaigns(start_date, end_date);
CREATE INDEX idx_campaigns_metrics ON campaigns USING GIN (metrics);
```

---

## Campaign Creators

```sql
CREATE TABLE campaign_creators (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    creator_id          UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    status              VARCHAR(30) NOT NULL DEFAULT 'invited',
    -- Relational fields for core financial data
    agreed_rate         NUMERIC(10,2),
    rate_currency       CHAR(3) DEFAULT 'USD',
    rate_type           VARCHAR(30),
    commission_pct      NUMERIC(5,2),
    -- Flexible fields for negotiation details, contract terms, etc.
    details             JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "brief_accepted_at": "2026-05-10T14:00:00Z",
    --   "contract_signed_at": "2026-05-11T09:00:00Z",
    --   "deliverables": [
    --     {"type": "instagram_reel", "count": 1, "due_date": "2026-06-01"},
    --     {"type": "tiktok_video", "count": 2, "due_date": "2026-06-15"}
    --   ],
    --   "notes": "Creator prefers to ship product to NYC address",
    --   "shipping_address": {"line1": "...", "city": "New York", "state": "NY", "zip": "10001"},
    --   "ugc_license": {"granted": true, "term_months": 12, "channels": ["paid_social", "website"]}
    -- }
    performance         JSONB NOT NULL DEFAULT '{}',
    -- performance example:
    -- {
    --   "total_impressions": 85000,
    --   "total_engagement": 3200,
    --   "conversions": 45,
    --   "revenue_generated": 2250.00,
    --   "affiliate_clicks": 890,
    --   "content_count": 3
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(campaign_id, creator_id)
);

CREATE INDEX idx_cc_campaign ON campaign_creators(campaign_id);
CREATE INDEX idx_cc_creator ON campaign_creators(creator_id);
CREATE INDEX idx_cc_status ON campaign_creators(status);
```

---

## Content Posts

```sql
CREATE TABLE content_posts (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_creator_id     UUID NOT NULL REFERENCES campaign_creators(id) ON DELETE CASCADE,
    platform                VARCHAR(50) NOT NULL,
    platform_post_id        VARCHAR(255),
    post_url                VARCHAR(500),
    post_type               VARCHAR(30) NOT NULL,
    published_at            TIMESTAMPTZ,
    -- Platform-specific metrics in JSONB (different platforms return different fields)
    metrics                 JSONB NOT NULL DEFAULT '{}',
    -- metrics example (Instagram):
    -- {
    --   "impressions": 42000,
    --   "reach": 31000,
    --   "likes": 1580,
    --   "comments": 95,
    --   "shares": 210,
    --   "saves": 340,
    --   "engagement_rate": 5.32,
    --   "video_views": 28000,
    --   "avg_watch_time_sec": 12.5,
    --   "profile_visits": 450,
    --   "website_clicks": 120,
    --   "last_synced_at": "2026-05-19T14:00:00Z"
    -- }
    -- metrics example (TikTok):
    -- {
    --   "views": 150000,
    --   "likes": 8500,
    --   "comments": 320,
    --   "shares": 1200,
    --   "engagement_rate": 6.68,
    --   "avg_watch_time_sec": 8.2,
    --   "completion_rate": 0.45,
    --   "last_synced_at": "2026-05-19T14:00:00Z"
    -- }
    compliance              JSONB NOT NULL DEFAULT '{}',
    -- compliance example:
    -- {
    --   "has_disclosure": true,
    --   "disclosure_types": ["hashtag_ad", "platform_tag"],
    --   "status": "compliant",
    --   "checks": [
    --     {"type": "ftc_disclosure", "result": "pass", "checked_by": "ai", "at": "2026-05-16T10:00:00Z"},
    --     {"type": "brand_safety", "result": "pass", "checked_by": "ai", "at": "2026-05-16T10:00:00Z"}
    --   ],
    --   "last_checked_at": "2026-05-16T10:00:00Z"
    -- }
    caption                 TEXT,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_cc ON content_posts(campaign_creator_id);
CREATE INDEX idx_content_platform ON content_posts(platform);
CREATE INDEX idx_content_published ON content_posts(published_at);
CREATE INDEX idx_content_metrics ON content_posts USING GIN (metrics);
CREATE INDEX idx_content_compliance ON content_posts USING GIN (compliance);
```

---

## Outreach

```sql
CREATE TABLE outreach_messages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    campaign_id         UUID REFERENCES campaigns(id),
    creator_id          UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    message_type        VARCHAR(20) NOT NULL,     -- email, dm
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    subject             VARCHAR(500),
    body                TEXT NOT NULL,
    is_ai_generated     BOOLEAN NOT NULL DEFAULT false,
    ai_context          JSONB NOT NULL DEFAULT '{}',
    -- ai_context example:
    -- {
    --   "model": "claude-opus-4-6",
    --   "referenced_posts": ["https://instagram.com/p/abc", "https://instagram.com/p/def"],
    --   "brand_fit_score": 87.5,
    --   "personalization_signals": ["recent_skincare_review", "uses_brand_competitor"]
    -- }
    tracking             JSONB NOT NULL DEFAULT '{}',
    -- tracking example:
    -- {
    --   "scheduled_at": "2026-05-20T09:00:00Z",
    --   "sent_at": "2026-05-20T09:01:23Z",
    --   "delivered_at": "2026-05-20T09:01:25Z",
    --   "opened_at": "2026-05-20T10:15:00Z",
    --   "opened_count": 3,
    --   "replied_at": "2026-05-20T14:30:00Z",
    --   "bounced": false
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE outreach_messages ENABLE ROW LEVEL SECURITY;
CREATE POLICY outreach_tenant ON outreach_messages
    USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE INDEX idx_outreach_org ON outreach_messages(organization_id);
CREATE INDEX idx_outreach_creator ON outreach_messages(creator_id);
CREATE INDEX idx_outreach_status ON outreach_messages(status);
```

---

## Payments & Affiliates

```sql
CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    campaign_creator_id UUID NOT NULL REFERENCES campaign_creators(id) ON DELETE CASCADE,
    amount              NUMERIC(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL DEFAULT 'USD',
    payment_type        VARCHAR(30) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    details             JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "payment_method": "stripe",
    --   "external_payment_id": "pi_abc123",
    --   "invoice_url": "https://...",
    --   "stripe_transfer_id": "tr_xyz",
    --   "tax_form_status": "w9_on_file",
    --   "paid_at": "2026-05-18T16:00:00Z"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
CREATE POLICY payments_tenant ON payments
    USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE INDEX idx_payments_org ON payments(organization_id);
CREATE INDEX idx_payments_cc ON payments(campaign_creator_id);
CREATE INDEX idx_payments_status ON payments(status);

CREATE TABLE affiliate_links (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_creator_id UUID NOT NULL REFERENCES campaign_creators(id) ON DELETE CASCADE,
    link_type           VARCHAR(20) NOT NULL,
    tracking_url        VARCHAR(500),
    discount_code       VARCHAR(100),
    destination_url     VARCHAR(500),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    expires_at          TIMESTAMPTZ,
    stats               JSONB NOT NULL DEFAULT '{}',
    -- stats example (updated periodically):
    -- {
    --   "clicks": 890,
    --   "conversions": 45,
    --   "revenue": 2250.00,
    --   "orders": [
    --     {"order_id": "shopify_123", "amount": 89.99, "at": "2026-05-15T10:00:00Z"},
    --     {"order_id": "shopify_124", "amount": 129.99, "at": "2026-05-16T14:00:00Z"}
    --   ],
    --   "last_updated_at": "2026-05-19T12:00:00Z"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_affiliate_cc ON affiliate_links(campaign_creator_id);
CREATE INDEX idx_affiliate_code ON affiliate_links(discount_code);
```

---

## Compliance & Consent

```sql
CREATE TABLE compliance_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    entity_type         VARCHAR(30) NOT NULL,      -- content_post, creator, campaign
    entity_id           UUID NOT NULL,
    event_type          VARCHAR(50) NOT NULL,
        -- disclosure_check, consent_granted, consent_revoked,
        -- deletion_requested, deletion_completed, brand_safety_flag
    severity            VARCHAR(20) NOT NULL DEFAULT 'info',
        -- info, warning, violation
    details             JSONB NOT NULL DEFAULT '{}',
    -- details example:
    -- {
    --   "check_type": "ftc_disclosure",
    --   "result": "fail",
    --   "explanation": "Caption missing #ad hashtag",
    --   "post_url": "https://instagram.com/p/abc",
    --   "checked_by": "ai",
    --   "regulation": "ftc_16cfr255",
    --   "recommended_action": "Contact creator to add disclosure"
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_org ON compliance_log(organization_id);
CREATE INDEX idx_compliance_entity ON compliance_log(entity_type, entity_id);
CREATE INDEX idx_compliance_event ON compliance_log(event_type);
CREATE INDEX idx_compliance_severity ON compliance_log(severity);
CREATE INDEX idx_compliance_details ON compliance_log USING GIN (details);
```

---

## Platform Integrations

```sql
CREATE TABLE integration_configs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    platform            VARCHAR(50) NOT NULL,
    is_enabled          BOOLEAN NOT NULL DEFAULT true,
    config              JSONB NOT NULL DEFAULT '{}',
    -- config example (Shopify):
    -- {
    --   "store_url": "mystore.myshopify.com",
    --   "api_version": "2026-04",
    --   "scopes": ["read_orders", "read_products"],
    --   "webhook_subscriptions": ["orders/create", "orders/paid"]
    -- }
    credentials         JSONB NOT NULL DEFAULT '{}',
    -- credentials example (encrypted at application layer):
    -- {
    --   "access_token": "encrypted:...",
    --   "refresh_token": "encrypted:...",
    --   "expires_at": "2026-06-19T10:00:00Z",
    --   "token_type": "bearer"
    -- }
    last_synced_at      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, platform)
);

ALTER TABLE integration_configs ENABLE ROW LEVEL SECURITY;
CREATE POLICY integrations_tenant ON integration_configs
    USING (organization_id = current_setting('app.current_org_id')::uuid);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organization & Users | 2 | Settings and preferences in JSONB |
| Creators | 1 | Profiles, audience, AI scores all in JSONB columns |
| Campaigns | 1 | Brief and rolled-up metrics in JSONB |
| Campaign Creators | 1 | Contract details and performance in JSONB |
| Content Posts | 1 | Platform-specific metrics and compliance in JSONB |
| Outreach | 1 | AI context and tracking in JSONB |
| Payments & Affiliates | 2 | Payment details and affiliate stats in JSONB |
| Compliance | 1 | Unified log with JSONB details |
| Integrations | 1 | Per-platform config and credentials in JSONB |
| **Total** | **11** | Significantly fewer tables than normalized model |

---

## Key Design Decisions

1. **Platform profiles as JSONB, not separate rows** — a creator's Instagram, TikTok, and YouTube profiles are stored as keys within a single `profiles` JSONB column rather than separate rows. This eliminates the join needed to display a creator's full profile and allows platform-specific fields without schema changes.

2. **Audience demographics are per-platform within JSONB** — each platform returns different demographic categories and granularities. JSONB absorbs this naturally. The trade-off is that cross-platform audience analysis requires JSONB extraction in queries.

3. **Campaign briefs embedded in campaigns** — rather than a separate `campaign_briefs` table, the brief lives as JSONB within the campaign. This simplifies the common "load campaign with its brief" query to a single row fetch.

4. **Compliance as a unified log** — instead of separate tables for FTC checks, consent records, and deletion requests, a single `compliance_log` table with JSONB details handles all compliance event types. The `event_type` and `severity` columns enable efficient filtering.

5. **Content metrics are platform-specific JSONB** — Instagram returns impressions, reach, saves; TikTok returns views, completion_rate. Rather than a union of all possible columns, each platform's metric set is stored as-is from the API response.

6. **AI scoring details in JSONB** — the `ai_scores` column on creators stores the full AI analysis output (brand safety, content quality, fatigue risk, niche authority). As AI models evolve and new scores are added, no migration is needed.

7. **GIN indexes on all major JSONB columns** — enables efficient containment queries (`@>`), existence checks (`?`), and path queries. For example, `WHERE profiles ? 'tiktok'` to find all creators with TikTok presence.

8. **Financial amounts remain relational** — despite the JSONB-heavy approach, payment amounts, currencies, and campaign budgets are relational columns with proper NUMERIC types. Financial data must not rely on schema-on-read validation.

9. **Application-layer JSON Schema validation** — every JSONB column has a corresponding JSON Schema definition in the application layer (using JSON Schema Draft 2020-12) that validates structure on write. This compensates for the lack of column-level database constraints.

10. **Integration credentials in JSONB** — each platform's OAuth token structure is different (some use refresh tokens, some use API keys, some have additional fields). JSONB accommodates all variants in a single table, with application-layer encryption for sensitive fields.
