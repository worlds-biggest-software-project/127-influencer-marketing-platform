# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Influencer Marketing Platform · Created: 2026-05-19

## Philosophy

This model uses a property graph layer alongside relational tables to model the relationship-rich aspects of influencer marketing: creator-to-brand partnerships, audience overlaps, category affiliations, influencer networks, and competitor analysis. The core operational data (campaigns, payments, compliance) lives in conventional relational tables, while the discovery, matching, and analytics layer uses a graph structure implemented via PostgreSQL's `ltree` extension and a flexible node/edge pattern.

Influencer marketing is fundamentally a network problem. Brands want to discover creators who are connected to their target audience, avoid creators whose audiences overlap excessively, detect influencer fraud rings, and trace the chain of influence from creator content to purchase. Graph queries answer these questions naturally — "find creators 2 hops from Brand X's existing creator network who share audience segments with top performers" — whereas relational JOINs become unwieldy for path traversal and network analysis.

This approach is inspired by social graph platforms (LinkedIn, Facebook) and recommendation engines that use graph traversal for discovery. The relational layer handles transactional operations (creating campaigns, processing payments), while the graph layer powers discovery, matching, and analytics workloads.

**Best for:** Teams building advanced discovery, audience overlap detection, influencer network analysis, and AI-powered recommendation features as core differentiators.

**Trade-offs:**
- Pro: Network traversal queries (audience overlap, influencer clusters, brand affinity) are natural and efficient
- Pro: Relationship-first discovery — "find similar creators" queries use graph traversal instead of brute-force filtering
- Pro: Fraud ring detection via connected component analysis on the graph
- Pro: Flexible — new relationship types added by inserting edge rows, not schema changes
- Con: Higher conceptual complexity — developers must understand both relational and graph paradigms
- Con: Graph queries can be expensive without careful index design and traversal depth limits
- Con: No standard PostgreSQL graph query language — must use recursive CTEs or ltree operators
- Con: Dual-model consistency — operational state and graph state must stay in sync
- Con: Less mature tooling compared to pure relational or pure graph databases

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IAB Creator Economy Taxonomy (April 2025) | Category nodes in the graph align with IAB vocabulary; edges link creators to IAB-defined categories |
| IAB Content Taxonomy 3.0 | Content topic classification used as graph node labels for content-based recommendations |
| ISO 3166-1 alpha-2 | Geographic nodes use ISO country codes as identifiers |
| ISO 4217 | Currency codes in relational payment tables |
| FTC 16 CFR Part 255 | Compliance tracked in relational tables with full audit |
| GDPR (EU) 2016/679 | Graph edges involving personal data are tagged for deletion propagation |
| W3C RDF/Property Graph | Edge/node model follows property graph conventions for potential future export to Neo4j or Apache AGE |

---

## Graph Layer

### Graph Nodes

```sql
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type       VARCHAR(50) NOT NULL,
        -- creator, brand, campaign, category, platform, country,
        -- audience_segment, content_topic, hashtag
    external_id     UUID,                       -- reference to relational table row
    label           VARCHAR(255) NOT NULL,       -- human-readable label
    properties      JSONB NOT NULL DEFAULT '{}', -- node-specific attributes
    -- properties examples by node_type:
    --
    -- creator:
    -- {
    --   "tier": "micro",
    --   "fraud_score": 12.5,
    --   "authenticity_score": 87.3,
    --   "primary_platform": "instagram",
    --   "follower_count_total": 165000
    -- }
    --
    -- category:
    -- {
    --   "iab_taxonomy_id": "IAB-601",
    --   "level": 2,
    --   "parent": "Beauty & Personal Care"
    -- }
    --
    -- audience_segment:
    -- {
    --   "age_range": "18-24",
    --   "gender": "female",
    --   "country": "US",
    --   "interest": "skincare"
    -- }
    --
    -- country:
    -- {
    --   "iso_code": "US",
    --   "region": "North America"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_external ON graph_nodes(external_id);
CREATE INDEX idx_graph_nodes_label ON graph_nodes(label);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN (properties);
```

