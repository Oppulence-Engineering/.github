# Oppulence Platform Architecture

> Revenue Operations OS - The first platform managing the complete revenue lifecycle from customer acquisition through payment recovery.

## Overview

Oppulence is a **Platform + Apps** architecture where a unified financial API powers multiple applications for SMB SaaS companies.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL PROVIDERS                              │
│  Stripe │ Plaid │ GoCardless │ Teller │ EnableBanking │ QuickBooks │ Xero  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DEVELOPER PLATFORM (Foundation)                       │
│                                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ Connections │  │Transactions │  │ Enrichment  │  │   Accounts  │        │
│  │    API      │  │    API      │  │    API      │  │     API     │        │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │  Core Services: RBAC │ Multi-Tenancy │ Rate Limiting │ Caching  │        │
│  └─────────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
┌───────────────────────┐ ┌───────────────────┐ ┌───────────────────────┐
│         POLAR         │ │       CANVAS      │ │     FUTURE APPS       │
│   Billing/Storefront  │ │   Dunning/Recovery│ │    (3rd Party Dev)    │
│                       │ │                   │ │                       │
│ • Subscriptions       │ │ • Failed Payment  │ │ • Your App Here       │
│ • Checkout            │ │   Detection       │ │ • Accounting Apps     │
│ • Customer Portal     │ │ • Recovery Flows  │ │ • Analytics Apps      │
│ • Invoicing           │ │ • Email Sequences │ │ • Marketplace         │
│ • Usage Billing       │ │ • Magic Links     │ │                       │
└───────────────────────┘ └───────────────────┘ └───────────────────────┘
                    │                 │
                    └────────┬────────┘
                             ▼
              ┌─────────────────────────────┐
              │        SYNC ENGINE          │
              │       Data Pipeline         │
              │                             │
              │ • Stripe Webhook Processing │
              │ • Real-time Data Sync       │
              │ • ClickHouse Analytics      │
              │ • Multi-tenant Isolation    │
              └─────────────────────────────┘
                             │
                             ▼
              ┌─────────────────────────────┐
              │      INFRASTRUCTURE         │
              │                             │
              │ • Storage (oppulence-storage)│
              │ • Usage Billing (OpenMeter) │
              │ • Databases (PostgreSQL,    │
              │   ClickHouse, Redis)        │
              └─────────────────────────────┘
```

## Repository Map

| Repository | Layer | Purpose | Stack |
|------------|-------|---------|-------|
| `oppulence-developer-platform` | **Platform** | Unified Financial API | Go + TypeScript/Cloudflare |
| `oppulence` | **App** | Billing/Storefront (Polar fork) | Python/FastAPI + Next.js |
| `oppulence-canvas` | **App** | Dunning & Recovery | TypeScript/Next.js + Supabase |
| `oppulence-sync-engine` | **Infrastructure** | Stripe Data Pipeline | TypeScript/Fastify + ClickHouse |
| `oppulence-storage` | **Infrastructure** | Object Storage | TypeScript/Fastify + S3 |
| `oppulence-usage-based-billing` | **Infrastructure** | Usage Metering (OpenMeter fork) | Go + Kafka + ClickHouse |

## Layer Details

### 1. Developer Platform (Foundation Layer)

**Repository**: `oppulence-developer-platform`

The unified API that aggregates financial data from multiple providers and exposes it through a single interface.

```
┌─────────────────────────────────────────────────────────────┐
│                   DEVELOPER PLATFORM                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  PROVIDER INTEGRATIONS                                       │
│  ┌─────────┐ ┌─────────┐ ┌───────────┐ ┌────────┐ ┌───────┐ │
│  │ Stripe  │ │  Plaid  │ │GoCardless │ │ Teller │ │Enable │ │
│  │ Connect │ │  OAuth  │ │ Agreement │ │  Auth  │ │Banking│ │
│  └─────────┘ └─────────┘ └───────────┘ └────────┘ └───────┘ │
│                                                              │
│  API ENDPOINTS                                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ POST /v1/connections      - Create provider connection│   │
│  │ GET  /v1/transactions     - Fetch transactions        │   │
│  │ POST /v1/transactions.sync - Trigger sync job         │   │
│  │ POST /v1/enrichment       - AI transaction enrichment │   │
│  │ GET  /v1/accounts         - List connected accounts   │   │
│  │ GET  /v1/balances         - Get account balances      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  AI ENRICHMENT (9+ Endpoints)                                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ • Merchant detection & categorization                 │   │
│  │ • Tax categorization                                  │   │
│  │ • Business purpose detection                          │   │
│  │ • Receipt requirement detection                       │   │
│  │ • Recurring transaction detection                     │   │
│  │ • Worker classification                               │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  CORE SERVICES                                               │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────┐  │
│  │    RBAC    │ │Multi-Tenant│ │Rate Limiting│ │  Cache   │  │
│  │ Permissions│ │ Workspaces │ │  Per-tenant │ │  Redis   │  │
│  └────────────┘ └────────────┘ └────────────┘ └──────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Tech Stack**:
- **API**: Cloudflare Workers (TypeScript/Hono)
- **Backend Services**: Go microservices
- **Database**: MySQL (PlanetScale) + ClickHouse
- **Cache**: Redis

