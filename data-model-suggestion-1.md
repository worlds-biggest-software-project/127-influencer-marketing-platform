# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Influencer Marketing Platform · Created: 2026-05-19

## Philosophy

This model follows classical third-normal-form (3NF) relational design. Every concept gets its own table with strong foreign key constraints, ensuring referential integrity across the entire system. Junction tables handle many-to-many relationships (creators-to-campaigns, campaigns-to-platforms, creators-to-categories). Reference data (countries, platforms, content types, compliance rules) lives in dedicated lookup tables aligned with industry standards.

This is the approach used by mature enterprise CRM systems and is well-understood by database administrators. It provides the strongest data integrity guarantees and the most predictable query performance through well-defined indexes. PostgreSQL row-level security (RLS) enforces multi-tenant isolation at the database layer, meaning application bugs cannot leak data across tenants.

The trade-off is a higher table count and more complex joins for cross-entity queries. However, for a platform where data correctness (payments, compliance, audience metrics) is critical, the integrity guarantees outweigh the verbosity.

**Best for:** Teams prioritizing data integrity, regulatory compliance, and long-term maintainability over rapid prototyping speed.

**Trade-offs:**
- Pro: Strongest referential integrity — no orphaned records, no inconsistent states
- Pro: Well-understood by any PostgreSQL developer; no exotic patterns to learn
- Pro: RLS-based multi-tenancy is battle-tested for SaaS
- Pro: Clean migration path — ALTER TABLE for schema changes
- Con: High table count (~45-55 tables) increases join complexity
- Con: Adding platform-specific fields requires schema migrations
- Con: Cross-entity analytics queries require multiple joins
- Con: Less flexible for rapidly evolving social platform data structures

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IAB Creator Economy Taxonomy (April 2025) | Creator tier classifications (nano, micro, mid, macro, mega) and campaign type definitions align with IAB vocabulary |
| ISO 3166-1 alpha-2 | Country codes for creator locations, audience geography, and jurisdiction-specific compliance rules |
| ISO 4217 | Currency codes for payment amounts, campaign budgets, and creator rate cards |
| OAuth 2.0 (RFC 6749) | Token storage schema for social platform API credentials |
| FTC 16 CFR Part 255 | Compliance check results and disclosure verification records stored in dedicated audit tables |
| GDPR (EU) 2016/679 | Consent records, data processing purposes, and deletion request tracking in dedicated tables |
| OpenAPI 3.1 / JSON Schema | API response validation schemas referenced in platform_api_configs |

---

## Entity Management

### Organizations (Tenants)

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, starter, pro, enterprise
    billing_email   VARCHAR(255),
    website_url     VARCHAR(500),
    industry        VARCHAR(100),
    country_code    CHAR(2),           -- ISO 3166-1 alpha-2
    gdpr_compliant  BOOLEAN NOT NULL DEFAULT false,
    ccpa_compliant  BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organizations_slug ON organizations(slug);
```

### Users & Roles

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member', -- owner, admin, manager, member, viewer
    avatar_url      VARCHAR(500),
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);

CREATE INDEX idx_users_org ON users(organization_id);
CREATE INDEX idx_users_email ON users(email);

-- Row-level security for multi-tenancy
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY users_tenant_isolation ON users
    USING (organization_id = current_setting('app.current_org_id')::uuid);
```

---

## Creator Management

### Creators

```sql
CREATE TABLE creators (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    full_name           VARCHAR(255),
    bio                 TEXT,
    primary_email       VARCHAR(255),
    secondary_email     VARCHAR(255),
    phone               VARCHAR(50),
    country_code        CHAR(2),               -- ISO 3166-1 alpha-2
    city                VARCHAR(100),
    primary_language    VARCHAR(10),            -- BCP 47 language tag
    tier                VARCHAR(20) NOT NULL DEFAULT 'unknown',
        -- IAB taxonomy: nano (1K-10K), micro (10K-50K), mid (50K-500K),
        -- macro (500K-1M), mega (1M+)
    is_verified         BOOLEAN NOT NULL DEFAULT false,
    fraud_score         NUMERIC(5,2),           -- 0.00 to 100.00
    authenticity_score  NUMERIC(5,2),           -- 0.00 to 100.00
    is_synthetic        BOOLEAN NOT NULL DEFAULT false,  -- AI-generated creator flag
    avatar_url          VARCHAR(500),
    website_url         VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_creators_tier ON creators(tier);
CREATE INDEX idx_creators_country ON creators(country_code);
CREATE INDEX idx_creators_fraud_score ON creators(fraud_score);
```

