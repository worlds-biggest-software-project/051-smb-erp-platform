# SMB ERP Platform -- Development Plan

> Project: 051-smb-erp-platform
> Created: 2026-05-25
> Status: Phased development plan -- ready for implementation

---

## Table of Contents

1. [Technology Decisions](#technology-decisions)
2. [Project Structure](#project-structure)
3. [Phase Dependency Graph](#phase-dependency-graph)
4. [Phase 1: Foundation & Core Infrastructure](#phase-1-foundation--core-infrastructure)
5. [Phase 2: Chart of Accounts & General Ledger](#phase-2-chart-of-accounts--general-ledger)
6. [Phase 3: Contacts, Products & Inventory](#phase-3-contacts-products--inventory)
7. [Phase 4: Purchasing & Sales Workflows](#phase-4-purchasing--sales-workflows)
8. [Phase 5: Invoicing & Payments](#phase-5-invoicing--payments)
9. [Phase 6: Bank Feeds & AI Reconciliation](#phase-6-bank-feeds--ai-reconciliation)
10. [Phase 7: Financial Reporting & Period Close](#phase-7-financial-reporting--period-close)
11. [Phase 8: Conversational ERP (NL Interface)](#phase-8-conversational-erp-nl-interface)
12. [Phase 9: AI Cash Flow Intelligence](#phase-9-ai-cash-flow-intelligence)
13. [Phase 10: Zero-Configuration Onboarding](#phase-10-zero-configuration-onboarding)
14. [Phase 11: Multi-Currency & Multi-Entity](#phase-11-multi-currency--multi-entity)
15. [Phase 12: Integrations, EDI & Compliance](#phase-12-integrations-edi--compliance)
16. [Definition of Done](#definition-of-done)

---

## Technology Decisions

### Database: PostgreSQL 16+ with Hybrid Relational + JSONB

**Decision:** Adopt Data Model Suggestion 3 (Hybrid Relational + JSONB) as the primary schema design, with select elements from Suggestion 1 (normalized relational for financial tables) and Suggestion 2 (event-sourced audit trail pattern for the audit log).

**Rationale:**
- The hybrid model's ~25-table schema provides the fastest path to MVP while maintaining database-level constraints on all monetary and compliance-critical fields (NUMERIC(19,4) for amounts, CHAR(3) for ISO 4217 currency codes, structured ISO 20022 address columns).
- JSONB columns on `organisation.locale_config`, `party.extended`, `product.attributes`, and document `metadata` fields provide the multi-jurisdiction flexibility required by the target market ($5M-$50M businesses) without schema migrations per locale. This directly supports the zero-configuration onboarding goal -- the AI setup engine can write inferred configurations into JSONB without DDL changes.
- The fully normalized approach (Suggestion 1, ~40 tables) is overkill for MVP and slows initial development. The event-sourced approach (Suggestion 2) introduces CQRS complexity that is premature before the core transactional flows are proven. The graph-relational hybrid (Suggestion 4) adds relationship analysis power we do not need until post-MVP.
- Keep line items as relational child tables (from Suggestion 1) rather than JSONB arrays. This is a deliberate departure from Suggestion 3's JSONB line items because: (a) invoice and PO line items are high-query-frequency fields in financial reporting, (b) database-level FK constraints on `account_id` per line are critical for GL integrity, and (c) the SMB transaction volume (~thousands of records) does not make join overhead a bottleneck.
- Adopt the immutable `audit_log` pattern from Suggestion 2 with `changes` stored as JSONB diffs -- this gives SOC 2 / ISO 27001 audit trail coverage without full event sourcing overhead.
- Plan to layer graph capabilities (Suggestion 4) in Phase 12 for supply chain analysis and vendor dependency mapping, once the core transactional system is stable.
- PostgreSQL Row-Level Security for tenant isolation (shared-schema multi-tenancy) provides the cost-efficiency needed for per-site pricing.

### Backend: TypeScript on Node.js (Fastify)

**Decision:** TypeScript with Fastify framework, Drizzle ORM, and Zod for validation.

**Rationale:**
- TypeScript provides end-to-end type safety from database schema to API response. Drizzle ORM generates typed queries from the PostgreSQL schema with zero abstraction overhead -- critical for an ERP where raw SQL performance matters for financial reporting.
- Fastify is ~3x faster than Express for JSON serialization, directly relevant for API-heavy ERP workloads. Its plugin architecture maps cleanly to ERP modules (accounting plugin, inventory plugin, purchasing plugin).
- Zod schemas serve triple duty: API input validation, JSONB structure validation (compensating for the lack of database-level type enforcement on JSONB columns), and OpenAPI 3.1 spec generation via `@fastify/swagger`.
- Node.js provides the async I/O model needed for concurrent bank feed syncs (Plaid), AI inference calls, and webhook processing.
- The alternative (Python/FastAPI) was considered for its stronger data science ecosystem (relevant for AI features), but the AI inference layer will call external LLM APIs (Claude, via Anthropic SDK) where the client language is irrelevant. The TypeScript ecosystem wins on frontend/backend code sharing and BI tooling integration.

### Frontend: Next.js 15 (App Router) with React 19

**Decision:** Next.js 15 with App Router, React Server Components, TanStack Table for data grids, and shadcn/ui component library.

**Rationale:**
- ERP interfaces are data-grid-heavy. TanStack Table provides the virtual scrolling, column resizing, sorting, filtering, and export capabilities that ERP users expect -- matching the table-intensive UX of Business Central and NetSuite.
- React Server Components reduce client bundle size for dashboard pages that are primarily read-only (financial reports, aging reports, trial balance) -- the server renders the HTML and streams it, avoiding large data payloads on the client.
- shadcn/ui provides accessible, unstyled primitives that can be themed to a professional finance UX without the design overhead of building from scratch or the weight of a full component library.
- Next.js API routes will proxy to the Fastify backend in development; in production, the frontend and backend deploy independently.

### AI Layer: Anthropic Claude API (via SDK) + LangChain.js

**Decision:** Claude for NL transaction parsing and reconciliation reasoning. LangChain.js for chain orchestration. Embeddings via local model for entity resolution.

**Rationale:**
- The conversational ERP feature requires structured output from natural language ("create a PO for 500 units of SKU-123 from Acme at $12 each, net-30" must produce a typed JSON command). Claude's tool-use / function-calling capability maps directly to this: define ERP operations as tools, let the model parse intent and extract parameters.
- Bank reconciliation explanation requires reasoning over match candidates with explainable confidence. Claude's extended thinking mode provides the chain-of-thought reasoning needed for "why did you match this transaction to this invoice?" -- a compliance requirement for AI-assisted reconciliation.
- LangChain.js orchestrates multi-step chains: NL input -> entity resolution -> command construction -> validation -> execution. This avoids hand-rolling chain logic.
- Entity resolution (matching "Acme" to vendor "Acme Corp, Inc." in the database) uses local embeddings (via `@xenova/transformers` running a small model like `all-MiniLM-L6-v2`) to avoid LLM round-trips for fuzzy matching. These embeddings are cached in PostgreSQL using `pgvector`.

### Authentication: OpenID Connect via NextAuth.js

**Decision:** NextAuth.js v5 with support for local credentials, Azure AD / Entra ID, Google Workspace, and Okta.

**Rationale:**
- Mid-market SMBs expect federated SSO with their identity provider (Azure AD for Microsoft shops, Google Workspace for Google shops). OpenID Connect covers both.
- SAML 2.0 SP-initiated flows required for enterprise customers will be added in Phase 11 via NextAuth's SAML provider.
- JWT tokens with tenant/org claims feed directly into PostgreSQL RLS policies (`SET app.current_tenant = '<tenant-id>'`).

### Deployment: Docker Compose (self-hosted) + Managed Cloud (Fly.io/Railway)

**Decision:** Docker Compose for self-hosted deployments; Fly.io or Railway for managed cloud.

**Rationale:**
- Self-hosted-first matches the ERPNext/Frappe Cloud model and the target audience (SMBs that want data sovereignty). Docker Compose provides a one-command deployment: `docker compose up` starts PostgreSQL, the backend, the frontend, and a Redis instance for job queues.
- Managed cloud option uses Fly.io (multi-region PostgreSQL, machine-per-tenant isolation possible) or Railway (simpler for single-region). Per-site pricing (not per-user) is the commercial model.
- BullMQ on Redis for background job processing: bank feed syncs, AI reconciliation batches, report generation, email delivery.

### API Design: REST with OpenAPI 3.1 + OData v4 Query Conventions

**Decision:** REST API documented via OpenAPI 3.1 spec. Adopt OData v4 query conventions ($filter, $orderby, $select, $expand, $top, $skip) for list endpoints.

**Rationale:**
- OpenAPI 3.1 is the industry standard for ERP API documentation (NetSuite, Business Central, Sage Intacct all publish OpenAPI specs). Auto-generated from Zod schemas.
- OData v4 query conventions reduce the custom query DSL surface. Customers migrating from Business Central (which uses OData natively) or SAP Business One (OData Service Layer) will find familiar query patterns.
- Full OData v4 compliance is not the goal -- only the query conventions ($filter, $orderby, etc.) for list endpoints. This avoids the OData metadata and batch protocol overhead.

### Licence: MIT

**Decision:** MIT licence, matching ERPNext.

**Rationale:**
- The project's stated goal is serving SMBs priced out of commercial AI ERP. MIT is the most permissive option, enabling commercial forks, embedded use, and integration without legal friction.
- GPL v3 (Odoo Community, Dolibarr) would restrict commercial embedding and may deter enterprise adoption.

---

## Project Structure

```
smb-erp-platform/
├── docker-compose.yml              # Self-hosted deployment
├── docker-compose.dev.yml           # Development environment
├── .env.example                     # Environment variable template
├── packages/
│   ├── shared/                      # Shared types, constants, validation schemas
│   │   ├── src/
│   │   │   ├── types/               # TypeScript type definitions
│   │   │   │   ├── accounting.ts
│   │   │   │   ├── inventory.ts
│   │   │   │   ├── purchasing.ts
│   │   │   │   ├── sales.ts
│   │   │   │   ├── party.ts
│   │   │   │   └── index.ts
│   │   │   ├── schemas/             # Zod validation schemas (shared between API and frontend)
│   │   │   │   ├── journal-entry.schema.ts
│   │   │   │   ├── purchase-order.schema.ts
│   │   │   │   ├── invoice.schema.ts
│   │   │   │   └── index.ts
│   │   │   ├── constants/           # ISO codes, enums, config defaults
│   │   │   │   ├── currencies.ts    # ISO 4217
│   │   │   │   ├── countries.ts     # ISO 3166-1/2
│   │   │   │   ├── account-types.ts
│   │   │   │   └── index.ts
│   │   │   └── utils/               # Currency formatting, date helpers
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── backend/                     # Fastify API server
│   │   ├── src/
│   │   │   ├── server.ts            # Fastify application bootstrap
│   │   │   ├── config/              # Environment and app configuration
│   │   │   ├── db/
│   │   │   │   ├── schema/          # Drizzle ORM schema definitions
│   │   │   │   │   ├── tenant.ts
│   │   │   │   │   ├── organisation.ts
│   │   │   │   │   ├── user.ts
│   │   │   │   │   ├── account.ts
│   │   │   │   │   ├── journal.ts
│   │   │   │   │   ├── party.ts
│   │   │   │   │   ├── product.ts
│   │   │   │   │   ├── inventory.ts
│   │   │   │   │   ├── purchasing.ts
│   │   │   │   │   ├── sales.ts
│   │   │   │   │   ├── invoice.ts
│   │   │   │   │   ├── payment.ts
│   │   │   │   │   ├── banking.ts
│   │   │   │   │   ├── tax.ts
│   │   │   │   │   ├── ai.ts
│   │   │   │   │   ├── audit.ts
│   │   │   │   │   └── index.ts
│   │   │   │   ├── migrations/      # Drizzle migration files
│   │   │   │   ├── seeds/           # Seed data (demo tenant, sample CoA)
│   │   │   │   └── client.ts        # Database connection pool
│   │   │   ├── modules/             # Business logic modules
│   │   │   │   ├── auth/
│   │   │   │   │   ├── routes.ts
│   │   │   │   │   ├── service.ts
│   │   │   │   │   └── middleware.ts
│   │   │   │   ├── accounting/
│   │   │   │   │   ├── routes.ts
│   │   │   │   │   ├── service.ts
│   │   │   │   │   ├── journal.service.ts
│   │   │   │   │   └── reports.service.ts
│   │   │   │   ├── inventory/
│   │   │   │   ├── purchasing/
│   │   │   │   ├── sales/
│   │   │   │   ├── invoicing/
│   │   │   │   ├── payments/
│   │   │   │   ├── banking/
│   │   │   │   ├── parties/
│   │   │   │   └── ai/
│   │   │   │       ├── routes.ts
│   │   │   │       ├── nl-command.service.ts
│   │   │   │       ├── reconciliation.service.ts
│   │   │   │       ├── cashflow.service.ts
│   │   │   │       └── onboarding.service.ts
│   │   │   ├── plugins/             # Fastify plugins
│   │   │   │   ├── auth.plugin.ts
│   │   │   │   ├── tenant.plugin.ts  # Sets RLS context
│   │   │   │   ├── audit.plugin.ts   # Request-level audit logging
│   │   │   │   └── ratelimit.plugin.ts
│   │   │   └── jobs/                # BullMQ background jobs
│   │   │       ├── bank-sync.job.ts
│   │   │       ├── reconciliation.job.ts
│   │   │       ├── report-generation.job.ts
│   │   │       └── exchange-rate.job.ts
│   │   ├── tests/
│   │   │   ├── unit/
│   │   │   ├── integration/
│   │   │   └── fixtures/
│   │   ├── drizzle.config.ts
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── frontend/                    # Next.js application
│       ├── src/
│       │   ├── app/
│       │   │   ├── layout.tsx
│       │   │   ├── (auth)/          # Login, SSO callback
│       │   │   ├── (dashboard)/     # Authenticated app shell
│       │   │   │   ├── layout.tsx   # Sidebar, top nav, org switcher
│       │   │   │   ├── page.tsx     # Dashboard home
│       │   │   │   ├── accounting/
│       │   │   │   │   ├── chart-of-accounts/
│       │   │   │   │   ├── journal-entries/
│       │   │   │   │   └── reports/
│       │   │   │   ├── inventory/
│       │   │   │   ├── purchasing/
│       │   │   │   ├── sales/
│       │   │   │   ├── invoices/
│       │   │   │   ├── payments/
│       │   │   │   ├── banking/
│       │   │   │   ├── contacts/
│       │   │   │   └── settings/
│       │   │   └── api/             # Next.js API routes (proxy to backend)
│       │   ├── components/
│       │   │   ├── ui/              # shadcn/ui primitives
│       │   │   ├── data-table/      # TanStack Table wrapper
│       │   │   ├── forms/           # Form components (react-hook-form + zod)
│       │   │   ├── ai/              # Conversational ERP chat interface
│       │   │   └── layout/          # Shell, sidebar, breadcrumbs
│       │   ├── hooks/
│       │   ├── lib/
│       │   │   ├── api-client.ts    # Typed API client (from OpenAPI spec)
│       │   │   └── utils.ts
│       │   └── styles/
│       ├── public/
│       ├── package.json
│       └── tsconfig.json
├── infra/
│   ├── postgres/
│   │   ├── init.sql                 # RLS policies, extensions (pgvector, pg_cron)
│   │   └── pg_hba.conf
│   └── redis/
│       └── redis.conf
├── docs/
│   ├── api/                         # Generated OpenAPI spec
│   └── architecture/               # Architecture decision records
├── turbo.json                       # Turborepo configuration
├── package.json                     # Root workspace
└── tsconfig.base.json
```

---

## Phase Dependency Graph

```
Phase 1: Foundation & Core Infrastructure
    |
    v
Phase 2: Chart of Accounts & General Ledger
    |
    +--> Phase 3: Contacts, Products & Inventory
    |         |
    |         v
    |    Phase 4: Purchasing & Sales Workflows
    |         |
    |         v
    |    Phase 5: Invoicing & Payments --------+
    |                                          |
    +------------------------------------------+
    |
    v
Phase 6: Bank Feeds & AI Reconciliation
    |
    v
Phase 7: Financial Reporting & Period Close
    |
    +---> Phase 8: Conversational ERP (NL Interface)
    |
    +---> Phase 9: AI Cash Flow Intelligence
    |
    +---> Phase 10: Zero-Configuration Onboarding
    |
    v
Phase 11: Multi-Currency & Multi-Entity
    |
    v
Phase 12: Integrations, EDI & Compliance
```

**Critical path:** 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7

**Parallel tracks (after Phase 7):** Phases 8, 9, and 10 can proceed in parallel once the core transactional system (Phases 1-7) is complete.

**Phase 11** depends on Phase 7 (financial reporting must work for a single entity before multi-entity consolidation).

**Phase 12** depends on Phase 11 (EDI and compliance features build on multi-entity and multi-currency support).

---

## Phase 1: Foundation & Core Infrastructure

**Goal:** Establish the development environment, database schema foundation, authentication, multi-tenancy, and API skeleton so that all subsequent phases have a working platform to build on.

**Duration:** 3-4 weeks

### Task 1.1: Monorepo Setup & Development Environment

**What:** Initialize the Turborepo monorepo with `packages/shared`, `packages/backend`, and `packages/frontend` workspaces. Configure TypeScript, ESLint, Prettier, and Vitest. Create `docker-compose.dev.yml` with PostgreSQL 16 and Redis 7.

**Design:**
```typescript
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "db:migrate": { "cache": false },
    "db:seed": { "cache": false, "dependsOn": ["db:migrate"] }
  }
}
```

```yaml
# docker-compose.dev.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: erp_dev
      POSTGRES_USER: erp
      POSTGRES_PASSWORD: erp_dev_password
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./infra/postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
volumes:
  pgdata:
```

**Testing:**
- [ ] `pnpm install` completes without errors from fresh clone
- [ ] `docker compose -f docker-compose.dev.yml up` starts PostgreSQL and Redis
- [ ] `pnpm run dev` starts both backend (port 3001) and frontend (port 3000) concurrently
- [ ] `pnpm run test` executes Vitest test suite (empty but passes)
- [ ] `pnpm run lint` passes on all workspaces
- [ ] TypeScript compilation succeeds across all workspaces with strict mode

### Task 1.2: Database Schema -- Tenant, Organisation, Users & RBAC

**What:** Create the Drizzle ORM schema definitions for `tenant`, `organisation`, `app_user`, `role`, `user_role`, `permission`, and `role_permission` tables. Write the initial Drizzle migration. Enable PostgreSQL RLS with tenant isolation policy on all tables.

**Design:**
```typescript
// packages/backend/src/db/schema/tenant.ts
import { pgTable, uuid, text, boolean, jsonb, timestamp } from 'drizzle-orm/pg-core';

export const tenant = pgTable('tenant', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  subscriptionPlan: text('subscription_plan').notNull().default('free'),
  config: jsonb('config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// packages/backend/src/db/schema/organisation.ts
export const organisation = pgTable('organisation', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenant.id),
  parentOrgId: uuid('parent_org_id').references(() => organisation.id),
  name: text('name').notNull(),
  legalName: text('legal_name'),
  taxId: text('tax_id'),
  lei: text('lei'),
  peppolParticipantId: text('peppol_participant_id'),
  countryCode: char('country_code', { length: 2 }).notNull(),
  jurisdiction: text('jurisdiction'),
  baseCurrency: char('base_currency', { length: 3 }).notNull().default('USD'),
  fiscalYearStartMonth: smallint('fiscal_year_start_month').notNull().default(1),
  isActive: boolean('is_active').notNull().default(true),
  localeConfig: jsonb('locale_config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

```sql
-- infra/postgres/init.sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS vector;

-- RLS helper: extract tenant_id from session
-- Application sets this before every request:
-- SET app.current_tenant = '<uuid>';
```

**Testing:**
- [ ] `pnpm run db:migrate` applies migration without errors
- [ ] `pnpm run db:seed` creates a demo tenant with one organisation and one admin user
- [ ] RLS test: query with `SET app.current_tenant = '<tenant-A>'` returns only tenant A data; tenant B data is invisible
- [ ] RLS test: INSERT without setting `app.current_tenant` is blocked by policy
- [ ] Unique constraint test: inserting duplicate `tenant.slug` throws constraint violation
- [ ] Foreign key test: inserting `organisation` with non-existent `tenant_id` throws FK violation
- [ ] `organisation.parent_org_id` self-reference works for hierarchical orgs

### Task 1.3: Authentication & Session Management

**What:** Implement NextAuth.js v5 with credential-based login (email/password with bcrypt) and placeholder OAuth providers (Azure AD, Google). Issue JWTs containing `tenant_id`, `org_id`, `user_id`, and role claims. Backend middleware extracts JWT claims and sets PostgreSQL RLS context.

**Design:**
```typescript
// packages/backend/src/plugins/tenant.plugin.ts
import { FastifyPluginAsync } from 'fastify';

export const tenantPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.addHook('preHandler', async (request, reply) => {
    const tenantId = request.user?.tenantId; // From JWT
    if (!tenantId) {
      reply.code(401).send({ error: 'Tenant context required' });
      return;
    }
    // Set PostgreSQL session variable for RLS
    await request.server.db.execute(
      sql`SELECT set_config('app.current_tenant', ${tenantId}, true)`
    );
  });
};
```

**Testing:**
- [ ] POST `/api/auth/register` creates a new tenant + organisation + admin user; returns JWT
- [ ] POST `/api/auth/login` with valid credentials returns JWT with correct claims
- [ ] POST `/api/auth/login` with invalid credentials returns 401
- [ ] Requests without `Authorization` header return 401
- [ ] Requests with expired JWT return 401
- [ ] JWT `tenant_id` claim correctly sets RLS context (verified by querying organisation data)
- [ ] Password stored as bcrypt hash (not plaintext) in database

### Task 1.4: API Skeleton & OpenAPI Spec Generation

**What:** Set up Fastify with `@fastify/swagger` and `@fastify/swagger-ui`. Create a health check endpoint and a basic CRUD route (for `organisation`) to validate the full stack: route -> Zod validation -> Drizzle query -> JSON response -> OpenAPI doc.

**Design:**
```typescript
// packages/backend/src/server.ts
import Fastify from 'fastify';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';

const app = Fastify({ logger: true });

await app.register(swagger, {
  openapi: {
    info: {
      title: 'SMB ERP Platform API',
      version: '0.1.0',
      description: 'AI-native, open-source ERP for small and medium businesses',
    },
    servers: [{ url: 'http://localhost:3001' }],
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
      },
    },
  },
});

await app.register(swaggerUi, { routePrefix: '/docs' });
```

**Testing:**
- [ ] GET `/health` returns `{ status: "ok", version: "0.1.0" }`
- [ ] GET `/docs` renders Swagger UI with the OpenAPI spec
- [ ] GET `/docs/json` returns valid OpenAPI 3.1 JSON
- [ ] GET `/api/organisations` returns organisations for the authenticated tenant only (RLS enforced)
- [ ] POST `/api/organisations` with invalid body returns 400 with Zod validation error details
- [ ] POST `/api/organisations` with valid body creates organisation and returns 201

### Task 1.5: Audit Log Infrastructure

**What:** Create the `audit_log` table and a Fastify plugin that automatically logs all write operations (POST, PUT, PATCH, DELETE) with the before/after diff in JSONB.

**Design:**
```typescript
// packages/backend/src/plugins/audit.plugin.ts
export const auditPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.addHook('onResponse', async (request, reply) => {
    if (['POST', 'PUT', 'PATCH', 'DELETE'].includes(request.method) && reply.statusCode < 400) {
      await fastify.db.insert(auditLog).values({
        tenantId: request.user.tenantId,
        userId: request.user.userId,
        action: request.method === 'POST' ? 'create' :
                request.method === 'DELETE' ? 'delete' : 'update',
        entityType: request.routeOptions.config?.entityType || 'unknown',
        entityId: request.auditContext?.entityId,
        changes: request.auditContext?.changes || null,
        ipAddress: request.ip,
        userAgent: request.headers['user-agent'],
      });
    }
  });
};
```

**Testing:**
- [ ] Creating an organisation via API produces an audit log entry with `action: 'create'`
- [ ] Updating an organisation produces an audit log entry with `changes` JSONB showing old and new values
- [ ] Audit log entries are tenant-isolated (RLS enforced)
- [ ] Audit log entries include IP address and user agent
- [ ] Audit log table is append-only (no UPDATE or DELETE operations succeed)
- [ ] GET `/api/audit-log?entity_type=organisation&entity_id=<id>` returns the change history

### Task 1.6: Frontend Shell & Navigation

**What:** Set up the Next.js 15 application with App Router, shadcn/ui, and the authenticated dashboard layout: sidebar navigation, top bar with user/org switcher, and placeholder pages for each ERP module.

**Design:**
```typescript
// packages/frontend/src/app/(dashboard)/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex h-screen">
      <Sidebar>
        <SidebarSection title="Finance">
          <SidebarLink href="/accounting/chart-of-accounts" icon={BookOpen} label="Chart of Accounts" />
          <SidebarLink href="/accounting/journal-entries" icon={FileText} label="Journal Entries" />
          <SidebarLink href="/accounting/reports" icon={BarChart} label="Reports" />
        </SidebarSection>
        <SidebarSection title="Operations">
          <SidebarLink href="/inventory" icon={Package} label="Inventory" />
          <SidebarLink href="/purchasing" icon={ShoppingCart} label="Purchasing" />
          <SidebarLink href="/sales" icon={DollarSign} label="Sales" />
        </SidebarSection>
        <SidebarSection title="Banking">
          <SidebarLink href="/invoices" icon={Receipt} label="Invoices" />
          <SidebarLink href="/payments" icon={CreditCard} label="Payments" />
          <SidebarLink href="/banking" icon={Building} label="Bank Accounts" />
        </SidebarSection>
      </Sidebar>
      <main className="flex-1 overflow-auto p-6">{children}</main>
    </div>
  );
}
```

**Testing:**
- [ ] Unauthenticated access to `/` redirects to `/login`
- [ ] Successful login redirects to dashboard
- [ ] Sidebar navigation links render correctly and route to placeholder pages
- [ ] Organisation switcher displays current org name and allows switching between orgs
- [ ] Responsive: sidebar collapses to icons on mobile viewport
- [ ] Dark mode toggle works (if configured)

---

## Phase 2: Chart of Accounts & General Ledger

**Goal:** Implement the double-entry accounting foundation: chart of accounts management, journal entry creation with debit/credit balancing, fiscal period management, and trial balance calculation.

**Duration:** 3-4 weeks

### Task 2.1: Chart of Accounts CRUD

**What:** Create the `account` table schema, API endpoints for CRUD operations, and the frontend chart-of-accounts tree view. Support hierarchical accounts via `parent_id`. Store GAAP/IFRS classification and XBRL tags in `metadata` JSONB.

**Design:**
```typescript
// packages/backend/src/modules/accounting/routes.ts
app.get('/api/accounts', {
  schema: {
    querystring: z.object({
      $filter: z.string().optional(),
      $orderby: z.string().optional().default('code asc'),
      account_type: z.enum(['asset','liability','equity','revenue','expense']).optional(),
      is_active: z.boolean().optional().default(true),
    }),
    response: { 200: z.object({ data: z.array(AccountSchema), count: z.number() }) },
  },
}, async (request, reply) => {
  const accounts = await accountService.list(request.query, request.user.tenantId, request.user.orgId);
  return { data: accounts, count: accounts.length };
});
```

**Testing:**
- [ ] POST `/api/accounts` creates a new account with `code`, `name`, `account_type`, and optional `parent_id`
- [ ] Duplicate `(tenant_id, org_id, code)` returns 409 Conflict
- [ ] GET `/api/accounts` returns flat list; GET `/api/accounts?view=tree` returns nested hierarchy
- [ ] PATCH `/api/accounts/:id` updates account name/type/metadata
- [ ] DELETE `/api/accounts/:id` fails if the account has journal lines referencing it (FK constraint)
- [ ] Deactivating an account (`is_active: false`) hides it from selection but preserves historical entries
- [ ] Account `metadata` JSONB stores XBRL element and GAAP/IFRS classification correctly
- [ ] Frontend tree view renders hierarchical accounts with expand/collapse

### Task 2.2: Fiscal Period Management

**What:** Create the `fiscal_period` table and API. Auto-generate fiscal periods for a fiscal year based on `organisation.fiscal_year_start_month`. Support opening, closing, and reopening periods.

**Design:**
```typescript
// packages/backend/src/modules/accounting/fiscal-period.service.ts
export async function generateFiscalPeriods(orgId: string, year: number): Promise<FiscalPeriod[]> {
  const org = await getOrganisation(orgId);
  const startMonth = org.fiscalYearStartMonth;
  const periods: NewFiscalPeriod[] = [];
  for (let i = 0; i < 12; i++) {
    const month = ((startMonth - 1 + i) % 12) + 1;
    const periodYear = startMonth > 1 && month < startMonth ? year + 1 : year;
    periods.push({
      orgId,
      name: `FY${year}-${String(i + 1).padStart(2, '0')}`,
      startDate: new Date(periodYear, month - 1, 1),
      endDate: endOfMonth(new Date(periodYear, month - 1, 1)),
      status: 'open',
    });
  }
  return db.insert(fiscalPeriod).values(periods).returning();
}
```

**Testing:**
- [ ] POST `/api/fiscal-periods/generate` with `year: 2026` creates 12 monthly periods
- [ ] Periods respect `fiscal_year_start_month` (e.g., April start creates Apr 2026 - Mar 2027)
- [ ] PATCH `/api/fiscal-periods/:id/close` sets status to 'closed' and records `closed_by`/`closed_at`
- [ ] Closing a period fails if there are draft journal entries in that period
- [ ] Reopening a closed period requires admin role
- [ ] Attempting to post a journal entry to a closed period returns 400

### Task 2.3: Journal Entry Creation & Double-Entry Enforcement

**What:** Create `journal_entry` and `journal_line` tables. Implement journal entry creation with automatic debit/credit balance validation. Support draft -> posted -> reversed lifecycle. Create database trigger to enforce balance constraint.

**Design:**
```typescript
// packages/backend/src/modules/accounting/journal.service.ts
export async function createJournalEntry(input: CreateJournalEntryInput): Promise<JournalEntry> {
  return db.transaction(async (tx) => {
    // Validate: total debits must equal total credits
    const totalDebits = input.lines.reduce((sum, l) => sum + l.debit, 0);
    const totalCredits = input.lines.reduce((sum, l) => sum + l.credit, 0);
    if (Math.abs(totalDebits - totalCredits) > 0.001) {
      throw new BalanceError(`Journal entry does not balance: debits (${totalDebits}) != credits (${totalCredits})`);
    }

    // Validate: fiscal period is open
    const period = await tx.query.fiscalPeriod.findFirst({
      where: and(eq(fiscalPeriod.id, input.fiscalPeriodId), eq(fiscalPeriod.status, 'open')),
    });
    if (!period) throw new ValidationError('Fiscal period is closed or does not exist');

    // Generate entry number
    const entryNumber = await generateSequence(tx, 'journal_entry', input.orgId);

    const [entry] = await tx.insert(journalEntry).values({
      orgId: input.orgId,
      entryNumber,
      entryDate: input.entryDate,
      fiscalPeriodId: input.fiscalPeriodId,
      description: input.description,
      sourceType: input.sourceType,
      sourceId: input.sourceId,
      status: 'draft',
      metadata: input.metadata || {},
    }).returning();

    await tx.insert(journalLine).values(
      input.lines.map((line, i) => ({
        journalEntryId: entry.id,
        accountId: line.accountId,
        description: line.description,
        debit: line.debit,
        credit: line.credit,
        currencyCode: line.currencyCode || 'USD',
        exchangeRate: line.exchangeRate || 1.0,
        baseDebit: line.debit * (line.exchangeRate || 1.0),
        baseCredit: line.credit * (line.exchangeRate || 1.0),
        dimensions: line.dimensions || {},
      }))
    );

    return entry;
  });
}
```

**Testing:**
- [ ] POST `/api/journal-entries` with balanced lines creates a draft entry
- [ ] POST `/api/journal-entries` with unbalanced lines returns 400 with balance error
- [ ] POST `/api/journal-entries` with zero total (all lines 0/0) is rejected
- [ ] Database trigger independently rejects unbalanced entries (defense in depth)
- [ ] Each journal line has either debit > 0 OR credit > 0, never both (CHECK constraint)
- [ ] PATCH `/api/journal-entries/:id/post` changes status to 'posted', sets `posted_by` and `posted_at`
- [ ] Posting to a closed fiscal period returns 400
- [ ] POST `/api/journal-entries/:id/reverse` creates a new entry with swapped debits/credits, links via `reversed_by_id`
- [ ] Editing a posted entry is forbidden (400)
- [ ] Auto-generated `entry_number` follows sequence pattern: `JE-2026-000001`
- [ ] Journal entry `metadata` JSONB correctly stores AI provenance when `source_type` is `ai_suggested`

### Task 2.4: Trial Balance Calculation

**What:** Implement a trial balance report endpoint that sums all posted journal lines by account, grouped by account type. This is the foundation for P&L and balance sheet reports in Phase 7.

**Design:**
```typescript
// packages/backend/src/modules/accounting/reports.service.ts
export async function getTrialBalance(orgId: string, asOfDate: Date): Promise<TrialBalanceRow[]> {
  return db.execute(sql`
    SELECT
      a.code,
      a.name,
      a.account_type,
      COALESCE(SUM(jl.base_debit), 0) AS total_debit,
      COALESCE(SUM(jl.base_credit), 0) AS total_credit,
      COALESCE(SUM(jl.base_debit), 0) - COALESCE(SUM(jl.base_credit), 0) AS balance
    FROM account a
    LEFT JOIN journal_line jl ON jl.account_id = a.id
    LEFT JOIN journal_entry je ON je.id = jl.journal_entry_id
      AND je.status = 'posted'
      AND je.entry_date <= ${asOfDate}
    WHERE a.org_id = ${orgId}
      AND a.is_active = TRUE
    GROUP BY a.id, a.code, a.name, a.account_type
    ORDER BY a.code
  `);
}
```

**Testing:**
- [ ] GET `/api/reports/trial-balance?as_of=2026-05-25` returns all accounts with their debit/credit totals
- [ ] Total debits equal total credits across all accounts (fundamental accounting equation)
- [ ] Only posted journal entries are included (draft/reversed entries excluded)
- [ ] `as_of` parameter correctly filters entries up to that date
- [ ] Accounts with no journal activity show zero balances
- [ ] Response includes account hierarchy information for grouped display

---

## Phase 3: Contacts, Products & Inventory

**Goal:** Implement the party (customer/vendor/employee) management, product catalogue, warehouse configuration, and stock-level tracking that purchasing and sales workflows depend on.

**Duration:** 3-4 weeks

### Task 3.1: Party Management (Customers, Vendors, Employees)

**What:** Create the `party` table with the unified customer/vendor/employee model. ISO 20022 structured address fields as relational columns. Contacts, bank accounts, and EDI configuration stored in `extended` JSONB.

**Design:**
```typescript
// packages/shared/src/schemas/party.schema.ts
export const CreatePartySchema = z.object({
  partyType: z.enum(['customer', 'vendor', 'employee', 'other']),
  code: z.string().min(1).max(20),
  displayName: z.string().min(1).max(200),
  legalName: z.string().optional(),
  taxId: z.string().optional(),
  // ISO 20022 structured address
  streetName: z.string().optional(),
  buildingNumber: z.string().optional(),
  postCode: z.string().optional(),
  townName: z.string().optional(),
  countrySubDivision: z.string().optional(),
  countryCode: z.string().length(2).optional(),
  email: z.string().email().optional(),
  phone: z.string().optional(),
  paymentTermsDays: z.number().int().min(0).max(365).default(30),
  creditLimit: z.number().nonnegative().optional(),
  currencyCode: z.string().length(3).default('USD'),
  extended: PartyExtendedSchema.optional().default({}),
});

export const PartyExtendedSchema = z.object({
  contacts: z.array(z.object({
    name: z.string(),
    title: z.string().optional(),
    email: z.string().email().optional(),
    phone: z.string().optional(),
    isPrimary: z.boolean().default(false),
  })).optional(),
  bankAccounts: z.array(z.object({
    bankName: z.string(),
    accountNumber: z.string().optional(),
    routingNumber: z.string().optional(),
    iban: z.string().optional(),
    bicSwift: z.string().optional(),
    currencyCode: z.string().length(3).default('USD'),
    isDefault: z.boolean().default(false),
  })).optional(),
  customFields: z.record(z.unknown()).optional(),
  tags: z.array(z.string()).optional(),
}).passthrough();
```

**Testing:**
- [ ] POST `/api/parties` creates a vendor with structured address and extended contacts
- [ ] GET `/api/parties?party_type=vendor` returns only vendors
- [ ] GET `/api/parties?party_type=customer` returns only customers
- [ ] Duplicate `(tenant_id, org_id, code)` returns 409 Conflict
- [ ] PATCH `/api/parties/:id` updates party details including `extended` JSONB (merge, not replace)
- [ ] Search: GET `/api/parties?$filter=display_name contains 'Acme'` returns matching parties
- [ ] Extended JSONB: adding contacts to `party.extended.contacts` works via PATCH
- [ ] Deactivating a party preserves historical references (invoices, POs)
- [ ] Frontend: party list with type filter tabs (All, Customers, Vendors, Employees)
- [ ] Frontend: party detail page shows contacts, bank accounts, and recent transactions

### Task 3.2: Product Catalogue

**What:** Create the `product` table with SKU, GTIN, pricing, and linked GL accounts (income and expense). Product attributes, categories, and extended data stored in `attributes` JSONB.

**Design:**
```typescript
// packages/backend/src/db/schema/product.ts
export const product = pgTable('product', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenant.id),
  sku: text('sku').notNull(),
  gtin: text('gtin'),
  name: text('name').notNull(),
  description: text('description'),
  productType: text('product_type').notNull().default('goods'),
  unitOfMeasure: text('unit_of_measure').notNull().default('EA'),
  costPrice: numeric('cost_price', { precision: 19, scale: 4 }).default('0'),
  sellPrice: numeric('sell_price', { precision: 19, scale: 4 }).default('0'),
  currencyCode: char('currency_code', { length: 3 }).notNull().default('USD'),
  isTrackable: boolean('is_trackable').notNull().default(true),
  reorderPoint: numeric('reorder_point', { precision: 12, scale: 3 }).default('0'),
  reorderQty: numeric('reorder_qty', { precision: 12, scale: 3 }).default('0'),
  incomeAccountId: uuid('income_account_id').references(() => account.id),
  expenseAccountId: uuid('expense_account_id').references(() => account.id),
  isActive: boolean('is_active').notNull().default(true),
  attributes: jsonb('attributes').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueSku: unique().on(table.tenantId, table.sku),
}));
```

**Testing:**
- [ ] POST `/api/products` creates a product with SKU, pricing, and linked accounts
- [ ] Duplicate `(tenant_id, sku)` returns 409 Conflict
- [ ] GET `/api/products?product_type=goods` filters by type
- [ ] GET `/api/products?$filter=sku eq 'SKU-123'` returns exact match
- [ ] GTIN field accepts valid GTINs and stores them correctly
- [ ] Product `attributes` JSONB stores categories, weight, dimensions, custom fields
- [ ] GIN index on `attributes` enables containment queries: `GET /api/products?attributes={"categories": ["electronics"]}`
- [ ] Linking `income_account_id` and `expense_account_id` to non-existent accounts fails (FK)
- [ ] Frontend: product list with search, type filter, and price columns
- [ ] Frontend: product detail page with attributes editor

### Task 3.3: Warehouse & Stock Level Management

**What:** Create `warehouse`, `stock_level`, and `stock_movement` tables. Implement stock receipt, shipment, transfer, and adjustment operations. Warehouse locations stored in `warehouse.config` JSONB.

**Design:**
```typescript
// packages/backend/src/modules/inventory/service.ts
export async function recordStockMovement(input: StockMovementInput): Promise<StockMovement> {
  return db.transaction(async (tx) => {
    const movement = await tx.insert(stockMovement).values({
      productId: input.productId,
      movementType: input.movementType,
      sourceWarehouseId: input.sourceWarehouseId,
      destWarehouseId: input.destWarehouseId,
      quantity: input.quantity,
      unitCost: input.unitCost,
      referenceType: input.referenceType,
      referenceId: input.referenceId,
      movedBy: input.userId,
    }).returning();

    // Update stock levels
    if (input.movementType === 'receipt' && input.destWarehouseId) {
      await upsertStockLevel(tx, input.productId, input.destWarehouseId, input.quantity);
    } else if (input.movementType === 'shipment' && input.sourceWarehouseId) {
      await upsertStockLevel(tx, input.productId, input.sourceWarehouseId, -input.quantity);
    } else if (input.movementType === 'transfer') {
      await upsertStockLevel(tx, input.productId, input.sourceWarehouseId!, -input.quantity);
      await upsertStockLevel(tx, input.productId, input.destWarehouseId!, input.quantity);
    }

    return movement[0];
  });
}
```

**Testing:**
- [ ] POST `/api/warehouses` creates a warehouse with config JSONB containing locations
- [ ] POST `/api/stock-movements` with `movement_type: 'receipt'` increases stock level at dest warehouse
- [ ] POST `/api/stock-movements` with `movement_type: 'shipment'` decreases stock level at source warehouse
- [ ] Shipment with quantity > available stock returns 400 (insufficient stock)
- [ ] Transfer between warehouses decreases source and increases dest atomically (single transaction)
- [ ] Stock adjustment with negative quantity reduces on-hand
- [ ] `quantity_available` generated column correctly equals `quantity_on_hand - quantity_reserved`
- [ ] GET `/api/stock-levels?product_id=<id>` returns stock across all warehouses
- [ ] GET `/api/stock-levels?warehouse_id=<id>` returns all products at that warehouse
- [ ] Stock movement records include `reference_type` and `reference_id` linking to source document (PO, SO)
- [ ] Frontend: inventory dashboard with product/warehouse matrix showing quantities

---

## Phase 4: Purchasing & Sales Workflows

**Goal:** Implement purchase order and sales order lifecycle management, including creation, approval, goods receipt, and shipment -- the transactional core that generates journal entries and stock movements.

**Duration:** 3-4 weeks

### Task 4.1: Purchase Order CRUD & Approval Workflow

**What:** Create `purchase_order` and `purchase_order_line` tables. Implement PO lifecycle: draft -> submitted -> approved -> received -> closed. Support configurable approval thresholds from `organisation.locale_config`.

**Design:**
```typescript
// packages/backend/src/modules/purchasing/service.ts
export async function approvePurchaseOrder(poId: string, userId: string): Promise<PurchaseOrder> {
  return db.transaction(async (tx) => {
    const po = await tx.query.purchaseOrder.findFirst({ where: eq(purchaseOrder.id, poId) });
    if (!po) throw new NotFoundError('Purchase order not found');
    if (po.status !== 'submitted') throw new WorkflowError('Only submitted POs can be approved');

    // Check approval authority from org locale_config
    const org = await tx.query.organisation.findFirst({ where: eq(organisation.id, po.orgId) });
    const threshold = org?.localeConfig?.approval_workflows?.purchase_order?.threshold || Infinity;
    if (po.total > threshold) {
      // Check user has manager role
      const hasRole = await checkUserRole(tx, userId, po.orgId, 'manager');
      if (!hasRole) throw new AuthorizationError('Approval authority insufficient for this amount');
    }

    const [updated] = await tx.update(purchaseOrder)
      .set({ status: 'approved', approvedBy: userId, approvedAt: new Date() })
      .where(eq(purchaseOrder.id, poId))
      .returning();
    return updated;
  });
}
```

**Testing:**
- [ ] POST `/api/purchase-orders` creates a PO with vendor, lines (product, qty, price), and payment terms
- [ ] PO line total is automatically calculated: `quantity * unit_price`
- [ ] PO `subtotal`, `tax_amount`, and `total` are calculated from line items
- [ ] PATCH `/api/purchase-orders/:id/submit` changes status from draft to submitted
- [ ] PATCH `/api/purchase-orders/:id/approve` changes status from submitted to approved
- [ ] Approval of PO above threshold requires manager role; insufficient role returns 403
- [ ] Editing a PO after approval is forbidden (returns 400)
- [ ] Cancelling a PO sets status to 'cancelled' and is irreversible
- [ ] Auto-generated `po_number` follows sequence pattern: `PO-2026-000001`
- [ ] Frontend: PO form with vendor selector, line item editor, and totals calculation
- [ ] Frontend: PO list with status filter tabs (Draft, Submitted, Approved, etc.)

### Task 4.2: Goods Receipt

**What:** Create `goods_receipt` and `goods_receipt_line` tables. Receiving goods against a PO updates the PO line's `quantity_received`, creates stock movements, and generates an accrual journal entry.

**Design:**
```typescript
// packages/backend/src/modules/purchasing/goods-receipt.service.ts
export async function createGoodsReceipt(input: GoodsReceiptInput): Promise<GoodsReceipt> {
  return db.transaction(async (tx) => {
    // Validate: PO must be approved
    const po = await tx.query.purchaseOrder.findFirst({ where: eq(purchaseOrder.id, input.purchaseOrderId) });
    if (!po || po.status !== 'approved') throw new WorkflowError('PO must be approved before receiving');

    // Create receipt
    const [receipt] = await tx.insert(goodsReceipt).values({ ... }).returning();

    for (const line of input.lines) {
      // Update PO line quantity_received
      await tx.update(purchaseOrderLine)
        .set({ quantityReceived: sql`quantity_received + ${line.quantityReceived}` })
        .where(eq(purchaseOrderLine.id, line.poLineId));

      // Create stock movement (receipt)
      await recordStockMovement(tx, {
        productId: line.productId,
        movementType: 'receipt',
        destWarehouseId: input.warehouseId,
        quantity: line.quantityReceived,
        unitCost: line.unitCost,
        referenceType: 'purchase_order',
        referenceId: po.id,
      });
    }

    // Generate accrual journal entry: debit Inventory, credit AP Accrual
    await createJournalEntry(tx, {
      orgId: po.orgId,
      entryDate: new Date(),
      description: `Goods receipt for ${po.poNumber}`,
      sourceType: 'goods_receipt',
      sourceId: receipt.id,
      lines: [
        { accountId: inventoryAccount, debit: totalCost, credit: 0 },
        { accountId: apAccrualAccount, debit: 0, credit: totalCost },
      ],
    });

    // Check if PO is fully received
    const allReceived = await checkFullyReceived(tx, po.id);
    if (allReceived) {
      await tx.update(purchaseOrder).set({ status: 'received' }).where(eq(purchaseOrder.id, po.id));
    }

    return receipt;
  });
}
```

**Testing:**
- [ ] POST `/api/goods-receipts` against an approved PO creates receipt with line items
- [ ] Stock levels at the receiving warehouse increase by the received quantity
- [ ] A stock movement record is created with `reference_type: 'purchase_order'`
- [ ] PO line `quantity_received` is updated correctly
- [ ] Receiving more than ordered quantity returns 400
- [ ] Partial receipt: PO status remains 'approved' until fully received
- [ ] Full receipt: PO status changes to 'received'
- [ ] Journal entry is created: debit Inventory, credit AP Accrual
- [ ] Receipt against a non-approved PO returns 400
- [ ] Frontend: goods receipt form pre-populated from PO with editable quantities

### Task 4.3: Sales Order CRUD & Fulfilment

**What:** Create `sales_order` and `sales_order_line` tables. Implement SO lifecycle: draft -> confirmed -> shipped -> closed. Shipping creates stock movements (shipment) and updates `quantity_shipped`.

**Design:**
```typescript
// packages/backend/src/modules/sales/service.ts
export async function shipSalesOrder(soId: string, shipmentLines: ShipmentLine[]): Promise<void> {
  return db.transaction(async (tx) => {
    const so = await tx.query.salesOrder.findFirst({ where: eq(salesOrder.id, soId) });
    if (!so || so.status !== 'confirmed') throw new WorkflowError('SO must be confirmed before shipping');

    for (const line of shipmentLines) {
      // Validate available stock
      const stock = await getStockLevel(tx, line.productId, line.warehouseId);
      if (stock.quantityAvailable < line.quantityShipped) {
        throw new InsufficientStockError(line.productId, line.warehouseId);
      }

      // Create stock movement (shipment)
      await recordStockMovement(tx, {
        productId: line.productId,
        movementType: 'shipment',
        sourceWarehouseId: line.warehouseId,
        quantity: line.quantityShipped,
        referenceType: 'sales_order',
        referenceId: so.id,
      });

      // Update SO line quantity_shipped
      await tx.update(salesOrderLine)
        .set({ quantityShipped: sql`quantity_shipped + ${line.quantityShipped}` })
        .where(eq(salesOrderLine.id, line.soLineId));
    }

    // Check if fully shipped
    const allShipped = await checkFullyShipped(tx, so.id);
    if (allShipped) {
      await tx.update(salesOrder).set({ status: 'shipped' }).where(eq(salesOrder.id, soId));
    }
  });
}
```

**Testing:**
- [ ] POST `/api/sales-orders` creates an SO with customer, lines, and delivery date
- [ ] SO line totals calculate correctly with discount percentage
- [ ] PATCH `/api/sales-orders/:id/confirm` changes status from draft to confirmed
- [ ] POST `/api/sales-orders/:id/ship` creates stock movements (shipment) and updates `quantity_shipped`
- [ ] Shipping more than confirmed quantity returns 400
- [ ] Shipping when stock is insufficient returns 400 with product/warehouse details
- [ ] Partial shipment: SO status remains 'confirmed' until fully shipped
- [ ] Full shipment: SO status changes to 'shipped'
- [ ] Stock movements reference the sales order
- [ ] Frontend: SO form with customer selector, line item editor, and shipping modal

---

## Phase 5: Invoicing & Payments

**Goal:** Implement the invoicing lifecycle (customer invoices and vendor bills), payment recording, and payment-to-invoice allocation. All financial transactions generate appropriate journal entries.

**Duration:** 3-4 weeks

### Task 5.1: Invoice & Vendor Bill Management

**What:** Create `invoice` and `invoice_line` tables. Support three invoice types: `customer_invoice`, `vendor_bill`, and `credit_note`. Invoices can be created from sales orders or purchase orders, or manually. Posting an invoice generates a journal entry.

**Design:**
```typescript
// packages/backend/src/modules/invoicing/service.ts
export async function postInvoice(invoiceId: string): Promise<Invoice> {
  return db.transaction(async (tx) => {
    const inv = await tx.query.invoice.findFirst({
      where: eq(invoice.id, invoiceId),
      with: { lines: true },
    });
    if (!inv || inv.status !== 'draft') throw new WorkflowError('Only draft invoices can be posted');

    // Generate journal entry based on invoice type
    const lines: JournalLineInput[] = [];
    if (inv.invoiceType === 'customer_invoice') {
      // Debit: Accounts Receivable
      lines.push({ accountId: arAccountId, debit: inv.total, credit: 0 });
      // Credit: Revenue per line item
      for (const line of inv.lines) {
        lines.push({ accountId: line.accountId, debit: 0, credit: line.lineTotal });
      }
      // Credit: Tax liability
      if (inv.taxAmount > 0) {
        lines.push({ accountId: taxLiabilityAccountId, debit: 0, credit: inv.taxAmount });
      }
    } else if (inv.invoiceType === 'vendor_bill') {
      // Credit: Accounts Payable
      lines.push({ accountId: apAccountId, debit: 0, credit: inv.total });
      // Debit: Expense per line item
      for (const line of inv.lines) {
        lines.push({ accountId: line.accountId, debit: line.lineTotal, credit: 0 });
      }
      // Debit: Tax receivable (input VAT)
      if (inv.taxAmount > 0) {
        lines.push({ accountId: taxReceivableAccountId, debit: inv.taxAmount, credit: 0 });
      }
    }

    const je = await createJournalEntry(tx, {
      orgId: inv.orgId,
      entryDate: inv.invoiceDate,
      description: `${inv.invoiceType}: ${inv.invoiceNumber}`,
      sourceType: 'invoice',
      sourceId: inv.id,
      lines,
    });

    // Post the journal entry
    await postJournalEntry(tx, je.id);

    // Update invoice status
    const [updated] = await tx.update(invoice)
      .set({ status: 'sent', journalEntryId: je.id })
      .where(eq(invoice.id, invoiceId))
      .returning();

    return updated;
  });
}
```

**Testing:**
- [ ] POST `/api/invoices` creates a customer invoice with line items referencing GL accounts
- [ ] POST `/api/invoices` creates a vendor bill with line items
- [ ] Invoice `amount_due` generated column equals `total - amount_paid`
- [ ] Creating an invoice from a sales order pre-populates lines from SO
- [ ] Creating a vendor bill from a purchase order pre-populates lines from PO
- [ ] Posting a customer invoice creates a journal entry: debit AR, credit Revenue + Tax
- [ ] Posting a vendor bill creates a journal entry: credit AP, debit Expense + Tax
- [ ] Posting an already-posted invoice returns 400
- [ ] Credit note creates a reversing journal entry
- [ ] Due date is calculated from `invoice_date + party.payment_terms_days`
- [ ] Frontend: invoice form with party selector, line item editor, tax calculation
- [ ] Frontend: invoice list with status badges and aging indicators

### Task 5.2: Payment Recording & Allocation

**What:** Create `payment` and `payment_allocation` tables. Payments can be allocated to one or more invoices. Recording a payment generates a journal entry and updates the invoice `amount_paid` and status.

**Design:**
```typescript
// packages/backend/src/modules/payments/service.ts
export async function recordPayment(input: PaymentInput): Promise<Payment> {
  return db.transaction(async (tx) => {
    const [payment] = await tx.insert(paymentTable).values({
      orgId: input.orgId,
      paymentNumber: await generateSequence(tx, 'payment', input.orgId),
      paymentType: input.paymentType,
      paymentMethod: input.paymentMethod,
      partyId: input.partyId,
      bankAccountId: input.bankAccountId,
      paymentDate: input.paymentDate,
      amount: input.amount,
      currencyCode: input.currencyCode,
      exchangeRate: input.exchangeRate,
      reference: input.reference,
      status: 'confirmed',
    }).returning();

    // Allocate to invoices
    let totalAllocated = 0;
    for (const alloc of input.allocations) {
      await tx.insert(paymentAllocation).values({
        paymentId: payment.id,
        invoiceId: alloc.invoiceId,
        amount: alloc.amount,
      });

      // Update invoice amount_paid
      await tx.update(invoice)
        .set({ amountPaid: sql`amount_paid + ${alloc.amount}` })
        .where(eq(invoice.id, alloc.invoiceId));

      // Check if fully paid
      const inv = await tx.query.invoice.findFirst({ where: eq(invoice.id, alloc.invoiceId) });
      if (inv && inv.amountPaid >= inv.total) {
        await tx.update(invoice).set({ status: 'paid' }).where(eq(invoice.id, alloc.invoiceId));
      } else if (inv && inv.amountPaid > 0) {
        await tx.update(invoice).set({ status: 'partial' }).where(eq(invoice.id, alloc.invoiceId));
      }

      totalAllocated += alloc.amount;
    }

    // Generate journal entry
    const jeLines: JournalLineInput[] = [];
    if (input.paymentType === 'incoming') {
      jeLines.push({ accountId: bankGlAccountId, debit: input.amount, credit: 0 });
      jeLines.push({ accountId: arAccountId, debit: 0, credit: totalAllocated });
    } else {
      jeLines.push({ accountId: apAccountId, debit: totalAllocated, credit: 0 });
      jeLines.push({ accountId: bankGlAccountId, debit: 0, credit: input.amount });
    }

    const je = await createJournalEntry(tx, { ... });
    await tx.update(paymentTable).set({ journalEntryId: je.id }).where(eq(paymentTable.id, payment.id));

    return payment;
  });
}
```

**Testing:**
- [ ] POST `/api/payments` records an incoming payment with allocations to invoices
- [ ] POST `/api/payments` records an outgoing payment to vendor bills
- [ ] Payment allocation updates invoice `amount_paid` correctly
- [ ] Fully paid invoice status changes to 'paid'
- [ ] Partially paid invoice status changes to 'partial'
- [ ] Over-allocating (total allocations > payment amount) returns 400
- [ ] Over-paying an invoice (amount_paid > total) returns 400
- [ ] Incoming payment creates JE: debit Bank, credit AR
- [ ] Outgoing payment creates JE: debit AP, credit Bank
- [ ] Payment can be voided (creates reversing JE, reverses invoice allocations)
- [ ] Frontend: payment form with invoice selector showing outstanding amounts
- [ ] Frontend: payment list with type and status filters

---

## Phase 6: Bank Feeds & AI Reconciliation

**Goal:** Connect to bank accounts via Plaid (and CSV import fallback), import bank transactions, and implement AI-powered reconciliation that matches bank transactions to invoices/payments with explainable confidence scores.

**Duration:** 4-5 weeks

### Task 6.1: Bank Account Setup & Transaction Import

**What:** Create `bank_account` and `bank_transaction` tables. Implement Plaid integration for automated bank feeds and CSV/OFX import for manual import. Deduplicate imported transactions via `import_id`.

**Design:**
```typescript
// packages/backend/src/modules/banking/plaid.service.ts
import { PlaidApi, Configuration, PlaidEnvironments } from 'plaid';

export async function syncBankTransactions(bankAccountId: string): Promise<number> {
  const bankAccount = await getBankAccount(bankAccountId);
  const plaid = new PlaidApi(new Configuration({
    basePath: PlaidEnvironments[config.plaid.environment],
    baseOptions: { headers: { 'PLAID-CLIENT-ID': config.plaid.clientId, 'PLAID-SECRET': config.plaid.secret } },
  }));

  const response = await plaid.transactionsSync({
    access_token: bankAccount.integration?.plaid?.accessToken,
  });

  let imported = 0;
  for (const txn of response.data.added) {
    const existing = await db.query.bankTransaction.findFirst({
      where: and(eq(bankTransaction.bankAccountId, bankAccountId), eq(bankTransaction.importId, txn.transaction_id)),
    });
    if (!existing) {
      await db.insert(bankTransaction).values({
        bankAccountId,
        transactionDate: txn.date,
        amount: txn.amount * -1, // Plaid uses negative for credits
        description: txn.name,
        counterpartyName: txn.merchant_name || txn.name,
        importSource: 'plaid',
        importId: txn.transaction_id,
      });
      imported++;
    }
  }
  return imported;
}
```

**Testing:**
- [ ] POST `/api/bank-accounts` creates a bank account linked to a GL account
- [ ] POST `/api/bank-accounts/:id/connect-plaid` initiates Plaid Link flow (sandbox mode)
- [ ] POST `/api/bank-accounts/:id/sync` imports transactions from Plaid
- [ ] Duplicate `import_id` transactions are skipped (idempotent sync)
- [ ] POST `/api/bank-accounts/:id/import-csv` parses and imports CSV bank statements
- [ ] CSV import handles different date formats configured in `bank_account.integration` JSONB
- [ ] GET `/api/bank-transactions?bank_account_id=<id>&is_reconciled=false` returns unreconciled transactions
- [ ] Bank transaction amounts: positive = credit (money in), negative = debit (money out)
- [ ] BullMQ job schedules automatic bank sync every 6 hours

### Task 6.2: AI-Powered Bank Reconciliation

**What:** Implement AI reconciliation that matches bank transactions to open invoices and recorded payments. Uses Claude for reasoning over candidate matches, produces confidence scores and natural-language explanations. Results stored in `bank_transaction.reconciliation` JSONB.

**Design:**
```typescript
// packages/backend/src/modules/ai/reconciliation.service.ts
import Anthropic from '@anthropic-ai/sdk';

export async function suggestReconciliation(bankTxnId: string): Promise<ReconciliationSuggestion> {
  const txn = await getBankTransaction(bankTxnId);
  const candidates = await findMatchCandidates(txn);

  const anthropic = new Anthropic();
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 1024,
    system: `You are a bank reconciliation assistant for an SMB ERP system.
Given a bank transaction and candidate matches (invoices, payments), determine the best match.
Return a JSON object with: matched_type, matched_id, confidence_score (0.00-1.00), explanation.
If no good match exists, return confidence_score < 0.50 with an explanation of why.`,
    messages: [{
      role: 'user',
      content: `Bank transaction:
- Date: ${txn.transactionDate}
- Amount: ${txn.amount}
- Description: "${txn.description}"
- Counterparty: "${txn.counterpartyName}"

Candidate matches:
${candidates.map(c => `- ${c.type} ${c.number}: $${c.amount}, party "${c.partyName}", due ${c.dueDate}`).join('\n')}`,
    }],
  });

  const suggestion = parseAIResponse(response);

  // Store suggestion on bank transaction
  await db.update(bankTransaction).set({
    reconciliation: {
      matched_type: suggestion.matchedType,
      matched_id: suggestion.matchedId,
      match_method: 'ai_suggested',
      confidence_score: suggestion.confidenceScore,
      explanation: suggestion.explanation,
    },
  }).where(eq(bankTransaction.id, bankTxnId));

  return suggestion;
}

async function findMatchCandidates(txn: BankTransaction): Promise<MatchCandidate[]> {
  // Find open invoices with similar amounts (within 5% tolerance)
  const invoices = await db.query.invoice.findMany({
    where: and(
      eq(invoice.status, 'sent'),
      between(invoice.total, txn.amount * 0.95, txn.amount * 1.05),
    ),
  });

  // Find unmatched payments with exact or similar amounts
  const payments = await db.query.payment.findMany({
    where: and(
      eq(payment.status, 'confirmed'),
      between(payment.amount, txn.amount * 0.95, txn.amount * 1.05),
    ),
  });

  return [...invoices.map(toCandidate), ...payments.map(toCandidate)];
}
```

**Testing:**
- [ ] POST `/api/bank-transactions/:id/suggest-match` returns an AI-generated match suggestion
- [ ] Suggestion includes `confidence_score` (0.00-1.00) and natural-language `explanation`
- [ ] Exact amount + counterparty match produces confidence > 0.90
- [ ] Similar amount but different counterparty produces confidence 0.50-0.80 with explanation
- [ ] No reasonable match produces confidence < 0.50 with "no match found" explanation
- [ ] POST `/api/bank-transactions/:id/confirm-match` marks the transaction as reconciled
- [ ] Confirmed reconciliation creates a link between bank transaction and invoice/payment
- [ ] POST `/api/bank-transactions/:id/batch-reconcile` processes all unreconciled transactions
- [ ] High-confidence matches (>0.95) are auto-confirmed; lower confidence requires manual review
- [ ] AI interaction is logged in `ai_interaction` table with input, output, and confidence
- [ ] Frontend: reconciliation view with bank transactions on left, suggested matches on right
- [ ] Frontend: confidence score displayed as color-coded badge (green >0.90, yellow 0.70-0.90, red <0.70)
- [ ] Frontend: "Accept" / "Reject" / "Manual Match" buttons per suggestion

---

## Phase 7: Financial Reporting & Period Close

**Goal:** Implement the core financial reports (P&L, Balance Sheet, Cash Flow Statement), AP/AR aging reports, and the month-end/year-end close process. These reports are the primary value deliverable for the CFO persona.

**Duration:** 3-4 weeks

### Task 7.1: Profit & Loss Statement

**What:** Generate a P&L statement from posted journal entries for a given fiscal period or date range. Group by account hierarchy (revenue accounts, expense accounts). Support comparison periods (current vs. prior year).

**Design:**
```typescript
// packages/backend/src/modules/accounting/reports.service.ts
export async function getProfitAndLoss(input: PLInput): Promise<PLReport> {
  const rows = await db.execute(sql`
    WITH period_entries AS (
      SELECT jl.account_id, SUM(jl.base_debit) as total_debit, SUM(jl.base_credit) as total_credit
      FROM journal_line jl
      JOIN journal_entry je ON je.id = jl.journal_entry_id
      WHERE je.org_id = ${input.orgId}
        AND je.status = 'posted'
        AND je.entry_date BETWEEN ${input.startDate} AND ${input.endDate}
      GROUP BY jl.account_id
    )
    SELECT
      a.code, a.name, a.account_type, a.parent_id,
      a.metadata->>'gaap_classification' as classification,
      COALESCE(pe.total_credit - pe.total_debit, 0) AS amount
    FROM account a
    LEFT JOIN period_entries pe ON pe.account_id = a.id
    WHERE a.org_id = ${input.orgId}
      AND a.account_type IN ('revenue', 'expense')
      AND a.is_active = TRUE
    ORDER BY a.account_type DESC, a.code
  `);

  const revenue = rows.filter(r => r.accountType === 'revenue');
  const expenses = rows.filter(r => r.accountType === 'expense');
  const totalRevenue = revenue.reduce((sum, r) => sum + r.amount, 0);
  const totalExpenses = expenses.reduce((sum, r) => sum + Math.abs(r.amount), 0);

  return {
    period: { startDate: input.startDate, endDate: input.endDate },
    revenue: { accounts: revenue, total: totalRevenue },
    expenses: { accounts: expenses, total: totalExpenses },
    netIncome: totalRevenue - totalExpenses,
  };
}
```

**Testing:**
- [ ] GET `/api/reports/profit-and-loss?start=2026-01-01&end=2026-03-31` returns revenue, expenses, and net income
- [ ] Revenue accounts show credit balances as positive amounts
- [ ] Expense accounts show debit balances as positive amounts
- [ ] Net income = total revenue - total expenses
- [ ] Comparison mode: `?compare=prior_year` returns current and prior year side by side
- [ ] Accounts with no activity in the period show zero (not excluded)
- [ ] Only posted entries are included
- [ ] Report respects account hierarchy for grouped subtotals
- [ ] Frontend: P&L report with expandable account groups and comparison columns
- [ ] Export: PDF and Excel download endpoints

### Task 7.2: Balance Sheet

**What:** Generate a balance sheet as of a specific date. Assets = Liabilities + Equity. Retained earnings calculated as cumulative net income from all prior periods.

**Testing:**
- [ ] GET `/api/reports/balance-sheet?as_of=2026-03-31` returns assets, liabilities, equity sections
- [ ] Balance sheet balances: total assets = total liabilities + total equity
- [ ] Retained earnings includes cumulative net income from inception to `as_of` date
- [ ] Current period net income is included in equity section
- [ ] Comparison mode: `?compare=prior_year` shows both dates
- [ ] Frontend: balance sheet report with hierarchical account display

### Task 7.3: AP/AR Aging Reports

**What:** Generate accounts payable and accounts receivable aging reports with standard aging buckets (Current, 1-30, 31-60, 61-90, 90+ days).

**Design:**
```typescript
export async function getARAgingReport(orgId: string, asOfDate: Date): Promise<AgingReport> {
  return db.execute(sql`
    SELECT
      p.code AS party_code,
      p.display_name AS party_name,
      i.invoice_number,
      i.invoice_date,
      i.due_date,
      i.total - i.amount_paid AS outstanding,
      CASE
        WHEN i.due_date >= ${asOfDate} THEN 'current'
        WHEN ${asOfDate} - i.due_date BETWEEN 1 AND 30 THEN '1_30'
        WHEN ${asOfDate} - i.due_date BETWEEN 31 AND 60 THEN '31_60'
        WHEN ${asOfDate} - i.due_date BETWEEN 61 AND 90 THEN '61_90'
        ELSE '90_plus'
      END AS aging_bucket
    FROM invoice i
    JOIN party p ON p.id = i.party_id
    WHERE i.org_id = ${orgId}
      AND i.invoice_type = 'customer_invoice'
      AND i.status NOT IN ('paid', 'cancelled')
      AND i.invoice_date <= ${asOfDate}
    ORDER BY p.display_name, i.due_date
  `);
}
```

**Testing:**
- [ ] GET `/api/reports/ar-aging?as_of=2026-05-25` returns customer aging with buckets
- [ ] GET `/api/reports/ap-aging?as_of=2026-05-25` returns vendor aging with buckets
- [ ] Aging buckets are correctly calculated relative to the `as_of` date
- [ ] Paid and cancelled invoices are excluded
- [ ] Summary row shows total outstanding per aging bucket
- [ ] Drill-down: clicking a customer shows their individual invoices
- [ ] Frontend: aging report with color-coded buckets and sortable columns

### Task 7.4: Period Close Process

**What:** Implement the month-end and year-end close process. Closing a period prevents new journal entries, generates closing entries (for revenue/expense accounts), and creates the opening balance for the next period.

**Testing:**
- [ ] POST `/api/fiscal-periods/:id/close` validates all entries are posted (no drafts remain)
- [ ] Closing generates a closing journal entry that zeros revenue and expense accounts to retained earnings
- [ ] Year-end close generates the retained earnings transfer entry
- [ ] After closing, attempting to post a new entry to that period returns 400
- [ ] Reopening a closed period requires admin role and creates an audit log entry
- [ ] Cash flow statement is generated from payment journal entries for the closed period
- [ ] Frontend: period close checklist showing outstanding items before close

---

## Phase 8: Conversational ERP (NL Interface)

**Goal:** Implement the natural-language transaction creation interface -- the core differentiator. Users can create POs, invoices, expenses, and other transactions by typing plain-language commands instead of navigating forms.

**Duration:** 4-5 weeks

### Task 8.1: NL Command Parser with Tool Use

**What:** Build the AI command processing pipeline that accepts natural-language input, uses Claude's tool-use capability to parse intent and extract parameters, resolves entities (vendors, products, accounts) via fuzzy matching, and constructs the appropriate API call.

**Design:**
```typescript
// packages/backend/src/modules/ai/nl-command.service.ts
const ERP_TOOLS = [
  {
    name: 'create_purchase_order',
    description: 'Create a new purchase order for goods or services from a vendor',
    input_schema: {
      type: 'object',
      properties: {
        vendor_name: { type: 'string', description: 'Vendor/supplier name (will be fuzzy matched)' },
        lines: {
          type: 'array',
          items: {
            type: 'object',
            properties: {
              product_sku_or_name: { type: 'string' },
              quantity: { type: 'number' },
              unit_price: { type: 'number' },
            },
            required: ['product_sku_or_name', 'quantity', 'unit_price'],
          },
        },
        payment_terms: { type: 'string', description: 'e.g. NET30, NET60, 2/10-NET30' },
      },
      required: ['vendor_name', 'lines'],
    },
  },
  {
    name: 'create_invoice',
    description: 'Create a customer invoice or vendor bill',
    input_schema: { ... },
  },
  {
    name: 'record_expense',
    description: 'Record a business expense',
    input_schema: { ... },
  },
  {
    name: 'check_stock',
    description: 'Check current stock levels for a product',
    input_schema: { ... },
  },
];

export async function processNLCommand(input: string, userId: string, orgId: string): Promise<NLCommandResult> {
  const anthropic = new Anthropic();

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 2048,
    system: `You are an ERP assistant. Parse the user's natural-language command and call the appropriate tool.
If the command is ambiguous, ask for clarification. If it doesn't match any tool, explain what you can do.`,
    tools: ERP_TOOLS,
    messages: [{ role: 'user', content: input }],
  });

  if (response.stop_reason === 'tool_use') {
    const toolCall = response.content.find(c => c.type === 'tool_use');
    const resolvedParams = await resolveEntities(toolCall.input, orgId);
    const result = await executeERPCommand(toolCall.name, resolvedParams, userId, orgId);

    // Log the interaction
    await logAIInteraction({
      userId,
      inputText: input,
      parsedIntent: toolCall.name,
      outputAction: `${toolCall.name}_executed`,
      outputEntityType: result.entityType,
      outputEntityId: result.entityId,
      confidenceScore: resolvedParams.overallConfidence,
    });

    return result;
  }

  return { type: 'clarification', message: extractTextResponse(response) };
}
```

**Testing:**
- [ ] Input "create a PO for 500 units of SKU-123 from Acme at $12 each, net-30" creates a valid PO
- [ ] Input "bill from Acme for $6,000 for 500 widgets" creates a vendor bill
- [ ] Input "invoice customer Contoso for 100 units of Widget A at $25 each" creates a customer invoice
- [ ] Input "record expense $42.50 for office supplies" creates an expense entry
- [ ] Input "what's our stock of SKU-123?" returns current stock levels
- [ ] Ambiguous vendor name (e.g., "Acme") resolves to correct vendor via fuzzy matching
- [ ] If multiple vendor matches exist, the system asks for clarification
- [ ] Unrecognized product SKU returns a clarification request
- [ ] Non-ERP commands (e.g., "what's the weather?") return a polite scope explanation
- [ ] Every interaction is logged in `ai_interaction` table
- [ ] Created documents are flagged with `ai_created: true` / `metadata.ai_generated: true`
- [ ] Processing latency < 3 seconds for typical commands

### Task 8.2: Entity Resolution via Embeddings

**What:** Build the fuzzy entity matching layer that resolves natural-language references ("Acme", "widgets", "John's company") to database records (vendor IDs, product IDs). Uses pgvector for embedding-based similarity search.

**Design:**
```typescript
// packages/backend/src/modules/ai/entity-resolution.service.ts
import { pipeline } from '@xenova/transformers';

let embedder: any;
async function getEmbedder() {
  if (!embedder) {
    embedder = await pipeline('feature-extraction', 'Xenova/all-MiniLM-L6-v2');
  }
  return embedder;
}

export async function resolveVendor(name: string, orgId: string): Promise<ResolvedEntity> {
  const embed = await getEmbedder();
  const [embedding] = await embed(name, { pooling: 'mean', normalize: true });

  const matches = await db.execute(sql`
    SELECT id, display_name, code,
           1 - (name_embedding <=> ${pgvector(embedding)}) AS similarity
    FROM party
    WHERE org_id = ${orgId}
      AND party_type = 'vendor'
      AND is_active = TRUE
    ORDER BY name_embedding <=> ${pgvector(embedding)}
    LIMIT 5
  `);

  if (matches[0]?.similarity > 0.85) {
    return { id: matches[0].id, name: matches[0].displayName, confidence: matches[0].similarity };
  }
  return { id: null, candidates: matches.slice(0, 3), confidence: matches[0]?.similarity || 0 };
}
```

**Testing:**
- [ ] "Acme" resolves to "Acme Corp, Inc." with confidence > 0.85
- [ ] "Acme Corp" resolves to same vendor with higher confidence
- [ ] Misspelling "Acne Corp" still resolves correctly (embedding similarity)
- [ ] Unknown vendor name returns top 3 candidates with confidence scores
- [ ] Product resolution: "widgets" resolves to "Widget Model A" if only one product matches
- [ ] Multiple product matches return candidates for user selection
- [ ] Embeddings are pre-computed when parties/products are created or updated
- [ ] pgvector index enables sub-100ms similarity search for typical tenant sizes

### Task 8.3: Conversational UI

**What:** Build the chat interface in the frontend -- a persistent command bar/chat panel where users can type natural-language commands and see results inline with links to the created documents.

**Testing:**
- [ ] Chat panel is accessible from every page via keyboard shortcut (Cmd+K / Ctrl+K)
- [ ] User types a command and sees a loading state during processing
- [ ] Successful command shows the created document with a link to view it
- [ ] Clarification requests show options the user can click to select
- [ ] Command history is persisted and searchable
- [ ] Chat panel shows recent AI interactions
- [ ] Error states (network failure, AI timeout) are handled gracefully
- [ ] Mobile: chat panel is full-screen modal

---

## Phase 9: AI Cash Flow Intelligence

**Goal:** Implement proactive 30/60/90-day cash flow forecasting driven by AP/AR aging patterns, payment history, and seasonality. Alert CFOs to emerging shortfalls before they become crises.

**Duration:** 3-4 weeks

### Task 9.1: Cash Flow Projection Engine

**What:** Build the cash flow forecasting model that analyses historical payment patterns (time-to-pay by customer, payment regularity), scheduled AP obligations, and expected AR collections to project daily cash positions for the next 30/60/90 days.

**Design:**
```typescript
// packages/backend/src/modules/ai/cashflow.service.ts
export async function generateCashFlowForecast(orgId: string): Promise<CashFlowForecast> {
  // 1. Current cash position from bank account balances
  const currentCash = await getCurrentCashBalance(orgId);

  // 2. Expected inflows: open AR invoices weighted by customer payment history
  const arInvoices = await getOpenARInvoices(orgId);
  const customerPaymentPatterns = await analysePaymentPatterns(orgId, 'incoming');
  const expectedInflows = arInvoices.map(inv => ({
    invoiceId: inv.id,
    amount: inv.amountDue,
    expectedDate: predictPaymentDate(inv, customerPaymentPatterns[inv.partyId]),
    confidence: customerPaymentPatterns[inv.partyId]?.confidence || 0.5,
  }));

  // 3. Expected outflows: open AP bills by due date
  const apBills = await getOpenAPBills(orgId);
  const expectedOutflows = apBills.map(bill => ({
    invoiceId: bill.id,
    amount: bill.amountDue,
    expectedDate: bill.dueDate, // Assume we pay on time
    confidence: 0.95,
  }));

  // 4. Project daily balances for 90 days
  const dailyProjections = projectDailyBalances(currentCash, expectedInflows, expectedOutflows, 90);

  // 5. Identify shortfall alerts
  const alerts = dailyProjections
    .filter(day => day.projectedBalance < 0)
    .map(day => ({
      date: day.date,
      shortfall: Math.abs(day.projectedBalance),
      severity: day.projectedBalance < -10000 ? 'critical' : 'warning',
    }));

  // 6. Use Claude for natural-language summary and recommendations
  const summary = await generateCashFlowNarrative(dailyProjections, alerts);

  return { currentCash, dailyProjections, alerts, summary };
}
```

**Testing:**
- [ ] GET `/api/reports/cash-flow-forecast` returns 90-day daily cash position projection
- [ ] Forecast includes current cash balance from bank accounts
- [ ] Expected inflows are weighted by customer payment history (late payers get later projected dates)
- [ ] Expected outflows use AP due dates
- [ ] Shortfall alerts are generated when projected balance goes negative
- [ ] Alert severity: 'critical' for shortfalls > $10K, 'warning' for smaller amounts
- [ ] AI-generated narrative summary explains the forecast in plain English
- [ ] Forecast accuracy tracking: compare predicted vs. actual (stored for model improvement)
- [ ] Frontend: cash flow chart with projected balance line, inflow/outflow bars, and alert markers
- [ ] Frontend: dashboard widget showing next 30-day cash forecast summary

### Task 9.2: Proactive Alerts & Early Payment Discount Detection

**What:** BullMQ scheduled job that generates weekly cash flow forecasts and sends alerts (email, in-app notification) when shortfalls are detected or when early payment discounts are available and cash is sufficient.

**Testing:**
- [ ] Weekly job generates forecast and sends email alerts for projected shortfalls
- [ ] In-app notification appears on dashboard for upcoming shortfalls
- [ ] System detects "2/10 NET30" terms where paying early saves money and cash is available
- [ ] Early payment discount alert includes the savings amount and the deadline
- [ ] Alert preferences are configurable per user (email on/off, threshold amounts)

---

## Phase 10: Zero-Configuration Onboarding

**Goal:** Implement the AI-driven onboarding that infers appropriate chart of accounts, tax rules, and approval workflows from a business description and optional bank statement upload, collapsing the multi-week ERP setup process.

**Duration:** 3-4 weeks

### Task 10.1: AI Configuration Engine

**What:** Build the onboarding wizard that collects a business description (industry, size, country, structure) and uses Claude to generate a complete initial configuration: chart of accounts, tax rates, payment terms, and approval workflows.

**Design:**
```typescript
// packages/backend/src/modules/ai/onboarding.service.ts
export async function generateConfiguration(input: OnboardingInput): Promise<ERPConfiguration> {
  const anthropic = new Anthropic();

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    system: `You are an ERP configuration specialist. Given a business description, generate a complete
initial ERP configuration including chart of accounts, tax rates, and payment terms.
Output must be valid JSON matching the provided schema.

Rules:
- Chart of accounts must follow ${input.accountingStandard || 'GAAP'} classification
- Tax rates must match the jurisdiction (${input.countryCode})
- Include industry-specific accounts for ${input.industry}
- Payment terms should reflect industry norms`,
    messages: [{
      role: 'user',
      content: `Business: ${input.businessDescription}
Industry: ${input.industry}
Country: ${input.countryCode}
Annual revenue: ${input.annualRevenue}
Employee count: ${input.employeeCount}
Accounting standard: ${input.accountingStandard}`,
    }],
  });

  const config = parseConfigurationResponse(response);

  // Validate generated config against Zod schemas
  const validatedAccounts = config.accounts.map(a => AccountSchema.parse(a));
  const validatedTaxRates = config.taxRates.map(t => TaxRateSchema.parse(t));

  return { accounts: validatedAccounts, taxRates: validatedTaxRates, ...config };
}

export async function applyConfiguration(orgId: string, config: ERPConfiguration): Promise<void> {
  await db.transaction(async (tx) => {
    // Create chart of accounts hierarchy
    for (const account of config.accounts) {
      await tx.insert(accountTable).values({ orgId, ...account });
    }

    // Create tax rates
    for (const taxRate of config.taxRates) {
      await tx.insert(taxRateTable).values({ ...taxRate });
    }

    // Set org locale_config
    await tx.update(organisation).set({
      localeConfig: config.localeConfig,
    }).where(eq(organisation.id, orgId));

    // Generate fiscal periods for current year
    await generateFiscalPeriods(orgId, new Date().getFullYear());
  });
}
```

**Testing:**
- [ ] POST `/api/onboarding/generate-config` with "US-based e-commerce business" generates appropriate CoA
- [ ] Generated CoA includes industry-specific accounts (e.g., "Shipping Expenses" for e-commerce)
- [ ] US configuration includes state sales tax rates for the specified state
- [ ] UK configuration includes VAT rates (standard, reduced, zero-rated)
- [ ] Generated fiscal periods match the configured fiscal year start month
- [ ] POST `/api/onboarding/apply-config` creates all accounts, tax rates, and fiscal periods
- [ ] Applied configuration is immediately usable (can create journal entries against generated accounts)
- [ ] Configuration can be previewed before applying (dry run mode)
- [ ] Invalid AI output is caught by Zod validation and retried
- [ ] Frontend: step-by-step onboarding wizard (business description -> review config -> apply)

### Task 10.2: Bank Statement Analysis for Configuration Refinement

**What:** Allow users to upload historical bank statements during onboarding. The AI analyses transaction patterns to refine the chart of accounts (add accounts for observed expense categories) and suggest vendor/customer records.

**Testing:**
- [ ] Upload CSV bank statement during onboarding
- [ ] AI categorises transactions into expense categories (rent, utilities, supplies, etc.)
- [ ] Suggested chart of accounts is refined based on actual spending patterns
- [ ] Common payees are suggested as vendor records (e.g., "Amazon" -> vendor "Amazon.com")
- [ ] Common payment sources are suggested as customer records
- [ ] User can review and accept/reject each suggestion
- [ ] Transaction categorisation accuracy > 80% for common expense types

---

## Phase 11: Multi-Currency & Multi-Entity

**Goal:** Implement full multi-currency support (transaction currency, exchange rate management, unrealised gain/loss) and multi-entity consolidation for SMBs with multiple legal entities.

**Duration:** 4-5 weeks

### Task 11.1: Multi-Currency Transactions

**What:** Extend all transaction tables (invoices, payments, POs, SOs) to handle non-base-currency transactions. Implement exchange rate management, automatic currency conversion on journal entries, and unrealised gain/loss calculation.

**Design:**
```typescript
// packages/backend/src/modules/accounting/currency.service.ts
export async function getExchangeRate(
  from: string, to: string, date: Date, tenantId: string
): Promise<number> {
  // 1. Check tenant-specific rate for this date
  const tenantRate = await db.query.exchangeRate.findFirst({
    where: and(
      eq(exchangeRate.tenantId, tenantId),
      eq(exchangeRate.fromCurrency, from),
      eq(exchangeRate.toCurrency, to),
      eq(exchangeRate.rateDate, date),
    ),
  });
  if (tenantRate) return tenantRate.rate;

  // 2. Fetch from ECB or OpenExchangeRates API
  const externalRate = await fetchExternalRate(from, to, date);

  // 3. Cache for future use
  await db.insert(exchangeRate).values({
    tenantId, fromCurrency: from, toCurrency: to, rate: externalRate, rateDate: date, source: 'ecb',
  });

  return externalRate;
}

export async function calculateUnrealisedGainLoss(orgId: string, asOfDate: Date): Promise<GainLossReport> {
  // Find all open foreign-currency invoices and revalue at current exchange rate
  const openInvoices = await db.query.invoice.findMany({
    where: and(
      eq(invoice.orgId, orgId),
      not(eq(invoice.currencyCode, org.baseCurrency)),
      notInArray(invoice.status, ['paid', 'cancelled']),
    ),
  });

  let totalGainLoss = 0;
  for (const inv of openInvoices) {
    const currentRate = await getExchangeRate(inv.currencyCode, org.baseCurrency, asOfDate, org.tenantId);
    const currentBaseValue = inv.amountDue * currentRate;
    const bookBaseValue = inv.amountDue * inv.exchangeRate;
    totalGainLoss += currentBaseValue - bookBaseValue;
  }

  return { asOfDate, unrealisedGainLoss: totalGainLoss, details: [...] };
}
```

**Testing:**
- [ ] Creating a EUR invoice for a USD-base organisation stores the exchange rate at time of creation
- [ ] Journal entries for foreign-currency invoices use `base_debit`/`base_credit` in base currency
- [ ] Exchange rate management: daily rates can be entered manually or fetched from ECB API
- [ ] BullMQ job fetches daily exchange rates automatically
- [ ] Unrealised gain/loss report shows revaluation impact on open foreign-currency items
- [ ] Payment in foreign currency records the actual exchange rate and calculates realised gain/loss
- [ ] Multi-currency trial balance shows both transaction currency and base currency columns
- [ ] Frontend: currency selector on all transaction forms
- [ ] Frontend: exchange rate management page

### Task 11.2: Multi-Entity Consolidation

**What:** Implement multi-entity support via the existing `organisation` hierarchy (parent/child). Enable intercompany transactions, elimination entries, and consolidated financial reporting.

**Testing:**
- [ ] Creating a child organisation under a parent establishes the consolidation relationship
- [ ] Each organisation has its own chart of accounts, fiscal periods, and transactions
- [ ] Intercompany invoice: creating an invoice where both parties are organisations within the same tenant
- [ ] Consolidation report: merged P&L and balance sheet across all entities in the hierarchy
- [ ] Intercompany eliminations: revenue/expense between entities is netted out in consolidated view
- [ ] Each entity can have a different base currency; consolidation converts to parent's currency
- [ ] Organisation switcher allows users to view data for any entity they have access to
- [ ] Frontend: consolidated reporting dashboard with entity selector

---

## Phase 12: Integrations, EDI & Compliance

**Goal:** Implement external system integrations (payment gateways, e-commerce platforms), EDI document exchange, PEPPOL e-invoicing, and compliance features required for production deployment.

**Duration:** 5-6 weeks

### Task 12.1: Payment Gateway Integration (Stripe)

**What:** Integrate with Stripe for customer payment collection. Enable "Pay Now" links on customer invoices that process payments via Stripe and automatically record the payment and reconciliation in the ERP.

**Testing:**
- [ ] POST `/api/invoices/:id/payment-link` generates a Stripe payment link for the invoice amount
- [ ] Customer completing payment triggers Stripe webhook -> payment recorded in ERP
- [ ] Invoice status updates to 'paid' automatically after successful Stripe payment
- [ ] Partial payments are supported via Stripe
- [ ] Failed payments are logged but do not create ERP payment records
- [ ] Stripe fees are recorded as a separate expense journal entry

### Task 12.2: EDI Document Exchange

**What:** Implement EDI integration for ANSI X12 (850/PO, 810/Invoice, 856/ASN) and UBL document exchange. Trading partner configuration stored in `party.extended.edi_config` JSONB.

**Testing:**
- [ ] Inbound X12 850 (Purchase Order) is parsed and creates a sales order in the ERP
- [ ] Outbound X12 810 (Invoice) is generated from a customer invoice
- [ ] Outbound X12 856 (Advance Ship Notice) is generated from a shipment
- [ ] UBL 2.1 Invoice generation for European trading partners
- [ ] EDI trading partner configuration via `party.extended.edi_config`
- [ ] EDI message log tracks all inbound/outbound messages with status
- [ ] Parse errors produce meaningful error messages and do not crash the import

### Task 12.3: PEPPOL E-Invoicing

**What:** Implement PEPPOL BIS Billing 3.0 compliance for EU e-invoicing. Send and receive invoices via the PEPPOL network using an Access Point provider (e.g., Storecove, Qvalia).

**Testing:**
- [ ] POST `/api/invoices/:id/send-peppol` transmits invoice via PEPPOL network
- [ ] Invoice is formatted as PEPPOL BIS Billing 3.0 UBL
- [ ] `party.peppol_participant_id` is used for routing
- [ ] Inbound PEPPOL invoices are received and create vendor bills
- [ ] PEPPOL transmission status is tracked in `invoice.metadata.peppol_sent`
- [ ] Validation: invoice must have all required PEPPOL fields before transmission

### Task 12.4: XBRL Financial Report Export

**What:** Generate XBRL-tagged financial reports for regulatory filing. Uses the XBRL taxonomy tags stored in `account.metadata.xbrl_element`.

**Testing:**
- [ ] GET `/api/reports/balance-sheet?format=xbrl` returns valid XBRL Inline document
- [ ] XBRL tags map correctly from `account.metadata.xbrl_element`
- [ ] Generated XBRL validates against US-GAAP taxonomy
- [ ] Filing-ready report includes all required SEC XBRL elements

### Task 12.5: API Rate Limiting, Webhooks & MCP Server

**What:** Implement production API features: rate limiting (per-tenant), webhook system for external integrations, and an MCP (Model Context Protocol) server that allows AI agents to query ERP data.

**Design:**
```typescript
// MCP Server tools for AI agent access
const MCP_TOOLS = [
  {
    name: 'query_invoices',
    description: 'Query invoices with filters (status, date range, party)',
    inputSchema: { ... },
  },
  {
    name: 'get_cash_position',
    description: 'Get current cash position across all bank accounts',
    inputSchema: { ... },
  },
  {
    name: 'get_ar_aging',
    description: 'Get accounts receivable aging report',
    inputSchema: { ... },
  },
];
```

**Testing:**
- [ ] API rate limiting: 100 requests/minute per tenant (configurable)
- [ ] Rate limit exceeded returns 429 with Retry-After header
- [ ] Webhook configuration: register URL + events (invoice.created, payment.received, etc.)
- [ ] Webhook delivery: POST to registered URL with event payload and HMAC signature
- [ ] Webhook retry: 3 retries with exponential backoff on failure
- [ ] MCP server: AI agents can query financial data, create transactions, and generate reports
- [ ] MCP server: authentication via API key with per-key permission scoping

### Task 12.6: Security Hardening & Compliance Preparation

**What:** Implement OWASP Top 10 controls, GDPR data subject request handling (export/delete), and preparation for SOC 2 Type II and ISO 27001 certification.

**Testing:**
- [ ] Input validation on all endpoints (Zod schemas prevent injection)
- [ ] SQL injection: parameterised queries only (verified by static analysis)
- [ ] CSRF protection on all state-changing endpoints
- [ ] CORS configuration restricts origins to known frontend domains
- [ ] Rate limiting prevents brute-force on authentication endpoints (5 attempts/minute)
- [ ] GDPR: GET `/api/data-subject/:email/export` generates a complete data export for a data subject
- [ ] GDPR: DELETE `/api/data-subject/:email` anonymises personal data while preserving financial records
- [ ] Audit log is tamper-evident (append-only, no UPDATE/DELETE)
- [ ] All secrets (API keys, Plaid tokens) stored encrypted at rest
- [ ] Dependency vulnerability scan passes with no critical/high CVEs

---

## Definition of Done

A phase is considered **complete** when ALL of the following criteria are met:

### Code Quality
- [ ] All tasks in the phase have been implemented
- [ ] TypeScript compiles with zero errors in strict mode
- [ ] ESLint passes with zero warnings (errors are blocking; warnings are addressed or suppressed with justification)
- [ ] No `any` types except where explicitly justified and documented

### Test Coverage
- [ ] All test cases listed in each task are passing
- [ ] Unit test coverage >= 80% for business logic (services)
- [ ] Integration tests cover all API endpoints introduced in the phase
- [ ] Database constraint tests verify FK, UNIQUE, and CHECK constraints
- [ ] RLS tests verify tenant isolation for all new tables

### Documentation
- [ ] OpenAPI spec is auto-generated and accurate for all new endpoints
- [ ] New Zod schemas are documented with descriptions and examples
- [ ] Architecture decision records updated for any design changes made during implementation
- [ ] CHANGELOG entry added for the phase

### Data Integrity
- [ ] All monetary calculations produce correct results for edge cases (zero amounts, maximum precision, rounding)
- [ ] Double-entry accounting invariant holds: total debits = total credits across all posted entries
- [ ] Multi-tenant isolation verified: no cross-tenant data leakage in any query

### Performance
- [ ] API response times < 200ms for single-record operations (p95)
- [ ] List endpoints with pagination return < 500ms for 10,000-record datasets (p95)
- [ ] Database queries have appropriate indexes (no sequential scans on tables > 1000 rows for common queries)
- [ ] No N+1 query patterns in any endpoint

### Deployment
- [ ] `docker compose up` starts a working instance from scratch with the demo seed data
- [ ] Database migrations apply cleanly from an empty database
- [ ] Environment variables documented in `.env.example`
- [ ] No hardcoded secrets or credentials in source code