**Key Files**:
- `apps/api/` - Cloudflare Workers API
- `apps/financial-engine-api/` - Financial endpoints
- `go/` - Go services (control plane, billing, assets)
- `internal/` - Shared packages (db, cache, encryption, RBAC)

---

### 2. Polar (Billing/Storefront App)

**Repository**: `oppulence` (Polar fork)

Full-featured billing and subscription management platform for SMBs.

```
┌─────────────────────────────────────────────────────────────┐
│                         POLAR                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  BILLING FEATURES                                            │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐   │
│  │ Subscriptions  │ │    Checkout    │ │   Invoicing    │   │
│  │ Monthly/Annual │ │ Hosted/Embedded│ │  Auto-generate │   │
│  └────────────────┘ └────────────────┘ └────────────────┘   │
│                                                              │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐   │
│  │Customer Portal │ │  Usage Billing │ │   Discounts    │   │
│  │  Self-service  │ │   Metered      │ │  Promo Codes   │   │
│  └────────────────┘ └────────────────┘ └────────────────┘   │
│                                                              │
│  STOREFRONT                                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ • Product catalog                                     │   │
│  │ • Pricing pages                                       │   │
│  │ • Checkout flows                                      │   │
│  │ • Customer dashboard                                  │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  INTEGRATIONS (via Developer Platform)                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ → GET /v1/accounts (bank reconciliation)              │   │
│  │ → POST /v1/enrichment (transaction categorization)    │   │
│  │ → GET /v1/transactions (financial data display)       │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Tech Stack**:
- **Backend**: Python/FastAPI + SQLAlchemy
- **Frontend**: Next.js + React + Tailwind
- **Database**: PostgreSQL
- **Queue**: Redis + Dramatiq

**Key Files**:
- `server/polar/` - Core backend modules
- `clients/apps/web/` - Next.js dashboard
- `server/migrations/` - Database migrations

---

### 3. Canvas (Dunning/Recovery App)

**Repository**: `oppulence-canvas`

Automated payment recovery and dunning system.

```
┌─────────────────────────────────────────────────────────────┐
│                        CANVAS                                │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  RECOVERY FEATURES                                           │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐   │
│  │ Failed Payment │ │   Recovery     │ │   Magic Link   │   │
│  │  Detection     │ │   Workflows    │ │ Payment Update │   │
│  └────────────────┘ └────────────────┘ └────────────────┘   │
│                                                              │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐   │
│  │ Email Sequences│ │  Tone Studio   │ │   Analytics    │   │
│  │  Pre-built     │ │ Brand-safe msg │ │ Recovery rates │   │
│  └────────────────┘ └────────────────┘ └────────────────┘   │
│                                                              │
│  WORKFLOW ENGINE                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ • Visual flow builder                                 │   │
│  │ • Pre-dunning / Soft decline / Hard decline / Past-due│   │
│  │ • Conditional logic and branching                     │   │
│  │ • Wait states and retry scheduling                    │   │
│  │ • A/B testing with holdout groups                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  INTEGRATIONS (via Developer Platform)                       │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ → GET /v1/transactions (failed payment detection)     │   │
│  │ → POST /v1/enrichment (customer context)              │   │
│  │ → GET /v1/accounts (bank status check)                │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Tech Stack**:
- **Frontend**: Next.js 16 + React 19 + TypeScript
- **Database**: Supabase (PostgreSQL)
- **Jobs**: Graphile Worker
- **Deployment**: Vercel + Docker/K8s

**Key Files**:
- `apps/web/` - Main Next.js application
- `packages/engine/` - Workflow engine
- `packages/worker/` - Background job processing
- `packages/db/` - Database schema (Drizzle)

---

### 4. Sync Engine (Data Pipeline)

**Repository**: `oppulence-sync-engine`

Real-time Stripe data synchronization with analytics capabilities.

