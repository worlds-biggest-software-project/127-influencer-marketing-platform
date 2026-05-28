# Influencer Marketing Platform — Phased Development Plan

> Project: 127-influencer-marketing-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22+) | Full-stack unification — API, frontend, and worker processes share types. Strong ecosystem for social API integrations, webhook handling, and real-time dashboards. LLM SDKs (@anthropic-ai/sdk, openai) are TypeScript-first. |
| API framework | Fastify 5 | Faster than Express with built-in schema validation via JSON Schema, native OpenAPI 3.1 generation via @fastify/swagger, first-class TypeScript support, and plugin architecture that maps well to phased delivery. |
| Frontend | Next.js 15 (App Router) | Server components reduce client bundle for data-heavy dashboards. Server actions handle form submissions. Pairs naturally with the Fastify API via tRPC or REST. Tailwind CSS + shadcn/ui for rapid UI development. |
| Database | PostgreSQL 16 | Row-level security for multi-tenancy, JSONB with GIN indexes for platform-specific data, partitioning for high-volume event/metrics tables, mature ecosystem. Adopting the Hybrid Relational + JSONB data model (Suggestion 3) for fastest MVP with platform flexibility. |
| ORM / query builder | Drizzle ORM | Type-safe SQL with zero runtime overhead, first-class PostgreSQL support (JSONB, arrays, RLS), and migration generation. Lighter than Prisma, more control over raw SQL when needed. |
| Cache / queue | Redis 7 (via BullMQ) | BullMQ for job queues (social API sync, AI scoring, compliance checks, email sending). Redis also serves as cache for creator profile data and rate-limit counters for social API calls. |
| LLM provider | Anthropic Claude API (primary) | Outreach personalization, content quality scoring, compliance text analysis, and creator matching all require strong reasoning. Claude Opus/Sonnet via @anthropic-ai/sdk with prompt caching for cost efficiency. |
| Search | Elasticsearch 8 / OpenSearch | Creator discovery across 50M+ profiles requires full-text search, faceted filtering, geo queries, and relevance scoring beyond PostgreSQL's capabilities. |
| Email | Resend (transactional) | API-first email delivery with open/click tracking, bounce handling, and webhook notifications. Simpler than SendGrid for outreach tracking. |
| Payments | Stripe Connect | Multi-party payments to creators in 120+ countries. Connect Express for onboarding creators, automatic tax form collection (W-9/W-8BEN). |
| Authentication | Lucia Auth + OAuth 2.0 | Lightweight auth library with session management. OAuth 2.0 flows for social platform connections (Instagram Graph API, TikTok, YouTube). |
| Testing | Vitest + Playwright | Vitest for unit/integration tests (fast, ESM-native, compatible with TypeScript). Playwright for E2E browser tests of the dashboard. |
| Code quality | Biome (lint + format) + tsc --noEmit | Biome replaces ESLint + Prettier with faster performance. TypeScript strict mode for type safety. |
| Containerization | Docker + docker-compose | Multi-service stack (API, worker, frontend, PostgreSQL, Redis, Elasticsearch) orchestrated locally. Production deployment via Docker or Kubernetes. |
| Monorepo | Turborepo | Manages shared packages (types, database schema, utilities) across API, frontend, and worker apps. Incremental builds for CI. |
| CI/CD | GitHub Actions | Automated test, lint, build, and Docker image publish pipeline. |

### Project Structure

```
influencer-marketing-platform/
├── turbo.json
├── package.json
├── docker-compose.yml
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── .env.example
├── packages/
│   ├── db/
│   │   ├── src/
│   │   │   ├── schema.ts              # Drizzle schema definitions
│   │   │   ├── migrations/            # Generated SQL migrations
│   │   │   ├── seed.ts                # Development seed data
│   │   │   └── index.ts               # Exported client + schema
│   │   ├── drizzle.config.ts
│   │   └── package.json
│   ├── shared/
│   │   ├── src/
│   │   │   ├── types/                 # Shared TypeScript types
│   │   │   │   ├── creator.ts
│   │   │   │   ├── campaign.ts
│   │   │   │   ├── compliance.ts
│   │   │   │   ├── outreach.ts
│   │   │   │   └── payment.ts
│   │   │   ├── constants/             # IAB taxonomy, platform enums
│   │   │   ├── validators/            # JSON Schema validators for JSONB columns
│   │   │   └── utils/                 # Shared utilities
│   │   └── package.json
│   └── ai/
│       ├── src/
│       │   ├── prompts/               # LLM prompt templates
│       │   ├── scoring/               # AI scoring pipelines
│       │   ├── personalization/       # Outreach generation
│       │   └── compliance/            # Content compliance analysis
│       └── package.json
├── apps/
│   ├── api/
│   │   ├── src/
│   │   │   ├── server.ts              # Fastify server setup
│   │   │   ├── plugins/               # Fastify plugins (auth, RLS, rate-limit)
│   │   │   ├── routes/
│   │   │   │   ├── creators/
│   │   │   │   ├── campaigns/
│   │   │   │   ├── outreach/
│   │   │   │   ├── content/
│   │   │   │   ├── payments/
│   │   │   │   ├── compliance/
│   │   │   │   ├── integrations/
│   │   │   │   └── auth/
│   │   │   ├── services/              # Business logic layer
│   │   │   ├── middleware/
│   │   │   └── webhooks/              # Inbound webhooks (Stripe, social platforms)
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── worker/
│   │   ├── src/
│   │   │   ├── queues/                # BullMQ queue definitions
│   │   │   ├── jobs/
│   │   │   │   ├── sync-creator-profiles.ts
│   │   │   │   ├── run-compliance-check.ts
│   │   │   │   ├── calculate-ai-scores.ts
│   │   │   │   ├── send-outreach-email.ts
│   │   │   │   ├── sync-content-metrics.ts
│   │   │   │   └── process-ecommerce-webhook.ts
│   │   │   └── index.ts
│   │   └── package.json
│   └── web/
│       ├── src/
│       │   ├── app/
│       │   │   ├── (auth)/             # Login, signup, OAuth callbacks
│       │   │   ├── (dashboard)/
│       │   │   │   ├── creators/       # Discovery + creator profiles
│       │   │   │   ├── campaigns/      # Campaign management
│       │   │   │   ├── outreach/       # Outreach sequences
│       │   │   │   ├── content/        # Content tracking
│       │   │   │   ├── payments/       # Payment management
│       │   │   │   ├── compliance/     # Compliance dashboard
│       │   │   │   ├── integrations/   # Platform connections
│       │   │   │   └── settings/       # Org settings
│       │   │   └── layout.tsx
│       │   ├── components/
│       │   │   ├── ui/                 # shadcn/ui components
│       │   │   ├── creators/
│       │   │   ├── campaigns/
│       │   │   └── charts/
│       │   ├── lib/
│       │   │   ├── api-client.ts       # Typed API client
│       │   │   └── hooks/
│       │   └── styles/
│       ├── package.json
│       └── next.config.ts
└── tests/
    ├── fixtures/                       # Shared test data
    │   ├── creators.json
    │   ├── campaigns.json
    │   └── social-api-responses/
    ├── e2e/                            # Playwright E2E tests
    └── integration/                    # Cross-service integration tests
```

---

## Phase 1: Foundation — Project Scaffolding, Database, and Auth

### Purpose

Establish the monorepo structure, database schema, authentication system, and development environment. After this phase, developers can run the full stack locally, create organizations and users, and authenticate via email/password. The database schema supports multi-tenant isolation via PostgreSQL RLS. This phase is the foundation for every subsequent phase.

### Tasks

#### 1.1 — Monorepo and Development Environment Setup

**What**: Initialize the Turborepo monorepo with all packages and apps, Docker Compose for local services, and CI configuration.

**Design**:

Root `package.json` workspaces:
```json
{
  "name": "influencer-marketing-platform",
  "private": true,
  "workspaces": ["packages/*", "apps/*"],
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "turbo lint",
    "typecheck": "turbo typecheck",
    "db:migrate": "turbo db:migrate --filter=@imp/db",
    "db:seed": "turbo db:seed --filter=@imp/db"
  }
}
```

`docker-compose.yml` services:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: imp_dev
      POSTGRES_USER: imp
      POSTGRES_PASSWORD: imp_dev_password
    ports: ["5432:5432"]
    volumes: ["pg_data:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.15.0
    environment:
      discovery.type: single-node
      xpack.security.enabled: "false"
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
    ports: ["9200:9200"]
```

Environment configuration (`packages/shared/src/config.ts`):
```typescript
import { z } from 'zod';

export const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().default('redis://localhost:6379'),
  ELASTICSEARCH_URL: z.string().url().default('http://localhost:9200'),
  ANTHROPIC_API_KEY: z.string().min(1),
  RESEND_API_KEY: z.string().min(1),
  STRIPE_SECRET_KEY: z.string().min(1),
  STRIPE_WEBHOOK_SECRET: z.string().min(1),
  SESSION_SECRET: z.string().min(32),
  APP_URL: z.string().url().default('http://localhost:3000'),
  API_URL: z.string().url().default('http://localhost:3001'),
});

export type Env = z.infer<typeof envSchema>;
```

**Testing**:
- `Unit: envSchema validates complete .env.example without errors`
- `Unit: envSchema rejects missing DATABASE_URL with ZodError`
- `Unit: envSchema applies defaults for optional fields`
- `Integration: docker-compose up starts all services, health checks pass within 30s`
- `Integration: turbo build completes without errors across all packages`

---

#### 1.2 — Database Schema and Migrations

**What**: Implement the Hybrid Relational + JSONB schema (Data Model Suggestion 3) using Drizzle ORM with RLS policies for multi-tenant isolation.

**Design**:

Core schema in `packages/db/src/schema.ts` (excerpted — key tables):

```typescript
import { pgTable, uuid, varchar, text, numeric, boolean, timestamp,
         jsonb, char, index, uniqueIndex } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

// --- Organizations ---
export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  planTier: varchar('plan_tier', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// --- Users ---
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  email: varchar('email', { length: 255 }).notNull(),
  fullName: varchar('full_name', { length: 255 }).notNull(),
  passwordHash: varchar('password_hash', { length: 255 }),
  role: varchar('role', { length: 50 }).notNull().default('member'),
  preferences: jsonb('preferences').notNull().default({}),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgEmailUnique: uniqueIndex('idx_users_org_email').on(table.organizationId, table.email),
  orgIdx: index('idx_users_org').on(table.organizationId),
  emailIdx: index('idx_users_email').on(table.email),
}));

// --- Creators ---
export const creators = pgTable('creators', {
  id: uuid('id').primaryKey().defaultRandom(),
  fullName: varchar('full_name', { length: 255 }),
  primaryEmail: varchar('primary_email', { length: 255 }),
  countryCode: char('country_code', { length: 2 }),        // ISO 3166-1 alpha-2
  tier: varchar('tier', { length: 20 }).notNull().default('unknown'),
      // IAB taxonomy: nano, micro, mid, macro, mega
  fraudScore: numeric('fraud_score', { precision: 5, scale: 2 }),
  authenticityScore: numeric('authenticity_score', { precision: 5, scale: 2 }),
  isSynthetic: boolean('is_synthetic').notNull().default(false),
  profiles: jsonb('profiles').notNull().default({}),
      // Per-platform: { instagram: { username, follower_count, ... }, tiktok: { ... } }
  audience: jsonb('audience').notNull().default({}),
      // Per-platform demographics: { instagram: { age: {...}, gender: {...}, countries: [...] } }
  aiScores: jsonb('ai_scores').notNull().default({}),
      // { brand_safety, content_quality, fatigue_risk, niche_authority: {...}, sentiment: {...} }
  contactInfo: jsonb('contact_info').notNull().default({}),
  categories: varchar('categories', { length: 100 }).array().notNull().default(sql`'{}'`),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  tierIdx: index('idx_creators_tier').on(table.tier),
  countryIdx: index('idx_creators_country').on(table.countryCode),
  fraudIdx: index('idx_creators_fraud').on(table.fraudScore),
}));