### Creator Social Profiles

```sql
CREATE TABLE creator_social_profiles (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id          UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    platform            VARCHAR(50) NOT NULL,  -- instagram, youtube, tiktok, linkedin, x, substack
    platform_user_id    VARCHAR(255),          -- platform-native ID
    username            VARCHAR(255) NOT NULL,
    profile_url         VARCHAR(500),
    follower_count      BIGINT,
    following_count     BIGINT,
    post_count          BIGINT,
    avg_engagement_rate NUMERIC(7,4),          -- percentage, e.g. 3.5200
    avg_views           BIGINT,
    avg_likes           BIGINT,
    avg_comments        BIGINT,
    growth_rate_30d     NUMERIC(7,4),          -- percentage change over 30 days
    is_verified         BOOLEAN NOT NULL DEFAULT false,
    last_synced_at      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(creator_id, platform)
);

CREATE INDEX idx_creator_profiles_platform ON creator_social_profiles(platform);
CREATE INDEX idx_creator_profiles_followers ON creator_social_profiles(follower_count);
CREATE INDEX idx_creator_profiles_engagement ON creator_social_profiles(avg_engagement_rate);
```

### Creator Audience Demographics

```sql
CREATE TABLE creator_audience_demographics (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_profile_id  UUID NOT NULL REFERENCES creator_social_profiles(id) ON DELETE CASCADE,
    snapshot_date       DATE NOT NULL,
    -- Age distribution (percentages summing to ~100)
    age_13_17_pct       NUMERIC(5,2),
    age_18_24_pct       NUMERIC(5,2),
    age_25_34_pct       NUMERIC(5,2),
    age_35_44_pct       NUMERIC(5,2),
    age_45_54_pct       NUMERIC(5,2),
    age_55_plus_pct     NUMERIC(5,2),
    -- Gender distribution
    gender_male_pct     NUMERIC(5,2),
    gender_female_pct   NUMERIC(5,2),
    gender_other_pct    NUMERIC(5,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(creator_profile_id, snapshot_date)
);

CREATE INDEX idx_audience_demo_profile ON creator_audience_demographics(creator_profile_id);
```

### Creator Audience Geography

```sql
CREATE TABLE creator_audience_countries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_profile_id  UUID NOT NULL REFERENCES creator_social_profiles(id) ON DELETE CASCADE,
    snapshot_date       DATE NOT NULL,
    country_code        CHAR(2) NOT NULL,      -- ISO 3166-1 alpha-2
    percentage          NUMERIC(5,2) NOT NULL,  -- share of total audience
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(creator_profile_id, snapshot_date, country_code)
);

CREATE INDEX idx_audience_geo ON creator_audience_countries(creator_profile_id, snapshot_date);
```

### Creator Categories

```sql
CREATE TABLE categories (
    id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name    VARCHAR(100) NOT NULL UNIQUE,   -- e.g. beauty, fashion, tech, fitness
    parent_id UUID REFERENCES categories(id)  -- hierarchical categories
);

CREATE TABLE creator_categories (
    creator_id  UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    category_id UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    relevance   NUMERIC(5,2),  -- AI-scored relevance 0-100
    PRIMARY KEY (creator_id, category_id)
);
```

---

## Campaign Management

### Campaigns

```sql
CREATE TABLE campaigns (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name                VARCHAR(255) NOT NULL,
    description         TEXT,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
        -- draft, active, paused, completed, archived
    campaign_type       VARCHAR(50) NOT NULL,
        -- IAB types: sponsored_post, product_seeding, affiliate, ambassador, ugc
    objective           VARCHAR(50),   -- awareness, engagement, conversion, traffic
    budget_total        NUMERIC(12,2),
    budget_spent        NUMERIC(12,2) NOT NULL DEFAULT 0,
    budget_currency     CHAR(3) NOT NULL DEFAULT 'USD',   -- ISO 4217
    start_date          DATE,
    end_date            DATE,
    target_platforms     VARCHAR(50)[] NOT NULL DEFAULT '{}',  -- array of platform names
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
```