```
┌─────────────────────────────────────────────────────────────┐
│                      SYNC ENGINE                             │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  DATA FLOW                                                   │
│                                                              │
│  Stripe ──webhook──► Fastify API ──► PostgreSQL             │
│                          │                                   │
│                          └──────► ClickHouse (analytics)     │
│                                                              │
│  FEATURES                                                    │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐   │
│  │  100% Stripe   │ │   Dual-Write   │ │  Multi-Tenant  │   │
│  │  Object Sync   │ │  PG + ClickHouse│ │   Isolation    │   │
│  └────────────────┘ └────────────────┘ └────────────────┘   │
│                                                              │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐   │
│  │ Webhook Replay │ │  Backfill Jobs │ │   Retry Queue  │   │
│  │  Event history │ │ Historical data│ │  Dead-letter   │   │
│  └────────────────┘ └────────────────┘ └────────────────┘   │
│                                                              │
│  SYNCED OBJECTS                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ customers, subscriptions, invoices, charges,          │   │
│  │ payment_intents, payment_methods, products, prices,   │   │
│  │ balance_transactions, transfers, payouts              │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

**Tech Stack**:
- **API**: Fastify (TypeScript)
- **Databases**: PostgreSQL + ClickHouse
- **Jobs**: Graphile Worker
- **ORM**: Prisma

**Key Files**:
- `packages/sync-engine/` - Core sync logic
- `packages/fastify-app/` - HTTP API
- `packages/worker/` - Background jobs

---

## Data Flow Diagram

```
                                USER / CUSTOMER
                                      │
                    ┌─────────────────┴─────────────────┐
                    ▼                                   ▼
            ┌─────────────┐                     ┌─────────────┐
            │    POLAR    │                     │   CANVAS    │
            │  Dashboard  │                     │  Dashboard  │
            └──────┬──────┘                     └──────┬──────┘
                   │                                   │
                   │ API calls                         │ API calls
                   ▼                                   ▼
            ┌─────────────────────────────────────────────────┐
            │              DEVELOPER PLATFORM                  │
            │                                                  │
            │  /v1/transactions  /v1/enrichment  /v1/accounts │
            └──────────────────────┬──────────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
            ┌───────────┐  ┌───────────┐  ┌───────────┐
            │  Stripe   │  │   Plaid   │  │ GoCardless│
            │    API    │  │    API    │  │    API    │
            └───────────┘  └───────────┘  └───────────┘
                    │
                    │ Webhooks
                    ▼
            ┌─────────────────────────────────────────────────┐
            │                  SYNC ENGINE                     │
            │                                                  │
            │  Stripe webhooks → Process → Store → Notify     │
            └──────────────────────┬──────────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
            ┌───────────┐  ┌───────────┐  ┌───────────┐
            │PostgreSQL │  │ClickHouse │  │   Redis   │
            │  (OLTP)   │  │  (OLAP)   │  │  (Cache)  │
            └───────────┘  └───────────┘  └───────────┘
```

## Authentication & Multi-Tenancy

```
┌─────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION FLOW                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. User signs in via Polar (NextAuth + WorkOS)              │
│                           │                                  │
│                           ▼                                  │
│  2. Gets Workspace ID + API Key                              │
│                           │                                  │
│                           ▼                                  │
│  3. Calls Developer Platform with API Key                    │
│     Header: Authorization: Bearer <api_key>                  │
│     Header: X-Workspace-Id: <workspace_id>                   │
│                           │                                  │
│                           ▼                                  │
│  4. Developer Platform validates + enforces RBAC             │
│                           │                                  │
│                           ▼                                  │
│  5. Data returned scoped to workspace (multi-tenant)         │
│                                                              │
└─────────────────────────────────────────────────────────────┘

MULTI-TENANCY MODEL:
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│  Workspace (Tenant)                                          │
│  ├── API Keys (multiple per workspace)                       │
│  ├── Connections (Stripe, Plaid, etc.)                       │
│  ├── Transactions (isolated per workspace)                   │
│  ├── Users (with roles: Owner, Admin, Member)                │
│  └── Settings (billing, preferences)                         │
│                                                              │
│  Row-Level Security (RLS) ensures data isolation             │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    PRODUCTION DEPLOYMENT                     │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  EDGE (Cloudflare)                                           │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Developer Platform API (Workers)                      │   │
│  │ • Global edge deployment                              │   │
│  │ • Auto-scaling                                        │   │
│  │ • DDoS protection                                     │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  COMPUTE (Vercel / Kubernetes)                               │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐   │
│  │  Polar (Web)   │ │  Canvas (Web)  │ │  Sync Engine   │   │
│  │    Vercel      │ │    Vercel      │ │   Kubernetes   │   │
│  └────────────────┘ └────────────────┘ └────────────────┘   │
│                                                              │
│  DATA (Managed Services)                                     │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐   │
│  │  PostgreSQL    │ │   ClickHouse   │ │     Redis      │   │
│  │  (Supabase/    │ │   (ClickHouse  │ │   (Upstash)    │   │
│  │   PlanetScale) │ │     Cloud)     │ │                │   │
│  └────────────────┘ └────────────────┘ └────────────────┘   │
│                                                              │
│  STORAGE                                                     │
│  ┌────────────────┐                                         │
│  │  S3 / R2       │                                         │
│  │  (Cloudflare)  │                                         │
│  └────────────────┘                                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Technology Stack Summary