### Graph Edges

```sql
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
        -- has_profile_on, belongs_to_category, targets_audience,
        -- collaborated_with, audience_overlaps, located_in,
        -- similar_to, competes_with, mentions_brand,
        -- participated_in, uses_hashtag, influences_segment
    weight          NUMERIC(7,4),               -- strength/relevance score (0-1 or 0-100)
    properties      JSONB NOT NULL DEFAULT '{}', -- edge-specific attributes
    -- properties examples by edge_type:
    --
    -- has_profile_on (creator -> platform):
    -- {
    --   "username": "janedoe",
    --   "follower_count": 45000,
    --   "engagement_rate": 3.52
    -- }
    --
    -- audience_overlaps (creator -> creator):
    -- {
    --   "overlap_pct": 35.2,
    --   "shared_segments": ["us_female_18_24", "beauty_interest"],
    --   "calculated_at": "2026-05-19T10:00:00Z"
    -- }
    --
    -- collaborated_with (creator -> brand):
    -- {
    --   "campaign_count": 3,
    --   "total_spend": 7500.00,
    --   "avg_roi": 2.8,
    --   "last_campaign_date": "2026-04-15"
    -- }
    --
    -- similar_to (creator -> creator):
    -- {
    --   "similarity_score": 0.87,
    --   "shared_categories": ["beauty", "skincare"],
    --   "algorithm": "content_embedding_cosine"
    -- }
    is_gdpr_relevant BOOLEAN NOT NULL DEFAULT false,  -- flag for deletion propagation
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_until     TIMESTAMPTZ,                       -- null = currently valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_graph_edges_weight ON graph_edges(weight DESC NULLS LAST);
CREATE INDEX idx_graph_edges_source_type ON graph_edges(source_id, edge_type);
CREATE INDEX idx_graph_edges_target_type ON graph_edges(target_id, edge_type);
CREATE INDEX idx_graph_edges_props ON graph_edges USING GIN (properties);
CREATE INDEX idx_graph_edges_valid ON graph_edges(valid_from, valid_until);
```

---

## Graph Query Examples

### Find creators similar to a top performer (lookalike discovery)

```sql
-- Given a high-performing creator, find similar creators via graph traversal
WITH top_creator AS (
    SELECT id FROM graph_nodes
    WHERE node_type = 'creator' AND external_id = '{{creator_uuid}}'
),
similar AS (
    SELECT e.target_id AS similar_node_id,
           e.weight AS similarity_score,
           e.properties->>'shared_categories' AS shared_cats
    FROM graph_edges e
    JOIN top_creator tc ON e.source_id = tc.id
    WHERE e.edge_type = 'similar_to'
      AND e.weight > 0.7
      AND (e.valid_until IS NULL OR e.valid_until > now())
    ORDER BY e.weight DESC
    LIMIT 20
)
SELECT gn.label, gn.properties, s.similarity_score, s.shared_cats
FROM similar s
JOIN graph_nodes gn ON gn.id = s.similar_node_id;
```

### Detect audience overlap between campaign creators

```sql
-- Find all audience overlaps among creators in a specific campaign
WITH campaign_creators AS (
    SELECT gn.id AS node_id, gn.label, gn.external_id
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.source_id = gn.id
    JOIN graph_nodes camp ON ge.target_id = camp.id
    WHERE camp.node_type = 'campaign'
      AND camp.external_id = '{{campaign_uuid}}'
      AND ge.edge_type = 'participated_in'
      AND gn.node_type = 'creator'
)
SELECT c1.label AS creator_1,
       c2.label AS creator_2,
       (e.properties->>'overlap_pct')::numeric AS overlap_pct,
       e.properties->>'shared_segments' AS shared_segments
FROM graph_edges e
JOIN campaign_creators c1 ON e.source_id = c1.node_id
JOIN campaign_creators c2 ON e.target_id = c2.node_id
WHERE e.edge_type = 'audience_overlaps'
  AND (e.properties->>'overlap_pct')::numeric > 20.0
ORDER BY overlap_pct DESC;
```