// --- Campaigns ---
export const campaigns = pgTable('campaigns', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  status: varchar('status', { length: 30 }).notNull().default('draft'),
      // draft, active, paused, completed, archived
  campaignType: varchar('campaign_type', { length: 50 }).notNull(),
      // IAB: sponsored_post, product_seeding, affiliate, ambassador, ugc
  objective: varchar('objective', { length: 50 }),
      // awareness, engagement, conversion, traffic
  budgetTotal: numeric('budget_total', { precision: 12, scale: 2 }),
  budgetSpent: numeric('budget_spent', { precision: 12, scale: 2 }).notNull().default('0'),
  budgetCurrency: char('budget_currency', { length: 3 }).notNull().default('USD'),
  startDate: timestamp('start_date', { mode: 'date' }),
  endDate: timestamp('end_date', { mode: 'date' }),
  targetPlatforms: varchar('target_platforms', { length: 50 }).array().notNull().default(sql`'{}'`),
  brief: jsonb('brief').notNull().default({}),
  metrics: jsonb('metrics').notNull().default({}),
  createdBy: uuid('created_by').references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgIdx: index('idx_campaigns_org').on(table.organizationId),
  statusIdx: index('idx_campaigns_status').on(table.status),
}));

// --- Campaign Creators ---
export const campaignCreators = pgTable('campaign_creators', {
  id: uuid('id').primaryKey().defaultRandom(),
  campaignId: uuid('campaign_id').notNull().references(() => campaigns.id, { onDelete: 'cascade' }),
  creatorId: uuid('creator_id').notNull().references(() => creators.id, { onDelete: 'cascade' }),
  status: varchar('status', { length: 30 }).notNull().default('invited'),
  agreedRate: numeric('agreed_rate', { precision: 10, scale: 2 }),
  rateCurrency: char('rate_currency', { length: 3 }).default('USD'),
  rateType: varchar('rate_type', { length: 30 }),
  commissionPct: numeric('commission_pct', { precision: 5, scale: 2 }),
  details: jsonb('details').notNull().default({}),
  performance: jsonb('performance').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  campaignCreatorUnique: uniqueIndex('idx_cc_unique').on(table.campaignId, table.creatorId),
  campaignIdx: index('idx_cc_campaign').on(table.campaignId),
  creatorIdx: index('idx_cc_creator').on(table.creatorId),
}));

// --- Content Posts ---
export const contentPosts = pgTable('content_posts', {
  id: uuid('id').primaryKey().defaultRandom(),
  campaignCreatorId: uuid('campaign_creator_id').notNull().references(() => campaignCreators.id, { onDelete: 'cascade' }),
  platform: varchar('platform', { length: 50 }).notNull(),
  platformPostId: varchar('platform_post_id', { length: 255 }),
  postUrl: varchar('post_url', { length: 500 }),
  postType: varchar('post_type', { length: 30 }).notNull(),
  caption: text('caption'),
  publishedAt: timestamp('published_at', { withTimezone: true }),
  metrics: jsonb('metrics').notNull().default({}),
  compliance: jsonb('compliance').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  ccIdx: index('idx_content_cc').on(table.campaignCreatorId),
  platformIdx: index('idx_content_platform').on(table.platform),
  publishedIdx: index('idx_content_published').on(table.publishedAt),
}));

// --- Outreach Messages ---
export const outreachMessages = pgTable('outreach_messages', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  campaignId: uuid('campaign_id').references(() => campaigns.id),
  creatorId: uuid('creator_id').notNull().references(() => creators.id, { onDelete: 'cascade' }),
  messageType: varchar('message_type', { length: 20 }).notNull(),
  status: varchar('status', { length: 20 }).notNull().default('draft'),
  subject: varchar('subject', { length: 500 }),
  body: text('body').notNull(),
  isAiGenerated: boolean('is_ai_generated').notNull().default(false),
  aiContext: jsonb('ai_context').notNull().default({}),
  tracking: jsonb('tracking').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgIdx: index('idx_outreach_org').on(table.organizationId),
  creatorIdx: index('idx_outreach_creator').on(table.creatorId),
  statusIdx: index('idx_outreach_status').on(table.status),
}));

// --- Payments ---
export const payments = pgTable('payments', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  campaignCreatorId: uuid('campaign_creator_id').notNull().references(() => campaignCreators.id, { onDelete: 'cascade' }),
  amount: numeric('amount', { precision: 12, scale: 2 }).notNull(),
  currency: char('currency', { length: 3 }).notNull().default('USD'),
  paymentType: varchar('payment_type', { length: 30 }).notNull(),
  status: varchar('status', { length: 20 }).notNull().default('pending'),
  details: jsonb('details').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgIdx: index('idx_payments_org').on(table.organizationId),
  ccIdx: index('idx_payments_cc').on(table.campaignCreatorId),
  statusIdx: index('idx_payments_status').on(table.status),
}));

// --- Affiliate Links ---
export const affiliateLinks = pgTable('affiliate_links', {
  id: uuid('id').primaryKey().defaultRandom(),
  campaignCreatorId: uuid('campaign_creator_id').notNull().references(() => campaignCreators.id, { onDelete: 'cascade' }),
  linkType: varchar('link_type', { length: 20 }).notNull(),
  trackingUrl: varchar('tracking_url', { length: 500 }),
  discountCode: varchar('discount_code', { length: 100 }),
  destinationUrl: varchar('destination_url', { length: 500 }),
  isActive: boolean('is_active').notNull().default(true),
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  stats: jsonb('stats').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  ccIdx: index('idx_affiliate_cc').on(table.campaignCreatorId),
  codeIdx: index('idx_affiliate_code').on(table.discountCode),
}));

// --- Compliance Log ---
export const complianceLog = pgTable('compliance_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull(),
  entityType: varchar('entity_type', { length: 30 }).notNull(),
  entityId: uuid('entity_id').notNull(),
  eventType: varchar('event_type', { length: 50 }).notNull(),
  severity: varchar('severity', { length: 20 }).notNull().default('info'),
  details: jsonb('details').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgIdx: index('idx_compliance_org').on(table.organizationId),
  entityIdx: index('idx_compliance_entity').on(table.entityType, table.entityId),
  severityIdx: index('idx_compliance_severity').on(table.severity),
}));

// --- Integration Configs ---
export const integrationConfigs = pgTable('integration_configs', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  platform: varchar('platform', { length: 50 }).notNull(),
  isEnabled: boolean('is_enabled').notNull().default(true),
  config: jsonb('config').notNull().default({}),
  credentials: jsonb('credentials').notNull().default({}),
  lastSyncedAt: timestamp('last_synced_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgPlatformUnique: uniqueIndex('idx_integration_org_platform').on(table.organizationId, table.platform),
}));
```

RLS setup migration (raw SQL executed after table creation):
```sql
-- Enable RLS on tenant-scoped tables
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE campaigns ENABLE ROW LEVEL SECURITY;
ALTER TABLE outreach_messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE payments ENABLE ROW LEVEL SECURITY;
ALTER TABLE integration_configs ENABLE ROW LEVEL SECURITY;

-- Create RLS policies
CREATE POLICY users_tenant ON users
  USING (organization_id = current_setting('app.current_org_id')::uuid);
CREATE POLICY campaigns_tenant ON campaigns
  USING (organization_id = current_setting('app.current_org_id')::uuid);
CREATE POLICY outreach_tenant ON outreach_messages
  USING (organization_id = current_setting('app.current_org_id')::uuid);
CREATE POLICY payments_tenant ON payments
  USING (organization_id = current_setting('app.current_org_id')::uuid);
CREATE POLICY integrations_tenant ON integration_configs
  USING (organization_id = current_setting('app.current_org_id')::uuid);
```

RLS context helper (`packages/db/src/rls.ts`):
```typescript
import { sql } from 'drizzle-orm';
import type { PostgresJsDatabase } from 'drizzle-orm/postgres-js';

export async function withTenant<T>(
  db: PostgresJsDatabase,
  organizationId: string,
  fn: (db: PostgresJsDatabase) => Promise<T>
): Promise<T> {
  return db.transaction(async (tx) => {
    await tx.execute(sql`SET LOCAL app.current_org_id = ${organizationId}`);
    return fn(tx as unknown as PostgresJsDatabase);
  });
}
```

**Testing**:
- `Unit: Drizzle schema compiles and generates valid SQL migration files`
- `Integration: migrations apply cleanly to empty PostgreSQL database`
- `Integration: RLS prevents cross-tenant data access — user in org A cannot query org B campaigns`
- `Integration: withTenant sets session variable and queries return only tenant-scoped rows`
- `Integration: seed script inserts sample organizations, users, and creators without constraint violations`
- `Unit: rollback migration drops all tables and policies cleanly`

---

#### 1.3 — Authentication and Session Management

**What**: Implement email/password authentication with Lucia Auth, session management, and organization-scoped authorization.

**Design**:

Auth service (`apps/api/src/services/auth.service.ts`):
```typescript
import { Lucia } from 'lucia';
import { DrizzlePostgreSQLAdapter } from '@lucia-auth/adapter-drizzle';
import { Argon2id } from 'oslo/password';

export interface SignupInput {
  email: string;
  fullName: string;
  password: string;
  organizationName: string;
}

export interface LoginInput {
  email: string;
  password: string;
}

export interface AuthSession {
  userId: string;
  organizationId: string;
  role: 'owner' | 'admin' | 'manager' | 'member' | 'viewer';
}

export class AuthService {
  constructor(
    private lucia: Lucia,
    private db: PostgresJsDatabase
  ) {}

  async signup(input: SignupInput): Promise<AuthSession>;
  async login(input: LoginInput): Promise<{ session: AuthSession; sessionId: string }>;
  async logout(sessionId: string): Promise<void>;
  async validateSession(sessionId: string): Promise<AuthSession | null>;
}
```

Session database tables (added to schema):
```typescript
export const sessions = pgTable('sessions', {
  id: varchar('id', { length: 255 }).primaryKey(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
});
```

Auth routes (`apps/api/src/routes/auth/`):
```typescript
// POST /api/auth/signup
// Request: { email, fullName, password, organizationName }
// Response: { user: { id, email, fullName, role }, organization: { id, name, slug } }
// Creates org + user in transaction. Sets session cookie.

// POST /api/auth/login
// Request: { email, password }
// Response: { user: { id, email, fullName, role, organizationId } }
// Validates credentials, creates session. Sets session cookie.

// POST /api/auth/logout
// Invalidates session. Clears cookie.

// GET /api/auth/me
// Returns current user and organization from session.
```

Auth Fastify plugin (`apps/api/src/plugins/auth.ts`):
```typescript
import { FastifyPluginAsync } from 'fastify';

declare module 'fastify' {
  interface FastifyRequest {
    session: AuthSession | null;
  }
}

export const authPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.decorateRequest('session', null);
  fastify.addHook('onRequest', async (request) => {
    const sessionId = request.cookies['imp_session'];
    if (sessionId) {
      request.session = await authService.validateSession(sessionId);
    }
  });
};

// Route-level guard
export function requireAuth(
  request: FastifyRequest,
  reply: FastifyReply
): asserts request is FastifyRequest & { session: AuthSession } {
  if (!request.session) {
    reply.code(401).send({ error: 'Unauthorized' });
    throw new Error('Unauthorized');
  }
}
```

**Testing**:
- `Unit: signup hashes password with Argon2id, stored hash is not plaintext`
- `Unit: signup rejects duplicate email within same organization`
- `Unit: signup creates organization with slug derived from name`
- `Unit: login with correct password returns valid session`
- `Unit: login with wrong password returns 401`
- `Unit: validateSession returns null for expired session`
- `Integration: full signup -> login -> /me -> logout flow works end-to-end`
- `Integration: requests without session cookie receive 401 on protected routes`
- `Integration: session cookie is httpOnly, secure (in production), sameSite=lax`

---

#### 1.4 — Fastify API Server and OpenAPI Setup

**What**: Configure the Fastify server with plugins for CORS, rate limiting, request validation, error handling, and auto-generated OpenAPI 3.1 documentation.

**Design**:

Server setup (`apps/api/src/server.ts`):
```typescript
import Fastify from 'fastify';
import cors from '@fastify/cors';
import cookie from '@fastify/cookie';
import rateLimit from '@fastify/rate-limit';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';