### Campaign Creators (Junction)

```sql
CREATE TABLE campaign_creators (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    creator_id          UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    status              VARCHAR(30) NOT NULL DEFAULT 'invited',
        -- invited, negotiating, contracted, active, completed, declined, removed
    agreed_rate         NUMERIC(10,2),
    rate_currency       CHAR(3) DEFAULT 'USD',     -- ISO 4217
    rate_type           VARCHAR(30),                -- flat_fee, per_post, cpm, commission_pct
    commission_pct      NUMERIC(5,2),               -- for affiliate campaigns
    brief_accepted_at   TIMESTAMPTZ,
    contract_signed_at  TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(campaign_id, creator_id)
);

CREATE INDEX idx_campaign_creators_campaign ON campaign_creators(campaign_id);
CREATE INDEX idx_campaign_creators_creator ON campaign_creators(creator_id);
CREATE INDEX idx_campaign_creators_status ON campaign_creators(status);
```

### Campaign Briefs

```sql
CREATE TABLE campaign_briefs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    title               VARCHAR(255) NOT NULL,
    content             TEXT NOT NULL,           -- markdown brief content
    deliverables        TEXT,                    -- expected deliverables description
    dos_and_donts       TEXT,                    -- brand guidelines
    hashtags_required   VARCHAR(255)[],          -- required hashtags
    disclosure_text     VARCHAR(255),            -- required FTC disclosure text
    version             INTEGER NOT NULL DEFAULT 1,
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_briefs_campaign ON campaign_briefs(campaign_id);
```

---

## Content Tracking

### Content Posts

```sql
CREATE TABLE content_posts (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_creator_id     UUID NOT NULL REFERENCES campaign_creators(id) ON DELETE CASCADE,
    platform                VARCHAR(50) NOT NULL,
    platform_post_id        VARCHAR(255),          -- native post ID
    post_url                VARCHAR(500),
    post_type               VARCHAR(30) NOT NULL,  -- image, video, reel, story, carousel, article
    caption                 TEXT,
    published_at            TIMESTAMPTZ,
    -- Metrics (updated periodically)
    impressions             BIGINT DEFAULT 0,
    reach                   BIGINT DEFAULT 0,
    likes                   BIGINT DEFAULT 0,
    comments                BIGINT DEFAULT 0,
    shares                  BIGINT DEFAULT 0,
    saves                   BIGINT DEFAULT 0,
    video_views             BIGINT DEFAULT 0,
    engagement_rate         NUMERIC(7,4),
    -- Compliance
    has_disclosure           BOOLEAN,
    disclosure_type          VARCHAR(50),           -- hashtag, platform_tag, caption_text
    compliance_status        VARCHAR(30) DEFAULT 'pending',
        -- pending, compliant, non_compliant, review_required
    compliance_checked_at    TIMESTAMPTZ,
    metrics_last_synced_at   TIMESTAMPTZ,
    created_at               TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at               TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_campaign_creator ON content_posts(campaign_creator_id);
CREATE INDEX idx_content_platform ON content_posts(platform);
CREATE INDEX idx_content_compliance ON content_posts(compliance_status);
CREATE INDEX idx_content_published ON content_posts(published_at);
```

---

## Outreach & CRM

### Outreach Sequences

```sql
CREATE TABLE outreach_sequences (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    campaign_id         UUID REFERENCES campaigns(id),
    name                VARCHAR(255) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft', -- draft, active, paused, completed
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE outreach_sequences ENABLE ROW LEVEL SECURITY;
CREATE POLICY outreach_sequences_tenant ON outreach_sequences
    USING (organization_id = current_setting('app.current_org_id')::uuid);
```

### Outreach Messages