### Find creators who reach a target audience segment

```sql
-- Find creators connected to a specific audience segment
WITH target_segment AS (
    SELECT id FROM graph_nodes
    WHERE node_type = 'audience_segment'
      AND properties @> '{"age_range": "18-24", "gender": "female", "country": "US"}'
)
SELECT gn.label AS creator_name,
       gn.properties->>'tier' AS tier,
       (gn.properties->>'follower_count_total')::bigint AS followers,
       e.weight AS reach_strength
FROM graph_edges e
JOIN target_segment ts ON e.target_id = ts.id
JOIN graph_nodes gn ON e.source_id = gn.id
WHERE e.edge_type = 'influences_segment'
  AND gn.node_type = 'creator'
  AND e.weight > 0.5
ORDER BY e.weight DESC
LIMIT 50;
```

### Fraud ring detection: find clusters of creators with high mutual overlap

```sql
-- Recursive CTE to find connected creator clusters (potential fraud rings)
WITH RECURSIVE fraud_cluster AS (
    -- Seed: start from a suspicious creator
    SELECT source_id, target_id, 1 AS depth
    FROM graph_edges
    WHERE source_id = (
        SELECT id FROM graph_nodes
        WHERE external_id = '{{suspicious_creator_uuid}}'
          AND node_type = 'creator'
    )
    AND edge_type = 'audience_overlaps'
    AND (properties->>'overlap_pct')::numeric > 80.0

    UNION ALL

    -- Recurse: follow high-overlap edges up to 3 hops
    SELECT e.source_id, e.target_id, fc.depth + 1
    FROM graph_edges e
    JOIN fraud_cluster fc ON e.source_id = fc.target_id
    WHERE e.edge_type = 'audience_overlaps'
      AND (e.properties->>'overlap_pct')::numeric > 80.0
      AND fc.depth < 3
)
SELECT DISTINCT gn.label, gn.properties->>'fraud_score' AS fraud_score
FROM fraud_cluster fc
JOIN graph_nodes gn ON gn.id = fc.target_id
WHERE gn.node_type = 'creator';
```

---

## Relational Layer (Operational Data)

### Organizations & Users

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan_tier       VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(organization_id, email)
);

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY users_tenant ON users
    USING (organization_id = current_setting('app.current_org_id')::uuid);