export async function buildServer() {
  const app = Fastify({
    logger: true,
    ajv: { customOptions: { removeAdditional: 'all', coerceTypes: false } },
  });

  await app.register(cors, { origin: process.env.APP_URL, credentials: true });
  await app.register(cookie, { secret: process.env.SESSION_SECRET });
  await app.register(rateLimit, { max: 100, timeWindow: '1 minute' });

  await app.register(swagger, {
    openapi: {
      info: {
        title: 'Influencer Marketing Platform API',
        version: '1.0.0',
        description: 'API for influencer discovery, campaign management, and ROI tracking',
      },
      servers: [{ url: process.env.API_URL || 'http://localhost:3001' }],
    },
  });
  await app.register(swaggerUi, { routePrefix: '/docs' });

  // Register auth plugin, then route modules
  await app.register(authPlugin);
  await app.register(authRoutes, { prefix: '/api/auth' });

  // Health check
  app.get('/health', async () => ({ status: 'ok', timestamp: new Date().toISOString() }));

  return app;
}
```

Standard error response format:
```typescript
interface ApiError {
  error: string;
  message: string;
  statusCode: number;
  details?: Record<string, unknown>;
}
```

**Testing**:
- `Unit: /health returns 200 with status ok`
- `Unit: unknown routes return 404 with ApiError format`
- `Unit: request validation failure returns 400 with field-level details`
- `Unit: rate limit returns 429 after exceeding threshold`
- `Integration: /docs serves Swagger UI with all registered routes`
- `Integration: CORS headers present for configured origin, absent for others`

---

## Phase 2: Creator Discovery Foundation

### Purpose

Build the creator data model, bulk import pipeline, Elasticsearch indexing, and the discovery API with advanced filtering. After this phase, users can search and filter creators by platform, location, engagement rate, follower count, audience demographics, and content categories. This is the core value proposition of the platform.

### Tasks

#### 2.1 — Creator Data Import Pipeline

**What**: Build a job-based pipeline that imports creator profile data from third-party data aggregators (Modash API or Phyllo API) into the creators table and indexes them in Elasticsearch.

**Design**:

Creator import types (`packages/shared/src/types/creator.ts`):
```typescript
export type Platform = 'instagram' | 'youtube' | 'tiktok' | 'linkedin' | 'x' | 'substack';

export type CreatorTier = 'nano' | 'micro' | 'mid' | 'macro' | 'mega' | 'unknown';
// IAB Creator Economy Taxonomy alignment:
// nano: 1K-10K, micro: 10K-50K, mid: 50K-500K, macro: 500K-1M, mega: 1M+

export interface PlatformProfile {
  platformUserId: string;
  username: string;
  profileUrl: string;
  followerCount: number;
  followingCount?: number;
  postCount?: number;
  avgEngagementRate: number;
  avgLikes?: number;
  avgComments?: number;
  avgViews?: number;
  growthRate30d?: number;
  isVerified: boolean;
  lastSyncedAt: string; // ISO 8601
}

export interface AudienceDemographics {
  snapshotDate: string; // ISO 8601 date
  age: Record<string, number>;       // { "18-24": 35.2, "25-34": 42.1, ... }
  gender: Record<string, number>;    // { "male": 32.0, "female": 66.5, ... }
  countries: Array<{ code: string; pct: number }>;  // ISO 3166-1 alpha-2
  interests?: string[];
  languages?: Array<{ code: string; pct: number }>; // BCP 47
}

export interface CreatorRecord {
  id: string;
  fullName: string;
  primaryEmail?: string;
  countryCode?: string;    // ISO 3166-1 alpha-2
  tier: CreatorTier;
  fraudScore?: number;     // 0-100
  authenticityScore?: number; // 0-100
  isSynthetic: boolean;
  profiles: Record<Platform, PlatformProfile>;
  audience: Record<Platform, AudienceDemographics>;
  aiScores: Record<string, unknown>;
  categories: string[];
}
```

Import job (`apps/worker/src/jobs/sync-creator-profiles.ts`):
```typescript
import { Job } from 'bullmq';

interface SyncCreatorJobData {
  source: 'modash' | 'phyllo' | 'manual';
  batchId: string;
  creatorIds?: string[];      // specific creators to sync
  filters?: {                 // for bulk discovery imports
    platform: Platform;
    minFollowers: number;
    maxFollowers: number;
    countries?: string[];
  };
}

// Job processes:
// 1. Fetch creator data from source API (paginated, respecting rate limits)
// 2. Transform response to CreatorRecord format
// 3. Upsert into PostgreSQL creators table (ON CONFLICT on platform_user_id)
// 4. Calculate tier from max follower count across platforms
// 5. Index/update Elasticsearch document
// 6. Emit 'creator.synced' event for downstream consumers
```

Tier calculation:
```typescript
export function calculateTier(profiles: Record<Platform, PlatformProfile>): CreatorTier {
  const maxFollowers = Math.max(
    ...Object.values(profiles).map(p => p.followerCount)
  );
  if (maxFollowers >= 1_000_000) return 'mega';
  if (maxFollowers >= 500_000) return 'macro';
  if (maxFollowers >= 50_000) return 'mid';
  if (maxFollowers >= 10_000) return 'micro';
  if (maxFollowers >= 1_000) return 'nano';
  return 'unknown';
}
```

**Testing**:
- `Unit: calculateTier returns correct tier for boundary values (999, 1000, 9999, 10000, etc.)`
- `Unit: transform function maps Modash API response to CreatorRecord correctly`
- `Unit: transform function handles missing optional fields without errors`
- `Integration (mocked API): import job fetches paginated results and inserts all creators`
- `Integration (mocked API): import job respects rate limit (max N requests/second)`
- `Integration: duplicate creator import (same platform_user_id) updates existing record, does not create duplicate`
- `Integration: Elasticsearch document created for each imported creator`

---

#### 2.2 — Elasticsearch Creator Index

**What**: Define the Elasticsearch index mapping for creators with analyzers for full-text search and configure the sync pipeline from PostgreSQL.

**Design**:

Index mapping:
```typescript
const creatorIndexMapping = {
  settings: {
    number_of_shards: 2,
    number_of_replicas: 1,
    analysis: {
      analyzer: {
        username_analyzer: {
          type: 'custom',
          tokenizer: 'standard',
          filter: ['lowercase', 'asciifolding'],
        },
      },
    },
  },
  mappings: {
    properties: {
      id: { type: 'keyword' },
      fullName: { type: 'text', analyzer: 'standard', fields: { keyword: { type: 'keyword' } } },
      countryCode: { type: 'keyword' },
      tier: { type: 'keyword' },
      categories: { type: 'keyword' },
      fraudScore: { type: 'float' },
      authenticityScore: { type: 'float' },
      isSynthetic: { type: 'boolean' },
      platforms: { type: 'keyword' },     // derived: ['instagram', 'tiktok']
      totalFollowers: { type: 'long' },   // derived: sum across platforms
      maxEngagementRate: { type: 'float' }, // derived: max across platforms
      profiles: {
        type: 'nested',
        properties: {
          platform: { type: 'keyword' },
          username: { type: 'text', analyzer: 'username_analyzer', fields: { keyword: { type: 'keyword' } } },
          followerCount: { type: 'long' },
          avgEngagementRate: { type: 'float' },
          isVerified: { type: 'boolean' },
        },
      },
      audience: {
        type: 'nested',
        properties: {
          platform: { type: 'keyword' },
          topCountry: { type: 'keyword' },
          femalePercent: { type: 'float' },
          age18to24Percent: { type: 'float' },
          age25to34Percent: { type: 'float' },
        },
      },
      updatedAt: { type: 'date' },
    },
  },
};
```

Search service (`apps/api/src/services/creator-search.service.ts`):
```typescript
export interface CreatorSearchFilters {
  query?: string;               // full-text search across name, username, bio
  platforms?: Platform[];       // must have profile on these platforms
  countries?: string[];         // creator country (ISO 3166-1)
  tiers?: CreatorTier[];
  categories?: string[];
  minFollowers?: number;
  maxFollowers?: number;
  minEngagementRate?: number;
  maxEngagementRate?: number;
  maxFraudScore?: number;
  audienceCountries?: string[]; // audience geography filter
  audienceGenderFemaleMin?: number;
  audienceAgeRange?: string;    // e.g. "18-24"
  audienceAgeMin?: number;      // min percentage for the age range
  excludeSynthetic?: boolean;
  sort?: 'relevance' | 'followers_desc' | 'engagement_desc' | 'fraud_asc';
  page?: number;
  pageSize?: number;            // max 100
}

export interface CreatorSearchResult {
  total: number;
  page: number;
  pageSize: number;
  creators: CreatorRecord[];
}

export class CreatorSearchService {
  async search(filters: CreatorSearchFilters): Promise<CreatorSearchResult>;
  async getById(id: string): Promise<CreatorRecord | null>;
  async getSimilar(id: string, limit?: number): Promise<CreatorRecord[]>;
}
```

**Testing**:
- `Unit: search with no filters returns paginated results with correct total count`
- `Unit: platform filter returns only creators with matching platform profiles`
- `Unit: engagement rate range filter works correctly at boundaries`
- `Unit: audience country filter matches creators with >10% audience in specified country`
- `Unit: sort by followers_desc orders correctly`
- `Unit: excludeSynthetic=true omits creators where isSynthetic is true`
- `Integration: full-text search for username finds correct creator`
- `Integration: combined filters (platform + country + engagement) return intersection`
- `Integration: pagination returns correct page/total, no duplicates across pages`
- `Fixture-based: search against 1000-creator fixture dataset returns expected results for 10 predefined queries`

---

#### 2.3 — Creator Discovery API Routes

**What**: REST API endpoints for searching, viewing, and listing creators, exposed through the Fastify server.

**Design**:

Routes (`apps/api/src/routes/creators/`):
```typescript
// GET /api/creators/search
// Query params: all fields from CreatorSearchFilters (flattened)
// Response: CreatorSearchResult
// Auth: required (any role)

// GET /api/creators/:id
// Response: CreatorRecord (full detail)
// Auth: required (any role)

// GET /api/creators/:id/similar
// Query: { limit?: number }
// Response: { creators: CreatorRecord[] }
// Auth: required (any role)

// POST /api/creators/import
// Body: { source: 'modash' | 'phyllo', filters: {...} }
// Response: { jobId: string, status: 'queued' }
// Auth: required (admin or owner)
// Enqueues a sync-creator-profiles job

// GET /api/creators/import/:jobId/status
// Response: { jobId, status: 'queued' | 'active' | 'completed' | 'failed', progress: number }
// Auth: required (admin or owner)
```

Fastify schema validation example:
```typescript
const searchSchema = {
  querystring: {
    type: 'object',
    properties: {
      query: { type: 'string', maxLength: 500 },
      platforms: { type: 'string' },  // comma-separated, parsed in handler
      countries: { type: 'string' },
      tiers: { type: 'string' },
      categories: { type: 'string' },
      minFollowers: { type: 'integer', minimum: 0 },
      maxFollowers: { type: 'integer', minimum: 0 },
      minEngagementRate: { type: 'number', minimum: 0, maximum: 100 },
      maxEngagementRate: { type: 'number', minimum: 0, maximum: 100 },
      maxFraudScore: { type: 'number', minimum: 0, maximum: 100 },
      excludeSynthetic: { type: 'boolean', default: true },
      sort: { type: 'string', enum: ['relevance', 'followers_desc', 'engagement_desc', 'fraud_asc'] },
      page: { type: 'integer', minimum: 1, default: 1 },
      pageSize: { type: 'integer', minimum: 1, maximum: 100, default: 20 },
    },
  },
};
```

**Testing**:
- `Integration: GET /api/creators/search without auth returns 401`
- `Integration: GET /api/creators/search with auth returns paginated results`
- `Integration: GET /api/creators/search?platforms=instagram,tiktok filters correctly`
- `Integration: GET /api/creators/:id returns full creator record`
- `Integration: GET /api/creators/:id with nonexistent ID returns 404`
- `Integration: POST /api/creators/import by member role returns 403`
- `Integration: POST /api/creators/import by admin returns 202 with jobId`
- `E2E: search page renders creator cards with platform badges and engagement rates`

---

## Phase 3: Campaign Management

### Purpose

Build the campaign lifecycle — creation, creator invitations, brief management, and status tracking. After this phase, users can create campaigns, invite creators, manage contract terms, and track campaign status through its lifecycle (draft -> active -> completed). This is the operational backbone of the platform.

### Tasks

#### 3.1 — Campaign CRUD and Lifecycle

**What**: API endpoints and business logic for creating, updating, and managing campaign lifecycle states.

**Design**:

Campaign service (`apps/api/src/services/campaign.service.ts`):
```typescript
export interface CreateCampaignInput {
  name: string;
  campaignType: 'sponsored_post' | 'product_seeding' | 'affiliate' | 'ambassador' | 'ugc';
  objective?: 'awareness' | 'engagement' | 'conversion' | 'traffic';
  budgetTotal?: number;
  budgetCurrency?: string;       // ISO 4217, default USD
  startDate?: string;            // ISO 8601
  endDate?: string;
  targetPlatforms: Platform[];
  brief?: CampaignBrief;
}