```sql
CREATE TABLE outreach_messages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sequence_id         UUID NOT NULL REFERENCES outreach_sequences(id) ON DELETE CASCADE,
    creator_id          UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    message_type        VARCHAR(20) NOT NULL,     -- email, dm
    subject             VARCHAR(500),
    body                TEXT NOT NULL,
    is_ai_generated     BOOLEAN NOT NULL DEFAULT false,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
        -- draft, scheduled, sent, delivered, opened, replied, bounced
    scheduled_at        TIMESTAMPTZ,
    sent_at             TIMESTAMPTZ,
    opened_at           TIMESTAMPTZ,
    replied_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outreach_msgs_sequence ON outreach_messages(sequence_id);
CREATE INDEX idx_outreach_msgs_creator ON outreach_messages(creator_id);
CREATE INDEX idx_outreach_msgs_status ON outreach_messages(status);
```

---

## Payments & Affiliate Tracking

### Payments

```sql
CREATE TABLE payments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    campaign_creator_id UUID NOT NULL REFERENCES campaign_creators(id) ON DELETE CASCADE,
    amount              NUMERIC(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL DEFAULT 'USD',   -- ISO 4217
    payment_type        VARCHAR(30) NOT NULL,              -- flat_fee, commission, bonus
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, approved, processing, completed, failed, refunded
    payment_method      VARCHAR(30),                       -- stripe, paypal, bank_transfer
    external_payment_id VARCHAR(255),                      -- Stripe payment intent ID etc.
    invoice_url         VARCHAR(500),
    paid_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
CREATE POLICY payments_tenant ON payments
    USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE INDEX idx_payments_org ON payments(organization_id);
CREATE INDEX idx_payments_campaign_creator ON payments(campaign_creator_id);
CREATE INDEX idx_payments_status ON payments(status);
```

### Affiliate Links & Codes

```sql
CREATE TABLE affiliate_links (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_creator_id UUID NOT NULL REFERENCES campaign_creators(id) ON DELETE CASCADE,
    link_type           VARCHAR(20) NOT NULL,     -- url, discount_code
    tracking_url        VARCHAR(500),
    discount_code       VARCHAR(100),
    destination_url     VARCHAR(500),
    clicks              BIGINT NOT NULL DEFAULT 0,
    conversions         BIGINT NOT NULL DEFAULT 0,
    revenue_generated   NUMERIC(12,2) NOT NULL DEFAULT 0,
    revenue_currency    CHAR(3) NOT NULL DEFAULT 'USD',
    is_active           BOOLEAN NOT NULL DEFAULT true,
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_affiliate_campaign_creator ON affiliate_links(campaign_creator_id);
CREATE INDEX idx_affiliate_discount_code ON affiliate_links(discount_code);
```

---

## Compliance & Audit

### Compliance Checks

```sql
CREATE TABLE compliance_checks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    content_post_id     UUID NOT NULL REFERENCES content_posts(id) ON DELETE CASCADE,
    check_type          VARCHAR(50) NOT NULL,
        -- ftc_disclosure, asa_disclosure, platform_tag, brand_safety, synthetic_creator
    result              VARCHAR(20) NOT NULL,      -- pass, fail, warning, manual_review
    details             TEXT,                       -- human-readable explanation
    rule_version        VARCHAR(20),               -- version of compliance rule applied
    checked_by          VARCHAR(20) NOT NULL,      -- system, ai, human
    reviewer_user_id    UUID REFERENCES users(id), -- if checked by human
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_post ON compliance_checks(content_post_id);
CREATE INDEX idx_compliance_result ON compliance_checks(result);
CREATE INDEX idx_compliance_type ON compliance_checks(check_type);
```

### GDPR Consent Records

```sql
CREATE TABLE consent_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id          UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    consent_type        VARCHAR(50) NOT NULL,
        -- data_processing, marketing_contact, audience_analysis, data_sharing
    granted             BOOLEAN NOT NULL,
    ip_address          INET,
    user_agent          TEXT,
    granted_at          TIMESTAMPTZ,
    revoked_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_consent_creator ON consent_records(creator_id);
CREATE INDEX idx_consent_type ON consent_records(consent_type);
```

### Data Deletion Requests (GDPR/CCPA)