```

### Creators (Relational)

```sql
CREATE TABLE creators (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    graph_node_id       UUID REFERENCES graph_nodes(id),  -- link to graph layer
    full_name           VARCHAR(255),
    primary_email       VARCHAR(255),
    country_code        CHAR(2),
    tier                VARCHAR(20) NOT NULL DEFAULT 'unknown',
    fraud_score         NUMERIC(5,2),
    authenticity_score  NUMERIC(5,2),
    is_synthetic        BOOLEAN NOT NULL DEFAULT false,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_creators_graph ON creators(graph_node_id);
CREATE INDEX idx_creators_tier ON creators(tier);
```

### Campaigns

```sql
CREATE TABLE campaigns (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    graph_node_id       UUID REFERENCES graph_nodes(id),  -- link to graph layer
    name                VARCHAR(255) NOT NULL,
    status              VARCHAR(30) NOT NULL DEFAULT 'draft',
    campaign_type       VARCHAR(50) NOT NULL,
    budget_total        NUMERIC(12,2),
    budget_spent        NUMERIC(12,2) NOT NULL DEFAULT 0,
    budget_currency     CHAR(3) NOT NULL DEFAULT 'USD',
    start_date          DATE,
    end_date            DATE,
    target_platforms    VARCHAR(50)[] NOT NULL DEFAULT '{}',
    brief               JSONB NOT NULL DEFAULT '{}',
    created_by          UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE campaigns ENABLE ROW LEVEL SECURITY;
CREATE POLICY campaigns_tenant ON campaigns
    USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE INDEX idx_campaigns_org ON campaigns(organization_id);
CREATE INDEX idx_campaigns_status ON campaigns(status);
CREATE INDEX idx_campaigns_graph ON campaigns(graph_node_id);
```

### Campaign Creators

```sql
CREATE TABLE campaign_creators (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id         UUID NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    creator_id          UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    status              VARCHAR(30) NOT NULL DEFAULT 'invited',
    agreed_rate         NUMERIC(10,2),
    rate_currency       CHAR(3) DEFAULT 'USD',
    rate_type           VARCHAR(30),
    commission_pct      NUMERIC(5,2),
    details             JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(campaign_id, creator_id)
);

CREATE INDEX idx_cc_campaign ON campaign_creators(campaign_id);
CREATE INDEX idx_cc_creator ON campaign_creators(creator_id);
```

### Content Posts

```sql
CREATE TABLE content_posts (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_creator_id     UUID NOT NULL REFERENCES campaign_creators(id) ON DELETE CASCADE,
    platform                VARCHAR(50) NOT NULL,
    platform_post_id        VARCHAR(255),
    post_url                VARCHAR(500),
    post_type               VARCHAR(30) NOT NULL,
    caption                 TEXT,
    published_at            TIMESTAMPTZ,
    metrics                 JSONB NOT NULL DEFAULT '{}',
    compliance_status       VARCHAR(30) DEFAULT 'pending',
    compliance_details      JSONB NOT NULL DEFAULT '{}',
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_content_cc ON content_posts(campaign_creator_id);
CREATE INDEX idx_content_compliance ON content_posts(compliance_status);
```

### Payments

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
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
CREATE POLICY payments_tenant ON payments
    USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE INDEX idx_payments_org ON payments(organization_id);
CREATE INDEX idx_payments_status ON payments(status);
```

### Compliance Log

```sql
CREATE TABLE compliance_log (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL,
    entity_type         VARCHAR(30) NOT NULL,
    entity_id           UUID NOT NULL,
    event_type          VARCHAR(50) NOT NULL,
    severity            VARCHAR(20) NOT NULL DEFAULT 'info',
    details             JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_compliance_org ON compliance_log(organization_id);
CREATE INDEX idx_compliance_entity ON compliance_log(entity_type, entity_id);
CREATE INDEX idx_compliance_severity ON compliance_log(severity);
```

### Outreach Messages

```sql
CREATE TABLE outreach_messages (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    campaign_id         UUID REFERENCES campaigns(id),
    creator_id          UUID NOT NULL REFERENCES creators(id) ON DELETE CASCADE,
    message_type        VARCHAR(20) NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'draft',
    subject             VARCHAR(500),
    body                TEXT NOT NULL,
    is_ai_generated     BOOLEAN NOT NULL DEFAULT false,
    tracking            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

ALTER TABLE outreach_messages ENABLE ROW LEVEL SECURITY;
CREATE POLICY outreach_tenant ON outreach_messages
    USING (organization_id = current_setting('app.current_org_id')::uuid);

CREATE INDEX idx_outreach_org ON outreach_messages(organization_id);
CREATE INDEX idx_outreach_creator ON outreach_messages(creator_id);
```

### Integration Configs

```sql
CREATE TABLE integration_configs (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    platform            VARCHAR(50) NOT NULL,
    is_enabled          BOOLEAN NOT NULL DEFAULT true,
    config              JSONB NOT NULL DEFAULT '{}',
    credentials         JSONB NOT NULL DEFAULT '{}',
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

## Graph Synchronization

The graph layer must stay in sync with relational data. This is handled via database triggers or application-layer event handlers.

```sql
-- Example trigger: when a new creator is inserted, create a graph node
CREATE OR REPLACE FUNCTION sync_creator_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_nodes (id, node_type, external_id, label, properties)
    VALUES (
        gen_random_uuid(),
        'creator',
        NEW.id,
        COALESCE(NEW.full_name, 'Unknown'),
        jsonb_build_object(
            'tier', NEW.tier,
            'fraud_score', NEW.fraud_score,
            'authenticity_score', NEW.authenticity_score,
            'country_code', NEW.country_code
        )
    )
    RETURNING id INTO NEW.graph_node_id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_creator_graph_sync
    BEFORE INSERT ON creators
    FOR EACH ROW
    EXECUTE FUNCTION sync_creator_to_graph();

-- Example trigger: when a campaign_creator is inserted, create graph edges
CREATE OR REPLACE FUNCTION sync_campaign_creator_to_graph()
RETURNS TRIGGER AS $$
DECLARE
    creator_node UUID;
    campaign_node UUID;
BEGIN
    SELECT graph_node_id INTO creator_node FROM creators WHERE id = NEW.creator_id;
    SELECT graph_node_id INTO campaign_node FROM campaigns WHERE id = NEW.campaign_id;

    IF creator_node IS NOT NULL AND campaign_node IS NOT NULL THEN
        INSERT INTO graph_edges (source_id, target_id, edge_type, properties)
        VALUES (creator_node, campaign_node, 'participated_in',
                jsonb_build_object('status', NEW.status, 'agreed_rate', NEW.agreed_rate));
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_campaign_creator_graph_sync
    AFTER INSERT ON campaign_creators
    FOR EACH ROW
    EXECUTE FUNCTION sync_campaign_creator_to_graph();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | Nodes + edges with JSONB properties |
| Organization & Users | 2 | Standard relational with RLS |
| Creators | 1 | Linked to graph_nodes via graph_node_id |
| Campaigns | 1 | Linked to graph_nodes via graph_node_id |
| Campaign Creators | 1 | Junction with JSONB details |
| Content Posts | 1 | Metrics and compliance in JSONB |
| Payments | 1 | With JSONB details |
| Compliance | 1 | Unified log |
| Outreach | 1 | With JSONB tracking |
| Integrations | 1 | Per-platform config |
| **Total** | **12** | Plus triggers for graph synchronization |

---

## Key Design Decisions

1. **Property graph in PostgreSQL, not a separate graph database** — using `graph_nodes` and `graph_edges` tables with JSONB properties instead of Neo4j or similar keeps the operational stack simple (one database), while still enabling graph traversal via recursive CTEs. If graph query volume grows, the same model can be migrated to Apache AGE (PostgreSQL graph extension) or exported to Neo4j.

2. **Bidirectional linking between relational and graph** — relational tables have `graph_node_id` pointing into the graph layer, and graph nodes have `external_id` pointing back to relational rows. This enables hybrid queries that start in the graph and finish in relational tables (or vice versa).

3. **Temporal edges with valid_from/valid_until** — relationships change over time (a creator may stop working with a brand, audience overlap percentages shift). Temporal validity on edges enables point-in-time graph queries without deleting historical data.

4. **Audience segments as graph nodes** — rather than storing audience demographics as table rows, they become nodes in the graph connected to creators via weighted `influences_segment` edges. This makes "find creators who reach segment X" a single-hop traversal.

5. **Edge weights enable ranked discovery** — the `weight` column on edges quantifies relationship strength (similarity scores, overlap percentages, reach strength). Discovery queries sort by weight to surface the best matches first.

6. **Fraud ring detection via graph traversal** — the recursive CTE pattern for following high-overlap edges detects clusters of creators with suspiciously similar audiences. This is a query that would be extremely complex in a pure relational model but is natural in a graph.

7. **GDPR flagging on edges** — the `is_gdpr_relevant` flag on edges enables systematic identification and deletion of personal-data relationships during data subject access or deletion requests.

8. **Graph sync via triggers** — database triggers ensure the graph layer stays consistent with relational inserts/updates without requiring application code to manage both layers explicitly. This centralizes the sync logic and prevents drift.

9. **Relational layer handles transactional operations** — campaign CRUD, payment processing, and compliance logging use standard relational patterns with RLS. The graph layer is optimized for read-heavy discovery and analytics, not transactional writes.

10. **Future-proof for dedicated graph database** — the node/edge/properties pattern is compatible with Neo4j's property graph model and Apache AGE. If the graph grows beyond PostgreSQL's recursive CTE performance limits, migration to a dedicated graph engine is straightforward.