| Layer | Component | Technology |
|-------|-----------|------------|
| **Platform** | Developer Platform API | Go + TypeScript/Cloudflare Workers |
| **App** | Polar (Billing) | Python/FastAPI + Next.js |
| **App** | Canvas (Dunning) | TypeScript/Next.js + Supabase |
| **Infrastructure** | Sync Engine | TypeScript/Fastify |
| **Infrastructure** | Storage | TypeScript/Fastify + S3 |
| **Infrastructure** | Usage Billing | Go + Kafka + ClickHouse |
| **Database** | OLTP | PostgreSQL (Supabase/PlanetScale) |
| **Database** | OLAP | ClickHouse |
| **Cache** | Distributed | Redis (Upstash) |
| **Edge** | CDN/API | Cloudflare Workers |
| **Deployment** | Apps | Vercel |
| **Deployment** | Services | Kubernetes (Helm) |

## API Integration Examples

### Canvas calling Developer Platform

```typescript
// Canvas: Detect failed payments
const response = await fetch('https://api.oppulence.io/v1/transactions', {
  headers: {
    'Authorization': `Bearer ${apiKey}`,
    'X-Workspace-Id': workspaceId,
  },
  body: JSON.stringify({
    provider: 'stripe',
    status: 'failed',
    type: 'charge',
    since: lastSyncTimestamp,
  })
});

const failedPayments = await response.json();
// Trigger recovery workflow for each failed payment
```

### Polar calling Developer Platform

```python
# Polar: Get enriched transactions for reconciliation
response = requests.get(
    'https://api.oppulence.io/v1/transactions',
    headers={
        'Authorization': f'Bearer {api_key}',
        'X-Workspace-Id': workspace_id,
    },
    params={
        'provider': 'plaid',
        'enrich': True,
        'since': last_sync,
    }
)

transactions = response.json()
# Match against Stripe payouts for reconciliation
```

### Sync Engine pushing to Developer Platform

```typescript
// Sync Engine: Push Stripe webhook data
await fetch('https://api.oppulence.io/v1/webhooks/stripe', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${internalApiKey}`,
    'X-Workspace-Id': workspaceId,
    'Stripe-Signature': stripeSignature,
  },
  body: JSON.stringify(stripeEvent),
});
```

## Future Roadmap

```
Phase 1 (Current)     Phase 2              Phase 3              Phase 4
─────────────────    ─────────────────    ─────────────────    ─────────────────
Developer Platform   Canvas Integration   Polar Integration    External Devs
     │                     │                    │                    │
     ▼                     ▼                    ▼                    ▼
┌─────────┐          ┌─────────┐          ┌─────────┐          ┌─────────┐
│ API MVP │ ──────►  │ Canvas  │ ──────►  │  Polar  │ ──────►  │Developer│
│ Deploy  │          │  Uses   │          │  Uses   │          │Ecosystem│
│         │          │   API   │          │   API   │          │         │
└─────────┘          └─────────┘          └─────────┘          └─────────┘
  Day 30               Day 60               Day 120              Day 180
```

---

## Quick Links

| Repository | Purpose | Docs |
|------------|---------|------|
| [oppulence-developer-platform](./oppulence-developer-platform) | Unified Financial API | [RFCs](./oppulence-developer-platform/docs/rfcs) |
| [oppulence](./oppulence) | Billing/Storefront | [README](./oppulence/README.md) |
| [oppulence-canvas](./oppulence-canvas) | Dunning/Recovery | [README](./oppulence-canvas/README.md) |
| [oppulence-sync-engine](./oppulence-sync-engine) | Stripe Data Sync | [README](./oppulence-sync-engine/README.md) |
| [oppulence-storage](./oppulence-storage) | Object Storage | [README](./oppulence-storage/README.md) |
| [oppulence-usage-based-billing](./oppulence-usaged-based-billing) | Usage Metering | [README](./oppulence-usaged-based-billing/README.md) |

---

*Last updated: November 2024*