```sql
CREATE TABLE data_deletion_requests (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    creator_id          UUID REFERENCES creators(id),
    requester_email     VARCHAR(255) NOT NULL,
    regulation          VARCHAR(10) NOT NULL,      -- gdpr, ccpa
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
        -- pending, in_progress, completed, rejected
    requested_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at        TIMESTAMPTZ,
    completed_by        UUID REFERENCES users(id),
    notes               TEXT
);

CREATE INDEX idx_deletion_status ON data_deletion_requests(status);
```

---

## Platform Integration

### OAuth Tokens

```sql
CREATE TABLE oauth_tokens (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    platform            VARCHAR(50) NOT NULL,    -- instagram, youtube, tiktok, shopify, stripe
    token_type          VARCHAR(20) NOT NULL,    -- access, refresh
    encrypted_token     TEXT NOT NULL,            -- encrypted at application layer
    scopes              VARCHAR(500)[],
    expires_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_oauth_org_platform ON oauth_tokens(organization_id, platform);
```

### E-commerce Conversions

```sql
CREATE TABLE ecommerce_conversions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    affiliate_link_id   UUID NOT NULL REFERENCES affiliate_links(id) ON DELETE CASCADE,
    order_id            VARCHAR(255),             -- external order ID (Shopify, etc.)
    order_amount        NUMERIC(12,2) NOT NULL,
    order_currency      CHAR(3) NOT NULL DEFAULT 'USD',
    commission_amount   NUMERIC(12,2),
    source_platform     VARCHAR(50),              -- shopify, woocommerce, magento, amazon
    converted_at        TIMESTAMPTZ NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ecom_affiliate ON ecommerce_conversions(affiliate_link_id);
CREATE INDEX idx_ecom_converted ON ecommerce_conversions(converted_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Organization & Users | 2 | Multi-tenant with RLS |
| Creator Management | 5 | Profiles, demographics, geography, categories |
| Campaign Management | 3 | Campaigns, creator assignments, briefs |
| Content Tracking | 1 | Posts with metrics and compliance fields |
| Outreach & CRM | 2 | Sequences and messages |
| Payments & Affiliate | 3 | Payments, affiliate links, e-commerce conversions |
| Compliance & Audit | 3 | Checks, consent, deletion requests |
| Platform Integration | 1 | OAuth token storage |
| Reference Data | 1 | Categories (hierarchical) |
| **Total** | **21** | Core schema; grows with feature additions |

---

## Key Design Decisions

1. **PostgreSQL RLS for multi-tenancy** — every tenant-scoped table has `organization_id` with row-level security policies. The application sets `app.current_org_id` per request, and PostgreSQL enforces isolation even if application code has bugs.

2. **Creators are shared across organizations** — the `creators` table is global (not tenant-scoped) because the same creator may work with multiple brands. The `campaign_creators` junction table connects them to organization-specific campaigns.

3. **Audience demographics as time-series snapshots** — `creator_audience_demographics` uses `snapshot_date` to track how audiences change over time, enabling trend analysis and detecting sudden shifts that indicate fraud.

4. **Separate audience geography table** — country-level distribution is normalized into its own table because the number of countries varies per creator. This avoids sparse columns and enables efficient geographic queries.

5. **ISO standards for all reference data** — country codes (ISO 3166-1), currency codes (ISO 4217), and language tags (BCP 47) ensure interoperability with external APIs and payment processors.

6. **IAB taxonomy alignment** — creator tiers and campaign types use IAB Creator Economy Taxonomy definitions, enabling cross-platform comparability and alignment with industry reporting standards.

7. **Compliance as first-class entities** — compliance checks, consent records, and deletion requests each have dedicated tables rather than being embedded in other entities. This provides clean audit trails for FTC, ASA, GDPR, and CCPA requirements.

8. **Encrypted OAuth tokens** — tokens are stored as encrypted text, not plaintext. The encryption/decryption happens at the application layer, keeping the database layer simple while meeting security requirements.

9. **Affiliate tracking separate from payments** — affiliate links and e-commerce conversions are tracked independently from payment records, because a conversion event and the resulting payment may occur at different times and go through different approval workflows.

10. **Content compliance is post-level** — each content post has its own compliance status and check history, because FTC requirements apply per-post, not per-campaign or per-creator.