export interface CampaignBrief {
  title: string;
  content: string;               // markdown
  deliverables: string[];
  hashtagsRequired: string[];    // must include #ad or #sponsored for FTC
  disclosureText: string;
  dos: string[];
  donts: string[];
}

// State machine for campaign lifecycle
// draft -> active -> paused -> active (resume)
// draft -> active -> completed
// any -> archived
export type CampaignStatus = 'draft' | 'active' | 'paused' | 'completed' | 'archived';

const VALID_TRANSITIONS: Record<CampaignStatus, CampaignStatus[]> = {
  draft: ['active', 'archived'],
  active: ['paused', 'completed', 'archived'],
  paused: ['active', 'completed', 'archived'],
  completed: ['archived'],
  archived: [],
};

export class CampaignService {
  async create(orgId: string, userId: string, input: CreateCampaignInput): Promise<Campaign>;
  async update(campaignId: string, input: Partial<CreateCampaignInput>): Promise<Campaign>;
  async transition(campaignId: string, newStatus: CampaignStatus): Promise<Campaign>;
  async list(orgId: string, filters: { status?: CampaignStatus; page?: number }): Promise<PaginatedResult<Campaign>>;
  async getById(campaignId: string): Promise<Campaign>;
  async delete(campaignId: string): Promise<void>; // only draft campaigns
}
```

API routes:
```typescript
// POST   /api/campaigns              — Create campaign
// GET    /api/campaigns              — List campaigns (org-scoped via RLS)
// GET    /api/campaigns/:id          — Get campaign detail
// PATCH  /api/campaigns/:id          — Update campaign
// POST   /api/campaigns/:id/activate — Transition to active
// POST   /api/campaigns/:id/pause    — Transition to paused
// POST   /api/campaigns/:id/complete — Transition to completed
// POST   /api/campaigns/:id/archive  — Transition to archived
// DELETE /api/campaigns/:id          — Delete (draft only)
```

**Testing**:
- `Unit: create campaign sets status to draft, budgetSpent to 0`
- `Unit: create campaign validates budgetCurrency against ISO 4217 codes`
- `Unit: transition from draft to active succeeds`
- `Unit: transition from draft to completed fails with InvalidTransitionError`
- `Unit: transition from archived to any status fails`
- `Unit: delete campaign in active status returns 409 Conflict`
- `Unit: delete draft campaign succeeds`
- `Integration: created campaign visible in list, scoped to creating org only`
- `Integration: campaign with brief includes FTC-required disclosure fields`

---

#### 3.2 — Campaign Creator Management

**What**: Invite creators to campaigns, manage negotiation status, set rates and contract terms.

**Design**:

Campaign creator service:
```typescript
export interface InviteCreatorInput {
  creatorId: string;
  agreedRate?: number;
  rateCurrency?: string;
  rateType?: 'flat_fee' | 'per_post' | 'cpm' | 'commission_pct';
  commissionPct?: number;
  deliverables?: Array<{
    type: string;
    count: number;
    dueDate?: string;
  }>;
}

// Status lifecycle for campaign creators:
// invited -> negotiating -> contracted -> active -> completed
// invited -> declined
// any -> removed
export type CampaignCreatorStatus =
  'invited' | 'negotiating' | 'contracted' | 'active' | 'completed' | 'declined' | 'removed';

export class CampaignCreatorService {
  async invite(campaignId: string, input: InviteCreatorInput): Promise<CampaignCreator>;
  async bulkInvite(campaignId: string, inputs: InviteCreatorInput[]): Promise<CampaignCreator[]>;
  async updateStatus(id: string, status: CampaignCreatorStatus): Promise<CampaignCreator>;
  async updateTerms(id: string, terms: Partial<InviteCreatorInput>): Promise<CampaignCreator>;
  async list(campaignId: string): Promise<CampaignCreator[]>;
  async remove(id: string): Promise<void>;
}
```

API routes:
```typescript
// POST   /api/campaigns/:id/creators        — Invite creator(s)
// GET    /api/campaigns/:id/creators        — List campaign creators
// PATCH  /api/campaigns/:id/creators/:ccId  — Update terms or status
// DELETE /api/campaigns/:id/creators/:ccId  — Remove creator
```

**Testing**:
- `Unit: invite creator to campaign creates campaign_creators row with status invited`
- `Unit: invite same creator twice to same campaign returns 409`
- `Unit: bulk invite 10 creators creates 10 rows in single transaction`
- `Unit: status transition invited -> contracted succeeds`
- `Unit: status transition completed -> invited fails`
- `Unit: updateTerms changes agreedRate and stores deliverables in details JSONB`
- `Integration: removed creator no longer appears in campaign creator list`
- `Integration: campaign creator list includes creator profile data via join`

---

#### 3.3 — Campaign Dashboard API

**What**: Aggregated campaign metrics endpoint for the dashboard, rolling up creator performance, content counts, and budget utilization.

**Design**:

```typescript
export interface CampaignDashboard {
  campaign: Campaign;
  summary: {
    creatorCount: number;
    creatorsByStatus: Record<CampaignCreatorStatus, number>;
    contentCount: number;
    totalImpressions: number;
    totalEngagement: number;
    totalClicks: number;
    totalConversions: number;
    totalRevenue: number;
    budgetUtilization: number;  // budgetSpent / budgetTotal as percentage
    roi: number;               // totalRevenue / budgetSpent
    complianceIssues: number;
  };
  topCreators: Array<{
    creator: CreatorRecord;
    impressions: number;
    engagement: number;
    conversions: number;
    revenue: number;
  }>;
}

// GET /api/campaigns/:id/dashboard
// Response: CampaignDashboard
```

**Testing**:
- `Unit: dashboard returns correct creator counts by status`
- `Unit: dashboard ROI calculation handles zero budget (returns 0, not NaN)`
- `Unit: topCreators sorted by revenue descending, limited to 10`
- `Integration: dashboard metrics update after content post metrics are synced`
- `Fixture-based: campaign with 5 creators and 15 posts returns expected aggregate values`

---

## Phase 4: Outreach and CRM

### Purpose

Build the outreach system — email templates, send/track workflows, and the creator CRM. After this phase, users can send outreach emails to creators (manually or from templates), track opens/replies, and manage the full creator relationship lifecycle. This phase also introduces the AI-powered outreach personalization.

### Tasks

#### 4.1 — Outreach Email Management

**What**: Create, send, and track outreach emails to creators with template support, send scheduling, and event tracking via Resend webhooks.

**Design**:

Outreach service:
```typescript
export interface CreateOutreachInput {
  campaignId?: string;
  creatorId: string;
  subject: string;
  body: string;               // HTML email body
  messageType: 'email';
  scheduledAt?: string;        // ISO 8601 — if set, email queued for future send
}

export interface OutreachTemplate {
  id: string;
  organizationId: string;
  name: string;
  subject: string;
  body: string;                // Handlebars template with {{creator.fullName}}, {{campaign.name}}, etc.
  variables: string[];         // extracted variable names
}

export class OutreachService {
  async create(orgId: string, input: CreateOutreachInput): Promise<OutreachMessage>;
  async sendNow(messageId: string): Promise<void>;
  async scheduleAll(sequenceIds: string[]): Promise<void>;
  async handleWebhook(event: ResendWebhookEvent): Promise<void>;
    // Updates tracking JSONB: sent_at, delivered_at, opened_at, replied_at, bounced
  async listByCreator(creatorId: string, orgId: string): Promise<OutreachMessage[]>;
  async listByCampaign(campaignId: string): Promise<OutreachMessage[]>;
}
```

Resend webhook handler (`apps/api/src/webhooks/resend.ts`):
```typescript
// POST /api/webhooks/resend
// Validates webhook signature
// Events: email.sent, email.delivered, email.opened, email.clicked, email.bounced
// Updates outreach_messages.tracking JSONB with event timestamps
```

BullMQ job for scheduled sends:
```typescript
// Job: send-outreach-email
// Data: { messageId: string }
// Delay: calculated from scheduledAt - now()
// Process: fetch message, call Resend API, update status to 'sent'
// Retry: 3 attempts with exponential backoff
```

**Testing**:
- `Unit: create outreach message validates email format for creator`
- `Unit: template rendering substitutes all variables correctly`
- `Unit: template with missing variable throws TemplateRenderError`
- `Integration (mocked Resend): sendNow calls Resend API and updates status to sent`
- `Integration (mocked Resend): scheduled message sent at correct time via BullMQ delay`
- `Integration: Resend webhook updates tracking.opened_at on email.opened event`
- `Integration: invalid webhook signature returns 401`
- `Integration: bounce event updates status to bounced and logs compliance event`

---

#### 4.2 — AI-Powered Outreach Personalization

**What**: LLM-generated personalized pitch emails that reference a creator's recent content, audience overlap with the brand, and collaboration fit.

**Design**:

AI personalization service (`packages/ai/src/personalization/outreach.ts`):
```typescript
export interface PersonalizationInput {
  creator: CreatorRecord;
  campaign: Campaign;
  brandContext: {
    name: string;
    industry: string;
    productDescription: string;
    targetAudience: string;
    previousCollaborations?: string[];
  };
  recentPosts?: Array<{
    platform: Platform;
    caption: string;
    postUrl: string;
    engagementRate: number;
  }>;
}

export interface PersonalizedOutreach {
  subject: string;
  body: string;                // HTML-formatted email
  aiContext: {
    model: string;
    referencedPosts: string[];
    brandFitScore: number;
    personalizationSignals: string[];
  };
}

export async function generatePersonalizedOutreach(
  input: PersonalizationInput
): Promise<PersonalizedOutreach>;
```

System prompt template:
```
You are an outreach specialist writing a partnership pitch email from {{brandContext.name}} to {{creator.fullName}}.

The creator's profile:
- Platform: {{creator.profiles | keys | join(', ')}}
- Total followers: {{creator.totalFollowers}}
- Categories: {{creator.categories | join(', ')}}
- Top audience: {{creator.audience.topCountry}} ({{creator.audience.topCountryPct}}%)

Recent content that aligns with the brand:
{{#each recentPosts}}
- {{this.platform}}: "{{this.caption | truncate(100)}}" ({{this.engagementRate}}% engagement) — {{this.postUrl}}
{{/each}}

Campaign details:
- Name: {{campaign.name}}
- Type: {{campaign.campaignType}}
- Deliverables: {{campaign.brief.deliverables | join(', ')}}

Write a concise, professional email that:
1. References specific recent content from the creator
2. Explains why the brand-creator fit is strong
3. Outlines the collaboration opportunity clearly
4. Includes a clear call to action
5. Keeps the tone authentic and personal, not corporate

Output JSON: { "subject": "...", "body": "...", "brandFitScore": 0-100, "personalizationSignals": ["...", "..."] }
```

API route:
```typescript
// POST /api/outreach/generate
// Body: PersonalizationInput
// Response: PersonalizedOutreach
// Auth: required (manager+)
// Rate limit: 20 requests/minute per organization (LLM cost control)
```

**Testing**:
- `Unit: prompt template renders all variables without Handlebars errors`
- `Unit: output JSON parsing handles valid Claude response`
- `Unit: brandFitScore is clamped to 0-100 range`
- `Integration (mocked LLM): generate returns personalized email referencing creator content`
- `Integration (mocked LLM): rate limit returns 429 after 20 requests in 1 minute`
- `Unit: personalizationSignals array is non-empty`
- `Integration: generated email stored with isAiGenerated=true and aiContext populated`

---

#### 4.3 — Creator CRM Features

**What**: Notes, tags, activity timeline, and relationship tracking for creator profiles within an organization's context.

**Design**:

Additional schema for CRM (added to `packages/db/src/schema.ts`):
```typescript
export const creatorNotes = pgTable('creator_notes', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  creatorId: uuid('creator_id').notNull().references(() => creators.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id),
  content: text('content').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const creatorTags = pgTable('creator_tags', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  creatorId: uuid('creator_id').notNull().references(() => creators.id, { onDelete: 'cascade' }),
  tag: varchar('tag', { length: 100 }).notNull(),
}, (table) => ({
  unique: uniqueIndex('idx_creator_tags_unique').on(table.organizationId, table.creatorId, table.tag),
}));
```

CRM API routes:
```typescript
// GET    /api/creators/:id/crm           — Get CRM data (notes, tags, activity)
// POST   /api/creators/:id/notes         — Add note
// PATCH  /api/creators/:id/notes/:noteId — Edit note
// DELETE /api/creators/:id/notes/:noteId — Delete note
// POST   /api/creators/:id/tags          — Add tag
// DELETE /api/creators/:id/tags/:tag     — Remove tag
// GET    /api/creators/:id/activity      — Timeline of all interactions
```

Activity timeline aggregation:
```typescript
export interface ActivityEvent {
  type: 'outreach_sent' | 'outreach_replied' | 'campaign_invited' | 'campaign_contracted'
    | 'content_published' | 'payment_processed' | 'note_added' | 'compliance_issue';
  timestamp: string;
  description: string;
  metadata: Record<string, unknown>;
}

// GET /api/creators/:id/activity returns ActivityEvent[] sorted by timestamp desc
// Sources: outreach_messages, campaign_creators, content_posts, payments, compliance_log, creator_notes
```

**Testing**:
- `Unit: add note stores content with userId and organizationId`
- `Unit: notes are organization-scoped — org A cannot see org B notes on same creator`
- `Unit: add duplicate tag (same org + creator + tag) returns 409`
- `Unit: activity timeline merges events from all sources in chronological order`
- `Integration: delete note removes it from activity timeline`
- `Integration: inviting creator to campaign appears in activity timeline`
- `Integration: CRM data endpoint returns notes, tags, and recent activity in single response`

---

## Phase 5: Content Tracking and Compliance

### Purpose

Build the content monitoring system that tracks published creator posts, syncs engagement metrics, and runs automated compliance checks for FTC disclosure requirements. After this phase, the platform can detect missing #ad/#sponsored disclosures, flag non-compliant content, and maintain audit trails for regulatory review.

### Tasks

#### 5.1 — Content Post Tracking

**What**: Register and track content posts created by campaign creators, with periodic metric syncing from social platform APIs.

**Design**:

Content service:
```typescript
export interface RegisterContentInput {
  campaignCreatorId: string;
  platform: Platform;
  platformPostId?: string;
  postUrl: string;
  postType: 'image' | 'video' | 'reel' | 'story' | 'carousel' | 'article';
  caption?: string;
  publishedAt?: string;
}

export class ContentService {
  async register(input: RegisterContentInput): Promise<ContentPost>;
  async syncMetrics(contentPostId: string): Promise<ContentPost>;
  async batchSyncMetrics(campaignId: string): Promise<void>; // syncs all posts in a campaign
  async listByCampaign(campaignId: string): Promise<ContentPost[]>;
  async listByCreator(campaignCreatorId: string): Promise<ContentPost[]>;
}
```

Metric sync job (`apps/worker/src/jobs/sync-content-metrics.ts`):
```typescript
// Job: sync-content-metrics
// Runs on schedule (every 6 hours for active campaigns) or on-demand
// For each content post:
// 1. Call platform API (Instagram Graph API / TikTok / YouTube Data API)
// 2. Extract platform-specific metrics into JSONB
// 3. Update content_posts.metrics
// 4. Recalculate campaign_creators.performance rollup
// 5. Recalculate campaigns.metrics rollup
```

**Testing**:
- `Unit: register content post validates postUrl format per platform`
- `Unit: register content post extracts platformPostId from URL if not provided`
- `Integration (mocked API): syncMetrics updates metrics JSONB with Instagram response fields`
- `Integration (mocked API): syncMetrics for TikTok stores TikTok-specific fields (completion_rate)`
- `Integration: batchSyncMetrics processes all posts for a campaign`
- `Integration: campaign metrics rollup sums correctly after content sync`
- `Unit: metric sync handles API rate limit (429) with exponential backoff`

---

#### 5.2 — FTC Compliance Checking

**What**: Automated compliance analysis of content posts for FTC disclosure requirements (16 CFR Part 255), ASA CAP Code, and platform-specific branded content policies.

**Design**:

Compliance checker (`packages/ai/src/compliance/disclosure-check.ts`):
```typescript
export interface ComplianceCheckResult {
  hasDisclosure: boolean;
  disclosureTypes: ('hashtag_ad' | 'hashtag_sponsored' | 'platform_tag' | 'caption_text')[];
  status: 'compliant' | 'non_compliant' | 'review_required';
  checks: Array<{
    type: 'ftc_disclosure' | 'asa_disclosure' | 'platform_tag' | 'brand_safety';
    result: 'pass' | 'fail' | 'warning';
    explanation: string;
    regulation: string;       // e.g. 'ftc_16cfr255', 'asa_cap_s2'
  }>;
  recommendedAction?: string;
}

export async function checkCompliance(
  post: ContentPost,
  campaign: Campaign
): Promise<ComplianceCheckResult>;
```

Rule-based check (fast path — no LLM needed):
```typescript
function checkDisclosureTags(caption: string, platform: Platform): ComplianceCheckResult {
  const ftcTags = ['#ad', '#sponsored', '#paidpartnership', '#advertisement'];
  const captionLower = caption.toLowerCase();

  const foundTags = ftcTags.filter(tag => captionLower.includes(tag));
  const hasHashtag = foundTags.length > 0;

  // FTC requires disclosure "above the fold" — check if tag is in first 125 characters
  const isAboveFold = ftcTags.some(tag => captionLower.indexOf(tag) < 125);

  // Build result...
}
```

LLM-enhanced check (for ambiguous cases):
```typescript
// Uses Claude to analyze:
// 1. Is the caption language clearly disclosing a paid partnership?
// 2. Are there deceptive claims that violate FTC guidelines?
// 3. Is the disclosure conspicuous (not buried in hashtag spam)?
// Triggered when rule-based check returns 'review_required'
```

Compliance job (`apps/worker/src/jobs/run-compliance-check.ts`):
```typescript
// Job: run-compliance-check
// Triggered when content is registered or metrics are synced
// 1. Run rule-based disclosure check
// 2. If result is review_required, run LLM-enhanced check
// 3. Update content_posts.compliance JSONB
// 4. Insert compliance_log entry
// 5. If non_compliant, enqueue notification to campaign manager
```

API routes:
```typescript
// GET  /api/compliance/dashboard         — Org-wide compliance summary
// GET  /api/compliance/campaign/:id      — Campaign compliance issues
// GET  /api/compliance/post/:id/checks   — All checks for a specific post
// POST /api/compliance/post/:id/recheck  — Trigger manual recheck
```

**Testing**:
- `Unit: caption with #ad in first 50 chars returns compliant`
- `Unit: caption with #ad after character 200 returns warning (not above fold)`
- `Unit: caption with no disclosure tags returns non_compliant`
- `Unit: caption with platform tag (Paid Partnership) but no hashtag returns compliant`
- `Unit: compliance check includes FTC 16 CFR Part 255 as regulation reference`
- `Integration (mocked LLM): ambiguous caption triggers LLM check and returns determination`
- `Integration: compliance failure creates compliance_log entry with severity=violation`
- `Integration: recheck updates compliance status and adds new check to history`
- `Fixture-based: 20 sample captions with known compliance status all classified correctly`

---

#### 5.3 — Compliance Audit Trail and Reporting

**What**: Queryable compliance log with filtering, export capability, and per-regulation summary reporting for FTC/ASA review preparation.

**Design**:

```typescript
export interface ComplianceReport {
  period: { start: string; end: string };
  totalPosts: number;
  compliantPosts: number;
  nonCompliantPosts: number;
  reviewRequired: number;
  byRegulation: Record<string, { pass: number; fail: number; warning: number }>;
  byCampaign: Array<{
    campaignId: string;
    campaignName: string;
    totalPosts: number;
    issues: number;
  }>;
  recentViolations: Array<{
    postUrl: string;
    creatorName: string;
    campaignName: string;
    checkType: string;
    explanation: string;
    detectedAt: string;
  }>;
}

// GET /api/compliance/report?start=2026-01-01&end=2026-06-01
// Response: ComplianceReport
```

**Testing**:
- `Unit: report aggregates correctly across multiple campaigns`
- `Unit: byRegulation groups FTC and ASA checks separately`
- `Unit: recentViolations sorted by detectedAt descending, limited to 50`
- `Integration: compliance report reflects real-time data after new checks are run`
- `Integration: report with no violations returns zero counts, not errors`

---

## Phase 6: Payments and Affiliate Tracking

### Purpose

Integrate Stripe Connect for creator payments, build affiliate link generation and tracking, and connect e-commerce conversion data from Shopify. After this phase, brands can pay creators directly through the platform, generate trackable affiliate links and discount codes, and attribute sales to specific creators.

### Tasks

#### 6.1 — Stripe Connect Integration

**What**: Onboard creators as Stripe Connect Express accounts, process payments (flat fees, commissions), and handle payouts.

**Design**:

Payment service:
```typescript
export interface CreatePaymentInput {
  campaignCreatorId: string;
  amount: number;
  currency: string;           // ISO 4217
  paymentType: 'flat_fee' | 'commission' | 'bonus';
}

export class PaymentService {
  async onboardCreator(creatorId: string): Promise<{ onboardingUrl: string }>;
    // Creates Stripe Connect Express account, returns onboarding link
  async checkOnboardingStatus(creatorId: string): Promise<'pending' | 'complete' | 'restricted'>;
  async createPayment(orgId: string, input: CreatePaymentInput): Promise<Payment>;
  async approvePayment(paymentId: string): Promise<Payment>;
  async processPayment(paymentId: string): Promise<Payment>;
    // Creates Stripe Transfer from platform to creator's connected account
  async listPayments(orgId: string, filters: PaymentFilters): Promise<PaginatedResult<Payment>>;
  async handleStripeWebhook(event: Stripe.Event): Promise<void>;
}
```

Stripe webhook handler:
```typescript
// POST /api/webhooks/stripe
// Events handled:
// - account.updated: Update creator onboarding status
// - transfer.created: Update payment status to processing
// - transfer.paid: Update payment status to completed, set paid_at
// - transfer.failed: Update payment status to failed, log error
```

API routes:
```typescript
// POST   /api/payments                    — Create payment
// POST   /api/payments/:id/approve        — Approve payment (manager+)
// POST   /api/payments/:id/process        — Process payment via Stripe (admin+)
// GET    /api/payments                    — List payments (org-scoped)
// GET    /api/payments/:id               — Payment detail
// POST   /api/creators/:id/stripe/onboard — Start Stripe onboarding
// GET    /api/creators/:id/stripe/status  — Check onboarding status
```

**Testing**:
- `Unit: createPayment validates ISO 4217 currency code`
- `Unit: createPayment rejects negative amounts`
- `Unit: payment status transition pending -> approved -> processing -> completed`
- `Unit: processPayment fails if creator not onboarded to Stripe`
- `Integration (mocked Stripe): onboardCreator creates Express account and returns URL`
- `Integration (mocked Stripe): processPayment creates Stripe Transfer with correct amount`
- `Integration: Stripe webhook transfer.paid updates payment status and budgetSpent`
- `Integration: invalid Stripe webhook signature returns 400`

---

#### 6.2 — Affiliate Link and Discount Code Generation

**What**: Generate trackable affiliate links and discount codes per campaign creator, with click tracking and conversion attribution.

**Design**:

Affiliate service:
```typescript
export interface CreateAffiliateLinkInput {
  campaignCreatorId: string;
  linkType: 'url' | 'discount_code';
  destinationUrl?: string;      // required for url type
  discountCode?: string;        // auto-generated if not provided
  expiresAt?: string;
}

export class AffiliateService {
  async createLink(input: CreateAffiliateLinkInput): Promise<AffiliateLink>;
  async recordClick(linkId: string, metadata: ClickMetadata): Promise<void>;
  async recordConversion(linkId: string, conversion: ConversionData): Promise<void>;
  async getStats(linkId: string): Promise<AffiliateStats>;
  async listByCampaign(campaignId: string): Promise<AffiliateLink[]>;
}
```

Click tracking endpoint (public, no auth):
```typescript
// GET /r/:trackingCode
// Increments click count, records referrer/user-agent, redirects to destinationUrl
// Response: 302 redirect
```

Discount code generation:
```typescript
function generateDiscountCode(
  brandSlug: string,
  creatorUsername: string
): string {
  // Format: BRAND_CREATOR_XXXX (e.g., GLOW_JANEDOE_A7K2)
  const suffix = nanoid(4).toUpperCase();
  return `${brandSlug}_${creatorUsername}_${suffix}`.toUpperCase().slice(0, 20);
}
```

**Testing**:
- `Unit: generateDiscountCode produces codes matching expected format`
- `Unit: generated codes are unique across 1000 generations`
- `Unit: click tracking increments counter and records metadata`
- `Integration: redirect endpoint returns 302 with correct destination URL`
- `Integration: expired link returns 410 Gone`
- `Integration: conversion recording updates stats.revenue and stats.conversions`
- `Integration: affiliate stats per campaign aggregates across all links`

---

#### 6.3 — Shopify E-commerce Integration

**What**: Connect to Shopify stores via OAuth, receive order webhooks, and attribute conversions to affiliate links and discount codes.

**Design**:

Shopify integration service:
```typescript
export class ShopifyIntegrationService {
  async initiateOAuth(orgId: string, shopUrl: string): Promise<{ authUrl: string }>;
  async handleCallback(orgId: string, code: string, shop: string): Promise<void>;
  async handleOrderWebhook(orgId: string, order: ShopifyOrder): Promise<void>;
    // 1. Check if order used a tracked discount code
    // 2. Check if order came via a tracked affiliate URL (utm_source)
    // 3. If matched, record conversion against affiliate_links
    // 4. Update campaign_creators.performance rollup
    // 5. If campaign has commission-based creators, calculate commission amount
}
```

Webhook routes:
```typescript
// POST /api/webhooks/shopify/:orgId
// Events: orders/create, orders/paid
// Validates HMAC signature
// Matches discount code or UTM params to affiliate links
```

**Testing**:
- `Unit: order with tracked discount code creates conversion record`
- `Unit: order with tracked UTM source creates conversion record`
- `Unit: order with no matching affiliate data is ignored (no error)`
- `Unit: commission calculation applies correct percentage from campaign_creators`
- `Integration (mocked Shopify): OAuth flow stores access token in integration_configs`
- `Integration: webhook with valid HMAC processes order`
- `Integration: webhook with invalid HMAC returns 401`
- `Integration: conversion attribution updates campaign budget_spent`

---

## Phase 7: Fraud Detection and AI Scoring

### Purpose

Build ML-powered creator scoring for fraud detection, audience authenticity, brand safety, and content quality. After this phase, the platform can identify fake followers, detect engagement manipulation, flag synthetic (AI-generated) influencers, and score creators on brand fit. These are key differentiators from incumbent platforms that rely on heuristic-based approaches.

### Tasks

#### 7.1 — Fraud Detection Scoring

**What**: Multi-signal fraud analysis that evaluates follower authenticity, engagement consistency, and growth patterns to produce a fraud score (0-100).

**Design**:

Fraud scoring pipeline (`packages/ai/src/scoring/fraud.ts`):
```typescript
export interface FraudSignals {
  followerGrowthAnomaly: number;      // sudden spikes vs. organic growth (0-100)
  engagementToFollowerRatio: number;  // normal is 1-5% for most tiers
  commentQualityScore: number;        // generic vs. substantive comments (0-100)
  followerDemographicConsistency: number; // expected vs. actual demographics (0-100)
  followingToFollowerRatio: number;   // high ratio suggests follow/unfollow spam
  postingConsistency: number;         // regular posting vs. burst patterns (0-100)
  likeToCommentRatio: number;         // abnormal ratios suggest purchased engagement
}

export interface FraudScore {
  score: number;             // 0 (clean) to 100 (high fraud risk)
  signals: FraudSignals;
  riskLevel: 'low' | 'medium' | 'high' | 'critical';
  explanation: string;       // human-readable summary
  scoredAt: string;
}

export async function calculateFraudScore(
  creator: CreatorRecord,
  historicalData?: CreatorRecord[] // previous snapshots for trend analysis
): Promise<FraudScore>;
```

Risk level thresholds:
```typescript
function getRiskLevel(score: number): FraudScore['riskLevel'] {
  if (score >= 80) return 'critical';
  if (score >= 60) return 'high';
  if (score >= 30) return 'medium';
  return 'low';
}
```

BullMQ job:
```typescript
// Job: calculate-ai-scores
// Runs after creator profile sync
// 1. Calculate fraud score
// 2. Calculate authenticity score
// 3. Update creators.fraudScore, creators.authenticityScore
// 4. Update creators.aiScores JSONB with full signal breakdown
// 5. Update Elasticsearch document
```

**Testing**:
- `Unit: creator with sudden 500% follower growth in 7 days scores high on followerGrowthAnomaly`
- `Unit: creator with 0.01% engagement rate on 1M followers scores high on engagementToFollowerRatio`
- `Unit: creator with consistent growth and normal engagement scores low (<20)`
- `Unit: getRiskLevel returns correct levels at boundaries (29->low, 30->medium, etc.)`
- `Fixture-based: 10 known-fraud creator profiles all score above 60`
- `Fixture-based: 10 known-authentic creator profiles all score below 30`
- `Integration: fraud score updates propagate to Elasticsearch index`

---

#### 7.2 — Synthetic Influencer Detection

**What**: Detect AI-generated or virtual influencers to help brands comply with FTC 2026 guidance on synthetic creators.

**Design**:

```typescript
export interface SyntheticDetectionResult {
  isSynthetic: boolean;
  confidence: number;          // 0-100
  signals: {
    profileImageAnalysis: 'human' | 'ai_generated' | 'avatar' | 'unknown';
    contentPatternAnalysis: 'organic' | 'synthetic' | 'unknown';
    bioLanguageAnalysis: 'natural' | 'templated' | 'unknown';
    accountHistoryAnalysis: 'established' | 'new_sudden_growth' | 'unknown';
  };
  explanation: string;
}

export async function detectSynthetic(
  creator: CreatorRecord,
  recentPosts?: ContentPost[]
): Promise<SyntheticDetectionResult>;
```

Detection approach:
1. Profile image analysis — check for AI-generated face artifacts (via Claude vision API)
2. Content pattern analysis — detect repetitive posting patterns, unnatural consistency
3. Bio language analysis — identify templated or machine-generated bios
4. Account history — sudden appearance with immediate high follower count

**Testing**:
- `Unit: known virtual influencer profile (e.g., Lil Miquela pattern) detected as synthetic`
- `Unit: known human influencer profile detected as not synthetic`
- `Unit: confidence score reflects number of positive signals`
- `Integration (mocked LLM): vision API analysis of AI-generated face returns ai_generated`
- `Integration: isSynthetic flag updates on creators table and Elasticsearch`

---

#### 7.3 — Brand Safety and Content Quality Scoring

**What**: AI-driven scoring of creator content for brand safety (controversial topics, offensive content) and content quality (writing clarity, visual aesthetics, audience alignment).

**Design**:

```typescript
export interface AiScores {
  brandSafety: number;          // 0-100 (100 = safest)
  contentQuality: number;       // 0-100
  audiencePurchasingIntent: number; // 0-100
  fatigueRisk: number;         // 0-100 (100 = highest fatigue risk)
  nicheAuthority: Record<string, number>; // { "beauty": 88, "fashion": 72 }
  sentiment: { positive: number; neutral: number; negative: number };
  scoredAt: string;
}

export async function calculateAiScores(
  creator: CreatorRecord,
  recentPosts: ContentPost[]
): Promise<AiScores>;
```

LLM prompt for content quality analysis:
```
Analyze the following creator content for brand partnership suitability.

Creator: {{creator.fullName}} ({{creator.tier}} tier, categories: {{creator.categories}})

Recent posts:
{{#each recentPosts}}
Post {{@index}}: {{this.caption | truncate(300)}}
Platform: {{this.platform}}, Engagement: {{this.metrics.engagement_rate}}%
{{/each}}

Score the following dimensions (0-100):
1. Brand Safety: Does the content avoid controversial, offensive, or risky topics?
2. Content Quality: Is the writing clear, engaging, and professional?
3. Audience Purchasing Intent: Does the audience engage in ways suggesting purchase intent?
4. Fatigue Risk: Is the creator showing signs of over-commercialization?
5. Niche Authority: How authoritative is the creator in each of their categories?
6. Sentiment: What percentage of engagement is positive/neutral/negative?

Output JSON: { "brandSafety": N, "contentQuality": N, ... }
```

**Testing**:
- `Unit: creator with recent controversial content scores low on brand safety`
- `Unit: creator with 5 consecutive sponsored posts scores high on fatigue risk`
- `Unit: niche authority scores are returned per category`
- `Integration (mocked LLM): scoring pipeline processes creator and stores results`
- `Integration: AI scores visible in creator search results and profile page`

---

## Phase 8: Frontend Dashboard

### Purpose

Build the Next.js web application providing the user-facing dashboard for all platform functionality — creator discovery, campaign management, outreach, content tracking, compliance, and payments. After this phase, the platform is fully usable through a browser interface.

### Tasks

#### 8.1 — Layout, Navigation, and Authentication UI

**What**: App shell with sidebar navigation, authentication pages (login, signup), and organization settings.

**Design**:

Layout structure:
```typescript
// app/(auth)/login/page.tsx — Login form
// app/(auth)/signup/page.tsx — Signup form (creates org + user)
// app/(dashboard)/layout.tsx — Sidebar + top bar + content area

interface SidebarNavItem {
  label: string;
  href: string;
  icon: LucideIcon;
  badge?: number;              // e.g., compliance issues count
}

const navItems: SidebarNavItem[] = [
  { label: 'Discovery', href: '/creators', icon: Search },
  { label: 'Campaigns', href: '/campaigns', icon: Megaphone },
  { label: 'Outreach', href: '/outreach', icon: Mail },
  { label: 'Content', href: '/content', icon: FileText },
  { label: 'Compliance', href: '/compliance', icon: Shield },
  { label: 'Payments', href: '/payments', icon: CreditCard },
  { label: 'Integrations', href: '/integrations', icon: Plug },
  { label: 'Settings', href: '/settings', icon: Settings },
];
```

API client (`apps/web/src/lib/api-client.ts`):
```typescript
export class ApiClient {
  constructor(private baseUrl: string) {}

  async get<T>(path: string, params?: Record<string, string>): Promise<T>;
  async post<T>(path: string, body: unknown): Promise<T>;
  async patch<T>(path: string, body: unknown): Promise<T>;
  async delete(path: string): Promise<void>;
}
```

**Testing**:
- `E2E: signup flow creates account and redirects to dashboard`
- `E2E: login with valid credentials reaches dashboard`
- `E2E: login with invalid credentials shows error message`
- `E2E: unauthenticated user redirected to /login`
- `E2E: sidebar navigation links navigate to correct pages`
- `E2E: mobile viewport shows hamburger menu`

---

#### 8.2 — Creator Discovery UI

**What**: Search interface with filters panel, creator cards grid, and detailed creator profile view.

**Design**:

Discovery page components:
```typescript
// components/creators/CreatorSearchFilters.tsx
// - Platform selector (multi-select checkboxes)
// - Country dropdown (ISO 3166-1 countries)
// - Follower range slider
// - Engagement rate range slider
// - Fraud score max slider
// - Category tags selector
// - Audience demographics filters (gender %, age range %)
// - Sort dropdown
// - "Exclude synthetic" toggle (default: on)

// components/creators/CreatorCard.tsx
// - Avatar, name, country flag
// - Platform badges with follower counts
// - Engagement rate (colored: green >3%, yellow 1-3%, red <1%)
// - Fraud score indicator (green/yellow/red)
// - Category tags
// - "View Profile" / "Add to Campaign" actions

// components/creators/CreatorProfile.tsx
// - Full profile with all platform stats
// - Audience demographics charts (age distribution, gender split, geography map)
// - AI scores (brand safety, content quality, fatigue risk)
// - Activity timeline
// - CRM notes and tags
// - Campaign history
```

**Testing**:
- `E2E: search page loads with default results`
- `E2E: applying platform filter updates results`
- `E2E: clicking creator card opens profile with full details`
- `E2E: audience demographics chart renders correctly`
- `E2E: "Add to Campaign" opens campaign selector modal`
- `E2E: search with no results shows empty state message`

---

#### 8.3 — Campaign Management UI

**What**: Campaign list, creation wizard, detail view with creator management and brief editor.

**Design**:

Campaign creation wizard steps:
```typescript
// Step 1: Basic info (name, type, objective, dates)
// Step 2: Budget (total, currency)
// Step 3: Target platforms selection
// Step 4: Brief editor (markdown with preview, required hashtags, dos/donts)
// Step 5: Review and create

// components/campaigns/CampaignList.tsx — filterable list with status badges
// components/campaigns/CampaignDetail.tsx — tabbed view:
//   - Overview (dashboard metrics)
//   - Creators (invited/contracted list)
//   - Content (tracked posts grid)
//   - Compliance (issues list)
//   - Payments (payment history)
//   - Brief (view/edit)
```

**Testing**:
- `E2E: campaign creation wizard completes all steps and creates campaign`
- `E2E: campaign list filters by status`
- `E2E: campaign detail shows correct metric totals`
- `E2E: inviting creator from campaign detail updates creator list`
- `E2E: campaign status transitions via UI buttons`

---

#### 8.4 — Outreach, Content, Compliance, and Payments UI

**What**: Remaining dashboard pages for outreach management, content tracking, compliance dashboard, and payment management.

**Design**:

Key pages:
```typescript
// Outreach
// - /outreach — message list with status filters
// - /outreach/compose — compose with template selector or AI generation
// - /outreach/templates — manage email templates

// Content
// - /content — grid of tracked posts with metrics and compliance badges
// - /content/:id — post detail with metrics chart and compliance check history

// Compliance
// - /compliance — dashboard with violation count, compliance rate chart
// - /compliance/report — generate and export compliance report

// Payments
// - /payments — payment list with status filters and totals
// - /payments/create — create payment (select campaign creator, enter amount)
```

**Testing**:
- `E2E: compose outreach with AI generation produces personalized email`
- `E2E: content grid shows compliance badges (green check / red warning)`
- `E2E: compliance dashboard shows correct violation count`
- `E2E: payment creation flow selects campaign creator and processes payment`
- `E2E: payment list shows correct status badges and totals`

---

## Phase 9: Platform Integrations

### Purpose

Connect to social platform APIs (Instagram Graph API, TikTok for Developers, YouTube Data API) for direct data access, and implement the integration management UI. After this phase, organizations can authenticate their social platform accounts, and the platform can pull data directly from official APIs where available.

### Tasks

#### 9.1 — Instagram Graph API Integration

**What**: OAuth 2.0 Business Login flow for Instagram, fetch creator insights and content metrics.

**Design**:

```typescript
export class InstagramIntegration {
  async initiateOAuth(orgId: string): Promise<{ authUrl: string }>;
    // OAuth 2.0 authorization code flow
    // Scopes: instagram_basic, instagram_manage_insights, pages_read_engagement
  async handleCallback(orgId: string, code: string): Promise<void>;
    // Exchange code for access token, store in integration_configs
  async getCreatorInsights(accessToken: string, userId: string): Promise<PlatformProfile>;
    // GET /{user-id}?fields=followers_count,media_count,...
  async getContentMetrics(accessToken: string, mediaId: string): Promise<ContentMetrics>;
    // GET /{media-id}/insights?metric=impressions,reach,likes,...
  async refreshToken(orgId: string): Promise<void>;
    // Exchange short-lived token for long-lived token (60-day expiry)
}
```

Rate limiting:
```typescript
// Instagram Graph API: 200 calls per hour per user token
// Implement token bucket rate limiter per integration
const instagramRateLimiter = new TokenBucket({
  maxTokens: 200,
  refillRate: 200,
  refillInterval: 3600000, // 1 hour
});
```

**Testing**:
- `Unit: OAuth URL includes correct scopes and redirect URI`
- `Unit: token refresh stores new token with updated expiry`
- `Integration (mocked API): getCreatorInsights maps response to PlatformProfile`
- `Integration (mocked API): rate limiter delays requests when tokens exhausted`
- `Integration: expired token triggers automatic refresh before API call`

---

#### 9.2 — TikTok and YouTube Integrations

**What**: Implement OAuth and data fetching for TikTok for Developers API and YouTube Data API v3.

**Design**:

```typescript
export class TikTokIntegration {
  async initiateOAuth(orgId: string): Promise<{ authUrl: string }>;
  async handleCallback(orgId: string, code: string): Promise<void>;
  async getCreatorInsights(accessToken: string, userId: string): Promise<PlatformProfile>;
    // TikTok Creator Search Insights API (2026) — no creator OAuth needed for discovery
  async getContentMetrics(accessToken: string, videoId: string): Promise<ContentMetrics>;
}

export class YouTubeIntegration {
  async initiateOAuth(orgId: string): Promise<{ authUrl: string }>;
  async handleCallback(orgId: string, code: string): Promise<void>;
  async getChannelStats(accessToken: string, channelId: string): Promise<PlatformProfile>;
    // YouTube Data API v3: channels.list with statistics
  async getVideoMetrics(accessToken: string, videoId: string): Promise<ContentMetrics>;
    // YouTube Analytics API: reports.query
  // Rate limit: 10,000 units/day per Google Cloud project
}
```

**Testing**:
- `Unit: TikTok Creator Search Insights API returns demographics without creator OAuth`
- `Unit: YouTube quota tracking prevents exceeding 10,000 units/day`
- `Integration (mocked API): TikTok OAuth flow stores credentials`
- `Integration (mocked API): YouTube channel stats mapped to PlatformProfile format`
- `Integration: platform-specific metric fields stored correctly in JSONB`

---

#### 9.3 — Integration Management UI

**What**: Settings page for connecting and managing social platform and e-commerce integrations.

**Design**:

```typescript
// /integrations page:
// - Card per platform (Instagram, TikTok, YouTube, Shopify)
// - Each card shows: connection status, last sync time, connected account
// - "Connect" button initiates OAuth flow
// - "Disconnect" button revokes token and removes integration
// - "Sync Now" button triggers manual data sync
// - Sync history with success/failure log
```

**Testing**:
- `E2E: clicking Connect for Instagram redirects to Meta OAuth page`
- `E2E: after OAuth callback, integration shows as connected`
- `E2E: Disconnect removes integration and shows disconnected state`
- `E2E: Sync Now triggers sync job and shows progress`

---

## Phase 10: Predictive Analytics and Dynamic Budget Optimization

### Purpose

Build ML-powered predictive ROI modeling and dynamic budget reallocation — two features identified in research as underserved opportunities in the competitive landscape. After this phase, brands can estimate campaign ROI before launch and automatically shift budget toward top-performing creators mid-campaign.

### Tasks

#### 10.1 — Predictive ROI Modeling

**What**: Estimate expected reach, engagement, and conversion for a campaign based on historical performance data from similar campaigns.

**Design**:

```typescript
export interface RoiPrediction {
  expectedImpressions: { low: number; mid: number; high: number };
  expectedEngagement: { low: number; mid: number; high: number };
  expectedConversions: { low: number; mid: number; high: number };
  expectedRevenue: { low: number; mid: number; high: number };
  expectedRoi: { low: number; mid: number; high: number };
  confidence: number;           // 0-100
  comparableCampaigns: number;  // how many similar campaigns informed the prediction
  methodology: string;          // human-readable explanation
}

export async function predictCampaignRoi(
  campaign: Campaign,
  campaignCreators: CampaignCreator[]
): Promise<RoiPrediction>;
```

Prediction methodology:
1. Find comparable campaigns (same type, similar budget, overlapping platforms, similar creator tiers)
2. Calculate per-creator expected performance from historical data
3. Aggregate with confidence intervals (low = 25th percentile, mid = median, high = 75th percentile)
4. Apply brand-specific adjustments if the brand has previous campaign history

API route:
```typescript
// POST /api/campaigns/:id/predict-roi
// Response: RoiPrediction
// Auth: required (manager+)
```

**Testing**:
- `Unit: prediction with zero comparable campaigns returns confidence=0 with fallback estimates`
- `Unit: prediction with 50+ comparable campaigns returns confidence>70`
- `Unit: low/mid/high intervals are ordered correctly (low < mid < high)`
- `Fixture-based: prediction for fixture campaign matches expected ranges`
- `Integration: prediction endpoint returns result within 5 seconds`

---

#### 10.2 — Dynamic Budget Reallocation

**What**: Automated system that monitors creator performance during active campaigns and reallocates budget from underperformers to over-performers.

**Design**:

```typescript
export interface BudgetReallocation {
  campaignId: string;
  recommendations: Array<{
    campaignCreatorId: string;
    creatorName: string;
    currentBudget: number;
    recommendedBudget: number;
    change: number;
    changePercent: number;
    reason: string;
    performanceMetrics: {
      roi: number;
      engagementRate: number;
      conversionRate: number;
    };
  }>;
  totalBudget: number;
  projectedRoiImprovement: number;  // percentage
  autoApproved: boolean;
}

export class BudgetOptimizer {
  async analyze(campaignId: string): Promise<BudgetReallocation>;
  async apply(campaignId: string, reallocation: BudgetReallocation): Promise<void>;
  async enableAutoOptimization(campaignId: string, config: AutoOptConfig): Promise<void>;
}

export interface AutoOptConfig {
  enabled: boolean;
  checkInterval: '6h' | '12h' | '24h';
  maxReallocationPct: number;     // max % of budget to shift per check (default 20)
  minPerformanceDelta: number;    // min ROI difference to trigger reallocation
  requireApproval: boolean;       // if true, creates recommendation; if false, auto-applies
}
```

BullMQ scheduled job:
```typescript
// Job: optimize-campaign-budget
// Runs at configured interval for campaigns with auto-optimization enabled
// 1. Fetch current creator performance metrics
// 2. Calculate relative ROI per creator
// 3. Identify underperformers (below campaign average ROI)
// 4. Generate reallocation recommendation
// 5. If requireApproval=false and change within maxReallocationPct, auto-apply
// 6. Otherwise, store recommendation and notify campaign manager
```

**Testing**:
- `Unit: reallocation shifts budget from lowest-ROI to highest-ROI creator`
- `Unit: reallocation respects maxReallocationPct limit`
- `Unit: campaign with single creator produces no reallocation`
- `Unit: all creators performing equally produces no reallocation`
- `Integration: auto-optimization job runs at configured interval`
- `Integration: applied reallocation updates campaign_creators.agreedRate`
- `Integration: reallocation creates compliance_log entry for audit`

---

## Phase 11: GDPR/CCPA Compliance and Data Governance

### Purpose

Implement data privacy features required by GDPR (EU) 2016/679 and CCPA (Cal. Civ. Code 1798.100) — consent management, data subject access requests, data deletion workflows, and data retention policies. After this phase, the platform meets regulatory requirements for handling creator personal data in EU and California jurisdictions.

### Tasks

#### 11.1 — Consent Management

**What**: Track, store, and enforce consent for data processing, marketing contact, audience analysis, and data sharing on a per-creator basis.

**Design**:

Additional schema:
```typescript
export const consentRecords = pgTable('consent_records', {
  id: uuid('id').primaryKey().defaultRandom(),
  creatorId: uuid('creator_id').notNull().references(() => creators.id, { onDelete: 'cascade' }),
  consentType: varchar('consent_type', { length: 50 }).notNull(),
    // data_processing, marketing_contact, audience_analysis, data_sharing
  granted: boolean('granted').notNull(),
  ipAddress: varchar('ip_address', { length: 45 }),
  userAgent: text('user_agent'),
  grantedAt: timestamp('granted_at', { withTimezone: true }),
  revokedAt: timestamp('revoked_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Consent service:
```typescript
export class ConsentService {
  async recordConsent(creatorId: string, type: ConsentType, granted: boolean, meta: ConsentMeta): Promise<void>;
  async revokeConsent(creatorId: string, type: ConsentType): Promise<void>;
  async getConsents(creatorId: string): Promise<ConsentRecord[]>;
  async hasConsent(creatorId: string, type: ConsentType): Promise<boolean>;
  async enforceConsent(creatorId: string, type: ConsentType): Promise<void>;
    // Throws ConsentRequiredError if consent not granted
}
```

**Testing**:
- `Unit: recordConsent stores IP address and user agent`
- `Unit: revokeConsent sets revokedAt timestamp without deleting record`
- `Unit: hasConsent returns false after revocation`
- `Unit: enforceConsent throws ConsentRequiredError for missing consent`
- `Integration: audience analysis API call blocked if audience_analysis consent not granted`

---

#### 11.2 — Data Subject Access and Deletion Requests

**What**: API and workflows for handling GDPR Article 15 (access requests) and Article 17 (right to erasure), and CCPA deletion requests.

**Design**:

```typescript
export interface DataAccessRequest {
  creatorId: string;
  regulation: 'gdpr' | 'ccpa';
  requestedData: 'all' | 'profile' | 'communications' | 'analytics';
}

export interface DataDeletionRequest {
  creatorId: string;
  regulation: 'gdpr' | 'ccpa';
  requesterEmail: string;
}

export class DataGovernanceService {
  async handleAccessRequest(req: DataAccessRequest): Promise<{ downloadUrl: string }>;
    // Compiles all data about creator into JSON export
    // Returns signed URL for download (expires in 48 hours)
    // GDPR: must respond within 30 days

  async handleDeletionRequest(req: DataDeletionRequest): Promise<{ requestId: string }>;
    // 1. Create data_deletion_requests record
    // 2. Queue deletion job
    // 3. Job removes: creator PII, outreach messages, consent records
    // 4. Retains: anonymized campaign performance data (legitimate interest)
    // 5. Removes from Elasticsearch index
    // CCPA: must complete within 45 days

  async listDeletionRequests(filters: { status?: string }): Promise<DeletionRequest[]>;
}
```

Data deletion schema:
```typescript
export const dataDeletionRequests = pgTable('data_deletion_requests', {
  id: uuid('id').primaryKey().defaultRandom(),
  creatorId: uuid('creator_id').references(() => creators.id),
  requesterEmail: varchar('requester_email', { length: 255 }).notNull(),
  regulation: varchar('regulation', { length: 10 }).notNull(),
  status: varchar('status', { length: 20 }).notNull().default('pending'),
    // pending, in_progress, completed, rejected
  requestedAt: timestamp('requested_at', { withTimezone: true }).notNull().defaultNow(),
  completedAt: timestamp('completed_at', { withTimezone: true }),
  completedBy: uuid('completed_by').references(() => users.id),
  notes: text('notes'),
});
```

**Testing**:
- `Unit: access request compiles all creator data into JSON export`
- `Unit: deletion request anonymizes PII but retains aggregated campaign data`
- `Unit: deletion removes creator from Elasticsearch index`
- `Unit: deletion revokes all consent records`
- `Integration: full deletion workflow completes within CCPA 45-day requirement`
- `Integration: deleted creator not returned in search results`
- `Integration: deleted creator's campaign performance data retained in anonymized form`

---

## Phase 12: Production Hardening and Deployment

### Purpose

Prepare the platform for production deployment — Docker images, CI/CD pipeline, monitoring, logging, security hardening, and performance optimization. After this phase, the platform can be deployed to a production environment and operated reliably.

### Tasks

#### 12.1 — Docker Build and Deployment Configuration

**What**: Multi-stage Docker builds for API, worker, and web apps, plus production docker-compose and Kubernetes manifests.

**Design**:

```dockerfile
# Dockerfile.api (multi-stage)
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json turbo.json ./
COPY packages/ packages/
COPY apps/api/ apps/api/
RUN npm ci && npx turbo build --filter=@imp/api

FROM node:22-alpine AS runner
WORKDIR /app
COPY --from=builder /app/apps/api/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
EXPOSE 3001
CMD ["node", "dist/server.js"]
```

Production `docker-compose.prod.yml`:
```yaml
services:
  api:
    build: { context: ., dockerfile: Dockerfile.api }
    environment: { NODE_ENV: production }
    deploy: { replicas: 2 }
  worker:
    build: { context: ., dockerfile: Dockerfile.worker }
    deploy: { replicas: 2 }
  web:
    build: { context: ., dockerfile: Dockerfile.web }
    deploy: { replicas: 2 }
```

**Testing**:
- `Integration: Docker build completes for all three services`
- `Integration: API container starts and /health returns 200`
- `Integration: Worker container connects to Redis and processes test job`
- `Integration: Web container serves Next.js app at port 3000`

---

#### 12.2 — CI/CD Pipeline

**What**: GitHub Actions workflow for automated testing, linting, type checking, building, and Docker image publishing.

**Design**:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx turbo lint
      - run: npx turbo typecheck

  test:
    runs-on: ubuntu-latest
    services:
      postgres: { image: postgres:16-alpine, env: { POSTGRES_DB: imp_test, POSTGRES_USER: imp, POSTGRES_PASSWORD: test }, ports: ["5432:5432"] }
      redis: { image: redis:7-alpine, ports: ["6379:6379"] }
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: npx turbo test

  build:
    needs: [lint-typecheck, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -f Dockerfile.api -t imp-api .
      - run: docker build -f Dockerfile.worker -t imp-worker .
      - run: docker build -f Dockerfile.web -t imp-web .
```

**Testing**:
- `Integration: CI pipeline passes on clean main branch`
- `Integration: CI fails on linting violation`
- `Integration: CI fails on type error`
- `Integration: CI fails on test failure`

---

#### 12.3 — Security Hardening

**What**: Implement OWASP API Security Top 10 mitigations, input sanitization, rate limiting per endpoint, and security headers.

**Design**:

Security measures (aligned with OWASP API Security Top 10):
```typescript
// OWASP API1: Broken Object Level Authorization
// - All entity access checks organization_id via RLS
// - Campaign creators verified to belong to campaign before updates

// OWASP API2: Broken Authentication
// - Argon2id password hashing (already in Phase 1)
// - Session expiry (7 days, refresh on activity)
// - Account lockout after 5 failed attempts (15-minute lockout)

// OWASP API3: Broken Object Property Level Authorization
// - Response serialization strips internal fields (passwordHash, encrypted tokens)
// - JSONB fields validated against JSON Schema on write

// OWASP API4: Unrestricted Resource Consumption
// - Per-endpoint rate limiting (auth: 10/min, search: 60/min, AI: 20/min)
// - Pagination enforced (max 100 items per page)
// - Request body size limit (1MB)

// OWASP API5: Broken Function Level Authorization
// - Role-based access: owner > admin > manager > member > viewer
// - Admin-only: user management, integrations, data deletion
// - Manager+: campaign management, payments, outreach
// - Member: read access, creator search

// Security headers (via @fastify/helmet):
// - Content-Security-Policy
// - X-Content-Type-Options: nosniff
// - Strict-Transport-Security
// - X-Frame-Options: DENY
```

Role-based access control:
```typescript
export function requireRole(minRole: UserRole) {
  const roleHierarchy: Record<UserRole, number> = {
    viewer: 0, member: 1, manager: 2, admin: 3, owner: 4,
  };
  return (request: FastifyRequest, reply: FastifyReply) => {
    if (!request.session || roleHierarchy[request.session.role] < roleHierarchy[minRole]) {
      reply.code(403).send({ error: 'Forbidden', message: `Requires ${minRole} role or above` });
      throw new Error('Forbidden');
    }
  };
}
```

**Testing**:
- `Unit: requireRole allows access for exact role match`
- `Unit: requireRole allows access for higher role`
- `Unit: requireRole denies access for lower role`
- `Unit: password hash is not present in user API responses`
- `Unit: account lockout triggers after 5 failed login attempts`
- `Integration: security headers present on all responses`
- `Integration: request body > 1MB returns 413`
- `Integration: SQL injection attempt in search query is sanitized`

---

#### 12.4 — Monitoring and Logging

**What**: Structured logging, health checks, and application metrics for production observability.

**Design**:

```typescript
// Structured logging via Pino (Fastify's built-in logger)
// Log format: JSON with timestamp, level, requestId, organizationId
// Sensitive fields (tokens, passwords, emails) redacted via Pino redaction

// Health check endpoint (expanded):
// GET /health
// Response: {
//   status: 'ok' | 'degraded' | 'down',
//   services: {
//     database: { status, latencyMs },
//     redis: { status, latencyMs },
//     elasticsearch: { status, latencyMs },
//   },
//   version: string,
//   uptime: number
// }

// Application metrics (optional Prometheus endpoint):
// - HTTP request duration histogram
// - Active sessions count
// - Queue depth per queue
// - Social API rate limit remaining
// - AI scoring latency
```

**Testing**:
- `Unit: health check returns degraded when Redis is down but DB is up`
- `Unit: health check returns down when DB is unreachable`
- `Unit: log output redacts email and token fields`
- `Integration: /health returns latency measurements for all services`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                         ─── required by everything
    │
    ├── Phase 2: Creator Discovery          ─── requires Phase 1
    │       │
    │       ├── Phase 3: Campaign Management    ─── requires Phase 2
    │       │       │
    │       │       ├── Phase 4: Outreach & CRM     ─── requires Phase 3
    │       │       │
    │       │       ├── Phase 5: Content & Compliance ─── requires Phase 3
    │       │       │       │
    │       │       │       └── Phase 6: Payments & Affiliates ─── requires Phase 5
    │       │       │
    │       │       └── Phase 7: Fraud & AI Scoring ─── requires Phase 2 + Phase 3
    │       │
    │       └── Phase 9: Platform Integrations ─── requires Phase 2
    │
    ├── Phase 8: Frontend Dashboard         ─── requires Phases 2-7 (builds incrementally)
    │
    ├── Phase 10: Predictive Analytics      ─── requires Phases 3 + 6 + 7
    │
    ├── Phase 11: GDPR/CCPA Compliance      ─── requires Phase 2 + Phase 4
    │
    └── Phase 12: Production Hardening      ─── requires all prior phases

Parallelism opportunities:
  - Phases 4, 5, and 7 can be developed concurrently after Phase 3
  - Phase 9 can be developed in parallel with Phases 4-7 (requires only Phase 2)
  - Phase 8 can begin after Phase 2 and grow incrementally as backend phases complete
  - Phase 11 can be developed in parallel with Phases 9-10
```

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented.
2. All unit tests pass (`npx turbo test`).
3. All integration tests pass (with mocked external dependencies).
4. Linting passes with zero errors (`npx turbo lint`).
5. Type checking passes with zero errors (`npx turbo typecheck`).
6. Docker build succeeds for all affected services.
7. The feature works end-to-end in a local development environment.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec at `/docs`.
9. Database migrations created and apply cleanly to fresh and existing databases.
10. New environment variables documented in `.env.example`.
11. No credentials, tokens, or secrets committed to the repository.
12. JSONB column changes have corresponding JSON Schema validators in `packages/shared/src/validators/`.
