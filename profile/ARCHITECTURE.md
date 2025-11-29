# RFC-001: Oppulence Platform Architecture

> Revenue Operations OS — unified financial data plane and application suite for SMB SaaS companies.

| Field | Value |
|-------|-------|
| **Status** | Draft (living document) |
| **RFC Number** | RFC-001 |
| **Drivers** | Platform Team |
| **Reviewers** | Infrastructure, Application, Security Leads |
| **Created** | 2024-11 |
| **Last Updated** | 2024-11-29 |
| **Target Audience** | Engineers, Architects, DevOps |

---

## Table of Contents

1. [Problem Statement & Goals](#1-problem-statement--goals)
2. [System Context (C4 Level 1)](#2-system-context-c4-level-1)
3. [Request Lifecycles (C4 Level 2/3)](#3-request-lifecycles-c4-level-23)
4. [Component Deep Dive](#4-component-deep-dive)
5. [Data Architecture](#5-data-architecture)
6. [API Specifications](#6-api-specifications)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [Security Architecture](#8-security-architecture)
9. [Reliability & Observability](#9-reliability--observability)
10. [Deployment Architecture](#10-deployment-architecture)
11. [Performance & Scaling](#11-performance--scaling)
12. [Failure Modes & Recovery](#12-failure-modes--recovery)
13. [Development Guidelines](#13-development-guidelines)
14. [Open Items & Roadmap](#14-open-items--roadmap)

---

## 1. Problem Statement & Goals

### 1.1 Problem

Modern SaaS vendors face fragmented revenue operations:

```mermaid
flowchart LR
    subgraph Current["Current State (Fragmented)"]
        A[Stripe] --> B[Custom Code]
        C[Plaid] --> D[Another Integration]
        E[QuickBooks] --> F[Manual Reconciliation]
        G[Dunning Tool] --> H[No Data Sharing]
    end

    subgraph Issues["Pain Points"]
        I[15-40% involuntary churn]
        J[30%+ revenue leakage]
        K[Manual reconciliation]
        L[No unified view]
    end

    Current --> Issues
```

**Quantified Impact:**
- **$50B+** annual losses globally from failed payments
- **15-40%** of subscription churn is preventable (involuntary)
- **30%+** potential revenue left on table from suboptimal pricing
- **20+ hours/month** spent on manual reconciliation per SMB

### 1.2 Goals

| Goal | Success Metric | Target |
|------|---------------|--------|
| Unified Financial API | Single integration for all providers | 5+ providers |
| Reduce Involuntary Churn | Recovery rate improvement | >40% |
| Developer Experience | Time to first API call | <5 minutes |
| Multi-tenant Isolation | Zero cross-tenant data leaks | 100% |
| Platform Reliability | API availability | 99.9% |
| Latency | p99 API response time | <400ms |

### 1.3 Non-Goals

- Designing pricing tiers or commercial SLAs
- UX copy or branding guidelines
- Mobile native applications (web-first)
- Real-time trading or high-frequency financial operations

---

## 2. System Context (C4 Level 1)

### 2.1 High-Level Architecture

```mermaid
flowchart TB
    subgraph External["External Systems"]
        Stripe["Stripe<br/>Payments & Billing"]
        Plaid["Plaid<br/>Bank Connections"]
        GoCardless["GoCardless<br/>Direct Debit"]
        Teller["Teller<br/>Bank Data"]
        EnableBanking["EnableBanking<br/>Open Banking"]
        QuickBooks["QuickBooks<br/>Accounting"]
        Xero["Xero<br/>Accounting"]
    end

    subgraph Edge["Edge Layer (Cloudflare)"]
        CF["Cloudflare Workers"]
        R2["R2 Storage"]
        KV["KV Store"]
        DO["Durable Objects"]
        Queue["Queues"]
    end

    subgraph Platform["Developer Platform"]
        API["API Gateway<br/>(Hono/TypeScript)"]
        CP["Control Plane<br/>(Go Services)"]
        FE["Financial Engine<br/>(TypeScript)"]
        Enrich["Enrichment Service<br/>(AI/ML)"]
    end

    subgraph Apps["First-Party Applications"]
        Polar["Polar<br/>Billing & Storefront<br/>(Python/FastAPI)"]
        Canvas["Canvas<br/>Dunning & Recovery<br/>(Next.js)"]
    end

    subgraph DataPlane["Data Plane Services"]
        Sync["Sync Engine<br/>(Fastify/TypeScript)"]
        Storage["Object Storage<br/>(Fastify/TypeScript)"]
        UBB["Usage-Based Billing<br/>(Go/OpenMeter)"]
    end

    subgraph State["State Management"]
        PG[("PostgreSQL<br/>OLTP")]
        CH[("ClickHouse<br/>OLAP")]
        Redis[("Redis<br/>Cache")]
        S3[("S3/R2<br/>Objects")]
        Kafka[("Kafka<br/>Events")]
    end

    External --> Edge
    Edge --> Platform
    Platform <--> Apps
    Platform <--> DataPlane
    DataPlane --> State
    Platform --> State
```

### 2.2 Repository Map

| Repository | Layer | Purpose | Stack | Status |
|------------|-------|---------|-------|--------|
| `oppulence-developer-platform` | **Platform** | Unified Financial API, Control Plane | Go + TypeScript/Cloudflare | 60% MVP |
| `oppulence` | **App** | Billing/Storefront (Polar fork) | Python/FastAPI + Next.js | Production |
| `oppulence-canvas` | **App** | Dunning & Recovery | TypeScript/Next.js + Supabase | v1.4.0 |
| `oppulence-sync-engine` | **Infrastructure** | Stripe Data Pipeline | TypeScript/Fastify + ClickHouse | Production |
| `oppulence-storage` | **Infrastructure** | Object Storage | TypeScript/Fastify + S3 | v1.11.2 |
| `oppulence-usage-based-billing` | **Infrastructure** | Usage Metering (OpenMeter fork) | Go + Kafka + ClickHouse | Beta |

---

## 3. Request Lifecycles (C4 Level 2/3)

### 3.1 Checkout & Payment Flow

```mermaid
sequenceDiagram
    autonumber
    participant User as End User
    participant UI as Polar Web UI
    participant PolarAPI as Polar API<br/>(FastAPI)
    participant DevAPI as Developer Platform<br/>(CF Worker)
    participant CP as Control Plane<br/>(Go)
    participant Stripe as Stripe API
    participant PG as PostgreSQL
    participant CH as ClickHouse
    participant Redis as Redis

    User->>UI: Click "Subscribe"
    UI->>PolarAPI: POST /api/checkout/sessions

    Note over PolarAPI: Validate session, workspace

    PolarAPI->>DevAPI: POST /v1/payments/intents

    Note over DevAPI: Headers:<br/>Authorization: Bearer {api_key}<br/>X-Workspace-Id: {workspace_id}<br/>Idempotency-Key: {uuid}

    DevAPI->>Redis: Check idempotency key
    Redis-->>DevAPI: Not found (proceed)

    DevAPI->>CP: Validate tenant + RBAC
    CP->>PG: Query workspace, permissions
    PG-->>CP: Workspace valid, permission granted
    CP-->>DevAPI: Auth OK

    DevAPI->>Stripe: POST /v1/payment_intents

    Note over Stripe: Creates PaymentIntent<br/>amount: 4900<br/>currency: usd

    Stripe-->>DevAPI: PaymentIntent object

    DevAPI->>Redis: Store idempotency key (TTL: 24h)
    DevAPI->>PG: Insert transaction record
    DevAPI->>CH: Emit payment.initiated event

    DevAPI-->>PolarAPI: PaymentIntent + client_secret
    PolarAPI-->>UI: Checkout session URL
    UI-->>User: Redirect to Stripe Checkout

    User->>Stripe: Complete payment
    Stripe-->>DevAPI: Webhook: payment_intent.succeeded

    DevAPI->>CP: Process webhook
    CP->>PG: Update transaction status
    CP->>CH: Emit payment.succeeded event
    CP-->>DevAPI: ACK

    DevAPI-->>Stripe: 200 OK
```

### 3.2 Failed Payment Recovery Flow

```mermaid
sequenceDiagram
    autonumber
    participant Stripe as Stripe
    participant Sync as Sync Engine
    participant PG as PostgreSQL
    participant CH as ClickHouse
    participant Worker as Graphile Worker
    participant DevAPI as Developer Platform
    participant Canvas as Canvas
    participant Email as Email Service
    participant Customer as Customer

    Stripe-->>Sync: Webhook: invoice.payment_failed

    Note over Sync: Verify signature<br/>Extract tenant from metadata

    Sync->>PG: Upsert failed payment record
    Sync->>Worker: Enqueue recovery job

    Worker->>PG: Fetch recovery workflow config
    PG-->>Worker: Workflow: pre-dunning → soft → hard

    Worker->>DevAPI: POST /v1/enrichment

    Note over DevAPI: Enrich customer context:<br/>- Payment history<br/>- Bank status<br/>- Risk score

    DevAPI-->>Worker: Enriched context

    Worker->>CH: Log workflow.started event

    alt Day 1: Pre-Dunning
        Worker->>Email: Send friendly reminder
        Email-->>Customer: "Your payment didn't go through"
        Worker->>CH: Log email.sent event
    end

    alt Day 3: Soft Decline (if not recovered)
        Worker->>Canvas: Generate magic link
        Canvas-->>Worker: https://pay.oppulence.io/update/{token}
        Worker->>Email: Send with magic link
        Email-->>Customer: "Update your payment method"
    end

    Customer->>Canvas: Click magic link
    Canvas->>DevAPI: GET /v1/customers/{id}/payment-methods
    Canvas->>Stripe: Update payment method
    Stripe-->>Canvas: Success
    Canvas->>DevAPI: POST /v1/payments/retry
    DevAPI->>Stripe: Retry charge
    Stripe-->>DevAPI: Charge succeeded

    DevAPI->>CH: Log recovery.succeeded event
    DevAPI->>PG: Update transaction status
```

### 3.3 Bank Data Aggregation Flow

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant DevAPI as Developer Platform
    participant CP as Control Plane
    participant Plaid as Plaid API
    participant PG as PostgreSQL
    participant Redis as Redis
    participant CH as ClickHouse

    App->>DevAPI: POST /v1/connections

    Note over DevAPI: Request body:<br/>{<br/>  "provider": "plaid",<br/>  "public_token": "...",<br/>  "institution_id": "ins_1"<br/>}

    DevAPI->>CP: Validate workspace + permissions
    CP-->>DevAPI: Authorized

    DevAPI->>Plaid: POST /item/public_token/exchange
    Plaid-->>DevAPI: access_token (encrypted)

    DevAPI->>PG: Store connection (encrypted token)

    Note over PG: INSERT INTO connections<br/>(workspace_id, provider, token_enc, status)

    DevAPI->>Plaid: GET /accounts/get
    Plaid-->>DevAPI: Account list

    DevAPI->>PG: Upsert accounts
    DevAPI->>CH: Emit connection.created event
    DevAPI-->>App: Connection created

    loop Daily Sync
        DevAPI->>Plaid: GET /transactions/sync
        Plaid-->>DevAPI: New transactions

        DevAPI->>DevAPI: AI Enrichment

        Note over DevAPI: Categorize, detect merchant,<br/>flag recurring, tax category

        DevAPI->>PG: Upsert transactions
        DevAPI->>Redis: Invalidate cache
        DevAPI->>CH: Emit transactions.synced event
    end
```

### 3.4 Usage-Based Billing Flow

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant UBB as Usage Billing<br/>(OpenMeter)
    participant Kafka as Kafka
    participant CH as ClickHouse
    participant PG as PostgreSQL
    participant Polar as Polar
    participant Stripe as Stripe

    App->>UBB: POST /v1/events

    Note over UBB: Event payload:<br/>{<br/>  "type": "api_call",<br/>  "subject": "workspace_123",<br/>  "data": {"tokens": 1500}<br/>}

    UBB->>Kafka: Produce event
    Kafka->>CH: Consumer writes to events table

    Note over CH: Tumbling window aggregation<br/>per workspace, per meter

    loop Hourly Aggregation
        CH->>CH: Compute usage per window
    end

    loop End of Billing Period
        Polar->>UBB: GET /v1/meters/{id}/query

        Note over Polar: Query params:<br/>from: 2024-11-01<br/>to: 2024-11-30<br/>group_by: workspace_id

        UBB->>CH: Execute aggregation query
        CH-->>UBB: Usage data
        UBB-->>Polar: Aggregated usage

        Polar->>Polar: Calculate charges

        Note over Polar: base_fee + (usage * rate)<br/>$49 + (150000 * $0.001) = $199

        Polar->>Stripe: Create invoice
        Stripe-->>Polar: Invoice created
        Polar->>PG: Update subscription record
    end
```

---

## 4. Component Deep Dive

### 4.1 Developer Platform

#### Architecture

```mermaid
flowchart TB
    subgraph Edge["Cloudflare Edge"]
        Workers["Workers<br/>(Hono Router)"]
        DO["Durable Objects<br/>(Rate Limiting)"]
        R2["R2 Storage<br/>(Files)"]
        KV["KV Store<br/>(Config Cache)"]
        Queue["Queues<br/>(Async Jobs)"]
    end

    subgraph API["API Layer"]
        Router["Router<br/>(/v1/*)"]
        AuthMW["Auth Middleware"]
        RateMW["Rate Limit MW"]
        ValidMW["Validation MW"]
        IdempMW["Idempotency MW"]
    end

    subgraph Services["Service Layer"]
        ConnSvc["Connection Service"]
        TxnSvc["Transaction Service"]
        EnrichSvc["Enrichment Service"]
        AcctSvc["Account Service"]
        WebhookSvc["Webhook Service"]
    end

    subgraph Adapters["Provider Adapters"]
        StripeAdapter["Stripe Adapter"]
        PlaidAdapter["Plaid Adapter"]
        GCAdapter["GoCardless Adapter"]
        TellerAdapter["Teller Adapter"]
        EBAdapter["EnableBanking Adapter"]
    end

    subgraph Go["Control Plane (Go)"]
        RBAC["RBAC Service"]
        Tenant["Tenant Service"]
        Billing["Billing Service"]
        Audit["Audit Service"]
        Secrets["Secrets Manager"]
    end

    Workers --> API
    API --> Services
    Services --> Adapters
    Services <--> Go
    Workers <--> DO
    Workers <--> R2
    Workers <--> KV
    Workers --> Queue
```

#### Directory Structure

```
oppulence-developer-platform/
├── apps/
│   ├── api/                          # Cloudflare Workers API
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   │   ├── v1/
│   │   │   │   │   ├── connections.ts
│   │   │   │   │   ├── transactions.ts
│   │   │   │   │   ├── accounts.ts
│   │   │   │   │   ├── enrichment.ts
│   │   │   │   │   ├── webhooks.ts
│   │   │   │   │   └── index.ts
│   │   │   │   └── health.ts
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── rateLimit.ts
│   │   │   │   ├── validation.ts
│   │   │   │   ├── idempotency.ts
│   │   │   │   └── errorHandler.ts
│   │   │   ├── services/
│   │   │   │   ├── connection.service.ts
│   │   │   │   ├── transaction.service.ts
│   │   │   │   ├── enrichment.service.ts
│   │   │   │   └── webhook.service.ts
│   │   │   ├── adapters/
│   │   │   │   ├── stripe/
│   │   │   │   ├── plaid/
│   │   │   │   ├── gocardless/
│   │   │   │   ├── teller/
│   │   │   │   └── enablebanking/
│   │   │   └── index.ts
│   │   ├── wrangler.toml
│   │   └── package.json
│   │
│   ├── financial-engine-api/         # Financial calculations
│   │   └── src/
│   │       ├── reconciliation/
│   │       ├── forecasting/
│   │       └── analytics/
│   │
│   └── dashboard/                    # Admin dashboard
│       └── src/
│
├── go/                               # Control Plane (Go services)
│   ├── cmd/
│   │   ├── control-plane/
│   │   ├── billing-service/
│   │   └── audit-service/
│   ├── internal/
│   │   ├── rbac/
│   │   ├── tenant/
│   │   ├── secrets/
│   │   └── audit/
│   └── pkg/
│       ├── middleware/
│       └── errors/
│
├── internal/                         # Shared TypeScript packages
│   ├── db/
│   ├── cache/
│   ├── encryption/
│   ├── validation/
│   └── rbac/
│
├── docs/
│   └── rfcs/
│
└── deployment/
    ├── kubernetes/
    └── terraform/
```

#### Configuration (wrangler.toml)

```toml
name = "oppulence-api"
main = "src/index.ts"
compatibility_date = "2024-01-01"
node_compat = true

[vars]
ENVIRONMENT = "production"
API_VERSION = "v1"

[[kv_namespaces]]
binding = "CONFIG"
id = "xxx"

[[r2_buckets]]
binding = "STORAGE"
bucket_name = "oppulence-storage"

[[durable_objects.bindings]]
name = "RATE_LIMITER"
class_name = "RateLimiter"

[[queues.producers]]
queue = "webhook-processing"
binding = "WEBHOOK_QUEUE"

[[queues.consumers]]
queue = "webhook-processing"
max_batch_size = 10
max_batch_timeout = 30

[env.staging]
vars = { ENVIRONMENT = "staging" }

[env.production]
vars = { ENVIRONMENT = "production" }
routes = [
  { pattern = "api.oppulence.io/*", zone_name = "oppulence.io" }
]
```

### 4.2 Polar (Billing/Storefront)

#### Architecture

```mermaid
flowchart TB
    subgraph Frontend["Frontend (Next.js)"]
        Web["Web App<br/>(apps/web)"]
        Console["Admin Console<br/>(apps/console)"]
        Checkout["Checkout Widget<br/>(packages/checkout)"]
    end

    subgraph Backend["Backend (FastAPI)"]
        API["API Server<br/>(Uvicorn)"]
        Workers["Background Workers<br/>(Dramatiq)"]
    end

    subgraph Modules["Domain Modules"]
        Subscription["Subscription<br/>Management"]
        Customer["Customer<br/>Management"]
        Invoice["Invoice<br/>Generation"]
        Checkout2["Checkout<br/>Sessions"]
        Benefit["Benefits &<br/>Entitlements"]
        Meter["Usage<br/>Metering"]
    end

    subgraph Integrations["Integrations"]
        StripeInt["Stripe"]
        GitHubInt["GitHub OAuth"]
        DevPlatform["Developer Platform"]
    end

    subgraph Data["Data Layer"]
        PG[("PostgreSQL")]
        Redis2[("Redis")]
        S3_2[("S3/Minio")]
    end

    Frontend --> Backend
    Backend --> Modules
    Modules --> Integrations
    Modules --> Data
    Workers --> Data
```

#### Directory Structure

```
oppulence/
├── server/
│   ├── polar/                        # Core backend
│   │   ├── subscription/
│   │   │   ├── endpoints.py
│   │   │   ├── service.py
│   │   │   ├── schemas.py
│   │   │   └── tasks.py
│   │   ├── customer/
│   │   ├── invoice/
│   │   ├── checkout/
│   │   ├── benefit/
│   │   ├── meter/
│   │   ├── organization/
│   │   ├── auth/
│   │   ├── integrations/
│   │   │   ├── stripe/
│   │   │   │   ├── service.py
│   │   │   │   ├── webhooks.py
│   │   │   │   └── schemas.py
│   │   │   └── github/
│   │   ├── models/                   # SQLAlchemy models
│   │   │   ├── subscription.py
│   │   │   ├── customer.py
│   │   │   ├── invoice.py
│   │   │   └── ...
│   │   ├── app.py                    # FastAPI app
│   │   └── config.py
│   │
│   ├── migrations/                   # Alembic migrations
│   ├── polar_ext/                    # Extensions
│   │   ├── accounting/
│   │   ├── banking/
│   │   └── openmeter/
│   │
│   ├── emails/                       # React Email templates
│   ├── tests/
│   └── pyproject.toml
│
├── clients/
│   ├── apps/
│   │   ├── web/                      # Main dashboard
│   │   │   ├── src/
│   │   │   │   ├── app/              # Next.js App Router
│   │   │   │   ├── components/
│   │   │   │   ├── hooks/
│   │   │   │   └── lib/
│   │   │   └── package.json
│   │   ├── console/                  # Admin console
│   │   └── www/                      # Marketing site
│   │
│   └── packages/
│       ├── ui/                       # Shared components
│       ├── client/                   # Generated API client
│       └── checkout/                 # Embeddable checkout
│
└── docker-compose.yml
```

#### Key Models (SQLAlchemy)

```python
# server/polar/models/subscription.py

class Subscription(Base):
    __tablename__ = "subscriptions"

    id: Mapped[UUID] = mapped_column(primary_key=True, default=uuid4)
    workspace_id: Mapped[UUID] = mapped_column(ForeignKey("workspaces.id"), index=True)
    customer_id: Mapped[UUID] = mapped_column(ForeignKey("customers.id"), index=True)
    product_id: Mapped[UUID] = mapped_column(ForeignKey("products.id"))
    price_id: Mapped[UUID] = mapped_column(ForeignKey("prices.id"))

    status: Mapped[SubscriptionStatus] = mapped_column(
        Enum(SubscriptionStatus),
        default=SubscriptionStatus.INCOMPLETE
    )

    current_period_start: Mapped[datetime] = mapped_column()
    current_period_end: Mapped[datetime] = mapped_column()
    cancel_at_period_end: Mapped[bool] = mapped_column(default=False)
    canceled_at: Mapped[datetime | None] = mapped_column()
    ended_at: Mapped[datetime | None] = mapped_column()

    # Stripe reference
    stripe_subscription_id: Mapped[str | None] = mapped_column(unique=True)

    # Metadata
    metadata: Mapped[dict] = mapped_column(JSONB, default=dict)
    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow)
    updated_at: Mapped[datetime] = mapped_column(onupdate=datetime.utcnow)

    # Relationships
    workspace: Mapped["Workspace"] = relationship(back_populates="subscriptions")
    customer: Mapped["Customer"] = relationship(back_populates="subscriptions")
    invoices: Mapped[list["Invoice"]] = relationship(back_populates="subscription")

    __table_args__ = (
        Index("ix_subscriptions_workspace_status", "workspace_id", "status"),
        Index("ix_subscriptions_customer", "customer_id"),
    )


class SubscriptionStatus(str, Enum):
    INCOMPLETE = "incomplete"
    INCOMPLETE_EXPIRED = "incomplete_expired"
    TRIALING = "trialing"
    ACTIVE = "active"
    PAST_DUE = "past_due"
    CANCELED = "canceled"
    UNPAID = "unpaid"
    PAUSED = "paused"
```

### 4.3 Canvas (Dunning/Recovery)

#### Architecture

```mermaid
flowchart TB
    subgraph Frontend["Frontend (Next.js 16)"]
        Dashboard["Recovery Dashboard"]
        FlowBuilder["Visual Flow Builder"]
        ToneStudio["Tone Studio"]
        Analytics["Analytics"]
    end

    subgraph API["API Layer"]
        tRPC["tRPC Router"]
        Actions["Server Actions"]
    end

    subgraph Engine["Workflow Engine"]
        Orchestrator["Workflow Orchestrator"]
        Scheduler["Job Scheduler"]
        Executor["Step Executor"]
    end

    subgraph Workers["Background Workers"]
        GraphileWorker["Graphile Worker"]
        EmailWorker["Email Worker"]
        RetryWorker["Retry Worker"]
    end

    subgraph Integrations2["Integrations"]
        DevPlatform2["Developer Platform"]
        Stripe2["Stripe Direct"]
        Email2["Email (React Email)"]
    end

    subgraph Data2["Data Layer"]
        Supabase["Supabase<br/>(PostgreSQL)"]
        Drizzle["Drizzle ORM"]
    end

    Frontend --> API
    API --> Engine
    Engine --> Workers
    Workers --> Integrations2
    Engine --> Data2
```

#### Directory Structure

```
oppulence-canvas/
├── apps/
│   └── web/                          # Main Next.js app
│       ├── src/
│       │   ├── app/                  # App Router
│       │   │   ├── (dashboard)/
│       │   │   │   ├── workflows/
│       │   │   │   ├── recovery/
│       │   │   │   ├── analytics/
│       │   │   │   └── settings/
│       │   │   ├── api/
│       │   │   │   └── trpc/
│       │   │   └── layout.tsx
│       │   ├── components/
│       │   │   ├── workflow-builder/
│       │   │   ├── tone-studio/
│       │   │   └── analytics/
│       │   ├── lib/
│       │   │   ├── trpc/
│       │   │   └── utils/
│       │   └── server/
│       │       ├── routers/
│       │       └── services/
│       ├── next.config.ts
│       └── package.json
│
├── packages/
│   ├── engine/                       # Workflow engine
│   │   ├── src/
│   │   │   ├── orchestrator.ts
│   │   │   ├── scheduler.ts
│   │   │   ├── executor.ts
│   │   │   ├── steps/
│   │   │   │   ├── email.step.ts
│   │   │   │   ├── wait.step.ts
│   │   │   │   ├── condition.step.ts
│   │   │   │   └── retry.step.ts
│   │   │   └── types.ts
│   │   └── package.json
│   │
│   ├── worker/                       # Graphile Worker jobs
│   │   ├── src/
│   │   │   ├── jobs/
│   │   │   │   ├── sendRecoveryEmail.ts
│   │   │   │   ├── retryPayment.ts
│   │   │   │   ├── executeWorkflow.ts
│   │   │   │   └── syncFailedPayments.ts
│   │   │   └── worker.ts
│   │   └── package.json
│   │
│   ├── db/                           # Database schema
│   │   ├── src/
│   │   │   ├── schema/
│   │   │   │   ├── workflows.ts
│   │   │   │   ├── recoveries.ts
│   │   │   │   ├── customers.ts
│   │   │   │   └── analytics.ts
│   │   │   ├── migrations/
│   │   │   └── client.ts
│   │   └── drizzle.config.ts
│   │
│   ├── email/                        # Email templates
│   │   └── templates/
│   │       ├── pre-dunning.tsx
│   │       ├── soft-decline.tsx
│   │       ├── hard-decline.tsx
│   │       └── payment-updated.tsx
│   │
│   └── tone-manager/                 # Tone customization
│       └── src/
│           ├── presets/
│           └── generator.ts
│
└── turbo.json
```

#### Workflow Schema (Drizzle)

```typescript
// packages/db/src/schema/workflows.ts

import { pgTable, uuid, text, jsonb, timestamp, pgEnum } from 'drizzle-orm/pg-core';

export const workflowStatusEnum = pgEnum('workflow_status', [
  'draft',
  'active',
  'paused',
  'archived'
]);

export const workflows = pgTable('workflows', {
  id: uuid('id').primaryKey().defaultRandom(),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id),
  name: text('name').notNull(),
  description: text('description'),
  status: workflowStatusEnum('status').default('draft').notNull(),

  // DAG definition
  definition: jsonb('definition').notNull().$type<WorkflowDefinition>(),

  // Trigger configuration
  triggerType: text('trigger_type').notNull(), // 'payment_failed', 'subscription_past_due'
  triggerConfig: jsonb('trigger_config').$type<TriggerConfig>(),

  // A/B testing
  holdoutPercentage: integer('holdout_percentage').default(0),

  // Metadata
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull(),
  createdBy: uuid('created_by').references(() => users.id),
});

export const workflowRuns = pgTable('workflow_runs', {
  id: uuid('id').primaryKey().defaultRandom(),
  workflowId: uuid('workflow_id').notNull().references(() => workflows.id),
  workspaceId: uuid('workspace_id').notNull().references(() => workspaces.id),
  customerId: uuid('customer_id').notNull(),

  status: text('status').notNull(), // 'running', 'completed', 'failed', 'cancelled'
  currentStepId: text('current_step_id'),

  // Context data
  context: jsonb('context').$type<WorkflowContext>(),

  // Results
  result: jsonb('result').$type<WorkflowResult>(),

  // Timing
  startedAt: timestamp('started_at').defaultNow().notNull(),
  completedAt: timestamp('completed_at'),

  // A/B cohort
  cohort: text('cohort'), // 'control', 'treatment'
});

// Type definitions
interface WorkflowDefinition {
  steps: WorkflowStep[];
  edges: WorkflowEdge[];
}

interface WorkflowStep {
  id: string;
  type: 'email' | 'wait' | 'condition' | 'retry_payment' | 'magic_link';
  config: Record<string, unknown>;
  position: { x: number; y: number };
}

interface WorkflowEdge {
  id: string;
  source: string;
  target: string;
  condition?: string; // For conditional branches
}
```

### 4.4 Sync Engine

#### Architecture

```mermaid
flowchart TB
    subgraph Ingestion["Ingestion Layer"]
        WebhookAPI["Webhook API<br/>(Fastify)"]
        Validator["Signature Validator"]
        Router["Tenant Router"]
    end

    subgraph Processing["Processing Layer"]
        Queue["Job Queue<br/>(Graphile Worker)"]
        Processor["Event Processor"]
        Transformer["Data Transformer"]
    end

    subgraph Storage2["Storage Layer"]
        PGWriter["PostgreSQL Writer"]
        CHWriter["ClickHouse Writer"]
        Deduplicator["Deduplicator"]
    end

    subgraph Outputs["Output Layer"]
        WebhookFwd["Webhook Forwarder"]
        APINotify["API Notifications"]
        CacheInval["Cache Invalidation"]
    end

    WebhookAPI --> Validator
    Validator --> Router
    Router --> Queue
    Queue --> Processor
    Processor --> Transformer
    Transformer --> PGWriter
    Transformer --> CHWriter
    Processor --> Deduplicator
    Processor --> Outputs
```

#### Directory Structure

```
oppulence-sync-engine/
├── packages/
│   ├── sync-engine/                  # Core sync logic
│   │   ├── src/
│   │   │   ├── processors/
│   │   │   │   ├── stripe/
│   │   │   │   │   ├── customer.processor.ts
│   │   │   │   │   ├── subscription.processor.ts
│   │   │   │   │   ├── invoice.processor.ts
│   │   │   │   │   ├── charge.processor.ts
│   │   │   │   │   └── index.ts
│   │   │   │   └── index.ts
│   │   │   ├── transformers/
│   │   │   │   ├── stripe-to-canonical.ts
│   │   │   │   └── types.ts
│   │   │   ├── writers/
│   │   │   │   ├── postgres.writer.ts
│   │   │   │   └── clickhouse.writer.ts
│   │   │   └── index.ts
│   │   └── package.json
│   │
│   ├── fastify-app/                  # HTTP API
│   │   ├── src/
│   │   │   ├── routes/
│   │   │   │   ├── webhooks/
│   │   │   │   │   └── stripe.ts
│   │   │   │   ├── health.ts
│   │   │   │   └── metrics.ts
│   │   │   ├── plugins/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── tenant.ts
│   │   │   │   └── rateLimit.ts
│   │   │   └── server.ts
│   │   └── package.json
│   │
│   ├── worker/                       # Background jobs
│   │   ├── src/
│   │   │   ├── jobs/
│   │   │   │   ├── processWebhook.ts
│   │   │   │   ├── backfill.ts
│   │   │   │   ├── retry.ts
│   │   │   │   └── cleanup.ts
│   │   │   └── worker.ts
│   │   └── package.json
│   │
│   └── clickhouse-financials/        # Analytics helpers
│       └── src/
│           ├── queries/
│           ├── migrations/
│           └── types.ts
│
├── prisma/
│   └── schema.prisma
│
└── docker-compose.yml
```

#### Prisma Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Tenant {
  id                    String   @id @default(uuid())
  stripeAccountId       String?  @unique
  stripeWebhookSecret   String?
  encryptedAccessToken  String?

  customers             Customer[]
  subscriptions         Subscription[]
  invoices              Invoice[]
  charges               Charge[]
  webhookEvents         WebhookEvent[]

  createdAt             DateTime @default(now())
  updatedAt             DateTime @updatedAt

  @@index([stripeAccountId])
}

model WebhookEvent {
  id              String   @id @default(uuid())
  tenantId        String
  tenant          Tenant   @relation(fields: [tenantId], references: [id])

  stripeEventId   String
  type            String
  apiVersion      String?

  payload         Json
  processedAt     DateTime?
  failedAt        DateTime?
  failureReason   String?
  retryCount      Int      @default(0)

  createdAt       DateTime @default(now())

  @@unique([tenantId, stripeEventId])
  @@index([tenantId, type])
  @@index([processedAt])
}

model Customer {
  id                  String   @id @default(uuid())
  tenantId            String
  tenant              Tenant   @relation(fields: [tenantId], references: [id])

  stripeCustomerId    String
  email               String?
  name                String?
  phone               String?

  metadata            Json?

  subscriptions       Subscription[]
  invoices            Invoice[]
  charges             Charge[]

  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  @@unique([tenantId, stripeCustomerId])
  @@index([tenantId, email])
}

model Subscription {
  id                      String   @id @default(uuid())
  tenantId                String
  tenant                  Tenant   @relation(fields: [tenantId], references: [id])
  customerId              String
  customer                Customer @relation(fields: [customerId], references: [id])

  stripeSubscriptionId    String
  status                  String

  currentPeriodStart      DateTime
  currentPeriodEnd        DateTime
  cancelAtPeriodEnd       Boolean  @default(false)
  canceledAt              DateTime?
  endedAt                 DateTime?

  items                   Json
  metadata                Json?

  invoices                Invoice[]

  createdAt               DateTime @default(now())
  updatedAt               DateTime @updatedAt

  @@unique([tenantId, stripeSubscriptionId])
  @@index([tenantId, status])
  @@index([customerId])
}

model Invoice {
  id                  String   @id @default(uuid())
  tenantId            String
  tenant              Tenant   @relation(fields: [tenantId], references: [id])
  customerId          String
  customer            Customer @relation(fields: [customerId], references: [id])
  subscriptionId      String?
  subscription        Subscription? @relation(fields: [subscriptionId], references: [id])

  stripeInvoiceId     String
  number              String?
  status              String

  amountDue           Int
  amountPaid          Int
  amountRemaining     Int
  currency            String

  dueDate             DateTime?
  paidAt              DateTime?

  hostedInvoiceUrl    String?
  invoicePdf          String?

  lines               Json
  metadata            Json?

  charges             Charge[]

  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  @@unique([tenantId, stripeInvoiceId])
  @@index([tenantId, status])
  @@index([customerId])
}

model Charge {
  id                  String   @id @default(uuid())
  tenantId            String
  tenant              Tenant   @relation(fields: [tenantId], references: [id])
  customerId          String
  customer            Customer @relation(fields: [customerId], references: [id])
  invoiceId           String?
  invoice             Invoice? @relation(fields: [invoiceId], references: [id])

  stripeChargeId      String
  status              String

  amount              Int
  amountRefunded      Int      @default(0)
  currency            String

  paymentMethodType   String?
  failureCode         String?
  failureMessage      String?

  metadata            Json?

  createdAt           DateTime @default(now())
  updatedAt           DateTime @updatedAt

  @@unique([tenantId, stripeChargeId])
  @@index([tenantId, status])
  @@index([customerId])
}
```

---

## 5. Data Architecture

### 5.1 Database Strategy

```mermaid
flowchart LR
    subgraph OLTP["OLTP (PostgreSQL)"]
        direction TB
        Tenants[("Tenants")]
        Users[("Users")]
        Connections[("Connections")]
        Transactions[("Transactions")]
        Subscriptions2[("Subscriptions")]
    end

    subgraph OLAP["OLAP (ClickHouse)"]
        direction TB
        Events[("Events")]
        Metrics[("Metrics")]
        Analytics2[("Analytics")]
    end

    subgraph Cache2["Cache (Redis)"]
        direction TB
        Sessions[("Sessions")]
        RateLimits[("Rate Limits")]
        TxnCache[("Transaction Cache")]
    end

    subgraph Objects["Object Storage (S3/R2)"]
        direction TB
        Invoices2[("Invoices")]
        Receipts[("Receipts")]
        Exports[("Exports")]
    end

    App[Application] --> OLTP
    App --> OLAP
    App --> Cache2
    App --> Objects
```

### 5.2 PostgreSQL Schema (Developer Platform)

```sql
-- Workspace (Tenant) table
CREATE TABLE workspaces (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,

    -- Billing
    stripe_customer_id VARCHAR(255),
    subscription_status VARCHAR(50) DEFAULT 'trialing',

    -- Settings
    settings JSONB DEFAULT '{}',

    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ
);

-- Enable RLS
ALTER TABLE workspaces ENABLE ROW LEVEL SECURITY;

-- API Keys
CREATE TABLE api_keys (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,

    name VARCHAR(255) NOT NULL,
    key_hash VARCHAR(255) NOT NULL UNIQUE, -- SHA-256 hash
    key_prefix VARCHAR(10) NOT NULL, -- First 8 chars for identification

    -- Permissions
    scopes TEXT[] DEFAULT '{}',

    -- Rate limiting
    rate_limit_per_minute INT DEFAULT 1000,

    -- Expiration
    expires_at TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,

    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    revoked_at TIMESTAMPTZ
);

CREATE INDEX idx_api_keys_workspace ON api_keys(workspace_id);
CREATE INDEX idx_api_keys_hash ON api_keys(key_hash);

-- Connections (Provider accounts)
CREATE TABLE connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,

    provider VARCHAR(50) NOT NULL, -- 'stripe', 'plaid', 'gocardless', etc.
    provider_account_id VARCHAR(255),

    -- Encrypted credentials
    access_token_encrypted BYTEA,
    refresh_token_encrypted BYTEA,

    -- Status
    status VARCHAR(50) DEFAULT 'active', -- 'active', 'needs_reauth', 'disconnected'
    last_sync_at TIMESTAMPTZ,
    error_message TEXT,

    -- Metadata
    metadata JSONB DEFAULT '{}',

    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE UNIQUE INDEX idx_connections_workspace_provider ON connections(workspace_id, provider, provider_account_id);

-- Accounts (Bank accounts, cards, etc.)
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    connection_id UUID NOT NULL REFERENCES connections(id) ON DELETE CASCADE,

    provider_account_id VARCHAR(255) NOT NULL,

    -- Account details
    name VARCHAR(255),
    official_name VARCHAR(255),
    type VARCHAR(50), -- 'checking', 'savings', 'credit', etc.
    subtype VARCHAR(50),

    -- Institution
    institution_id VARCHAR(255),
    institution_name VARCHAR(255),

    -- Balance
    current_balance DECIMAL(19, 4),
    available_balance DECIMAL(19, 4),
    currency VARCHAR(3) DEFAULT 'USD',
    balance_updated_at TIMESTAMPTZ,

    -- Mask
    mask VARCHAR(10), -- Last 4 digits

    -- Status
    status VARCHAR(50) DEFAULT 'active',

    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_accounts_workspace ON accounts(workspace_id);
CREATE INDEX idx_accounts_connection ON accounts(connection_id);

-- Transactions
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    account_id UUID REFERENCES accounts(id) ON DELETE SET NULL,
    connection_id UUID REFERENCES connections(id) ON DELETE SET NULL,

    provider_transaction_id VARCHAR(255) NOT NULL,
    provider VARCHAR(50) NOT NULL,

    -- Transaction details
    amount DECIMAL(19, 4) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',

    date DATE NOT NULL,
    datetime TIMESTAMPTZ,

    -- Description
    name VARCHAR(500),
    merchant_name VARCHAR(255),

    -- Categorization
    category VARCHAR(255),
    category_id VARCHAR(255),

    -- Status
    pending BOOLEAN DEFAULT FALSE,

    -- Enrichment (AI-powered)
    enrichment JSONB DEFAULT '{}',
    enriched_at TIMESTAMPTZ,

    -- Original data
    raw_data JSONB,

    -- Timestamps
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_transactions_workspace ON transactions(workspace_id);
CREATE INDEX idx_transactions_account ON transactions(account_id);
CREATE INDEX idx_transactions_date ON transactions(workspace_id, date DESC);
CREATE UNIQUE INDEX idx_transactions_provider ON transactions(workspace_id, provider, provider_transaction_id);

-- Enable RLS on all tables
ALTER TABLE api_keys ENABLE ROW LEVEL SECURITY;
ALTER TABLE connections ENABLE ROW LEVEL SECURITY;
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;

-- RLS Policies
CREATE POLICY workspace_isolation ON transactions
    USING (workspace_id = current_setting('app.workspace_id')::UUID);

CREATE POLICY workspace_isolation ON accounts
    USING (workspace_id = current_setting('app.workspace_id')::UUID);

CREATE POLICY workspace_isolation ON connections
    USING (workspace_id = current_setting('app.workspace_id')::UUID);
```

### 5.3 ClickHouse Schema (Analytics)

```sql
-- Events table (append-only)
CREATE TABLE events (
    event_id UUID,
    workspace_id UUID,

    event_type LowCardinality(String),
    event_source LowCardinality(String), -- 'stripe', 'plaid', 'canvas', 'polar'

    -- Entity references
    entity_type LowCardinality(String), -- 'transaction', 'subscription', 'invoice'
    entity_id String,

    -- Event data
    data String, -- JSON

    -- Timing
    event_timestamp DateTime64(3),
    processed_at DateTime64(3) DEFAULT now64(3),

    -- Partitioning
    event_date Date DEFAULT toDate(event_timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (workspace_id, event_type, event_timestamp)
TTL event_date + INTERVAL 2 YEAR;

-- Transactions fact table
CREATE TABLE transactions_fact (
    transaction_id UUID,
    workspace_id UUID,
    account_id UUID,

    provider LowCardinality(String),

    amount Decimal(19, 4),
    currency LowCardinality(String),

    transaction_date Date,
    transaction_datetime DateTime64(3),

    category LowCardinality(String),
    merchant_name String,

    is_recurring UInt8,
    is_subscription UInt8,

    -- Enrichment scores
    confidence_score Float32,

    -- Timestamps
    created_at DateTime64(3),

    -- Partitioning
    month Date DEFAULT toStartOfMonth(transaction_date)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(month)
ORDER BY (workspace_id, transaction_date, transaction_id);

-- Recovery analytics
CREATE TABLE recovery_events (
    event_id UUID,
    workspace_id UUID,
    workflow_id UUID,
    workflow_run_id UUID,
    customer_id UUID,

    event_type LowCardinality(String), -- 'started', 'email_sent', 'recovered', 'failed'

    -- Recovery details
    original_amount Decimal(19, 4),
    recovered_amount Decimal(19, 4),
    currency LowCardinality(String),

    -- A/B testing
    cohort LowCardinality(String), -- 'control', 'treatment'

    -- Timing
    event_timestamp DateTime64(3),

    -- Partitioning
    event_date Date DEFAULT toDate(event_timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (workspace_id, workflow_id, event_timestamp);

-- Materialized view for recovery rates
CREATE MATERIALIZED VIEW recovery_rates_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (workspace_id, workflow_id, cohort, date)
AS SELECT
    workspace_id,
    workflow_id,
    cohort,
    toDate(event_timestamp) AS date,

    countIf(event_type = 'started') AS attempts,
    countIf(event_type = 'recovered') AS recoveries,
    sumIf(original_amount, event_type = 'started') AS total_at_risk,
    sumIf(recovered_amount, event_type = 'recovered') AS total_recovered
FROM recovery_events
GROUP BY workspace_id, workflow_id, cohort, date;

-- Query: Recovery rate by workflow
-- SELECT
--     workflow_id,
--     cohort,
--     sum(recoveries) / sum(attempts) AS recovery_rate,
--     sum(total_recovered) / sum(total_at_risk) AS amount_recovery_rate
-- FROM recovery_rates_mv
-- WHERE workspace_id = {workspace_id}
--   AND date >= today() - 30
-- GROUP BY workflow_id, cohort;
```

### 5.4 Redis Data Structures

```
# Key naming convention: {service}:{workspace_id}:{entity}:{id}

# Rate limiting (sliding window)
ZSET ratelimit:{workspace_id}:{api_key}:requests
  - score: timestamp (ms)
  - member: request_id
  - TTL: 60s

# Idempotency keys
STRING idempotency:{workspace_id}:{route}:{key}
  - value: JSON response
  - TTL: 24h

# Transaction cache
HASH txn_cache:{workspace_id}:{account_id}
  - field: transaction_id
  - value: JSON transaction
  - TTL: 120s

# Session data
HASH session:{session_id}
  - user_id
  - workspace_id
  - permissions
  - TTL: 7d

# Webhook processing lock
STRING webhook_lock:{workspace_id}:{event_id}
  - value: worker_id
  - TTL: 300s (5 min)
  - SET NX (only if not exists)

# Connection status cache
STRING connection:{workspace_id}:{connection_id}:status
  - value: JSON status
  - TTL: 60s

# Feature flags
HASH features:{workspace_id}
  - field: feature_name
  - value: enabled (1/0)
  - No TTL (persistent)
```

---

## 6. API Specifications

### 6.1 Developer Platform API

#### Base URL & Versioning

```
Production: https://api.oppulence.io/v1
Staging:    https://api.staging.oppulence.io/v1
```

#### Authentication

```http
# API Key Authentication
Authorization: Bearer op_live_xxxxxxxxxxxx

# Required Headers
X-Workspace-Id: ws_xxxxxxxxxxxx
X-Request-Id: req_xxxxxxxxxxxx (optional, for tracing)
Idempotency-Key: idem_xxxxxxxxxxxx (for POST/PUT/DELETE)
```

#### Rate Limits

| Plan | Requests/min | Burst | Concurrent |
|------|-------------|-------|------------|
| Free | 100 | 20 | 5 |
| Pro | 1,000 | 100 | 20 |
| Business | 10,000 | 500 | 50 |
| Enterprise | Custom | Custom | Custom |

#### Endpoints

##### POST /v1/connections

Create a new provider connection.

```http
POST /v1/connections
Content-Type: application/json
Authorization: Bearer op_live_xxxx
X-Workspace-Id: ws_xxxx

{
  "provider": "plaid",
  "public_token": "public-sandbox-xxxx",
  "institution_id": "ins_1",
  "metadata": {
    "user_id": "usr_123"
  }
}
```

**Response (201 Created):**
```json
{
  "id": "conn_xxxxxxxxxxxx",
  "provider": "plaid",
  "status": "active",
  "institution": {
    "id": "ins_1",
    "name": "Chase"
  },
  "accounts": [
    {
      "id": "acc_xxxxxxxxxxxx",
      "name": "Checking",
      "type": "depository",
      "subtype": "checking",
      "mask": "1234",
      "current_balance": 5000.00,
      "available_balance": 4800.00,
      "currency": "USD"
    }
  ],
  "created_at": "2024-11-29T10:00:00Z"
}
```

##### GET /v1/transactions

List transactions with filtering and pagination.

```http
GET /v1/transactions?account_id=acc_xxxx&since=2024-11-01&limit=100&cursor=txn_xxxx
Authorization: Bearer op_live_xxxx
X-Workspace-Id: ws_xxxx
```

**Query Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `account_id` | string | Filter by account |
| `connection_id` | string | Filter by connection |
| `provider` | string | Filter by provider |
| `since` | ISO 8601 date | Start date |
| `until` | ISO 8601 date | End date |
| `status` | string | `pending`, `posted` |
| `category` | string | Transaction category |
| `min_amount` | number | Minimum amount |
| `max_amount` | number | Maximum amount |
| `enrich` | boolean | Include AI enrichment |
| `limit` | integer | Page size (max 500) |
| `cursor` | string | Pagination cursor |

**Response (200 OK):**
```json
{
  "data": [
    {
      "id": "txn_xxxxxxxxxxxx",
      "account_id": "acc_xxxxxxxxxxxx",
      "provider": "plaid",
      "provider_transaction_id": "abc123",

      "amount": -42.50,
      "currency": "USD",

      "date": "2024-11-28",
      "datetime": "2024-11-28T14:30:00Z",

      "name": "AMAZON.COM*123ABC",
      "merchant_name": "Amazon",

      "category": "Shopping",
      "category_id": "19013000",

      "pending": false,

      "enrichment": {
        "merchant": {
          "name": "Amazon",
          "logo_url": "https://...",
          "category": "E-commerce",
          "website": "amazon.com"
        },
        "category": {
          "primary": "Shopping",
          "detailed": "Online Marketplace",
          "confidence": 0.95
        },
        "tax": {
          "deductible": false,
          "category": "personal"
        },
        "recurring": {
          "is_recurring": false
        }
      },

      "created_at": "2024-11-28T15:00:00Z"
    }
  ],
  "pagination": {
    "has_more": true,
    "next_cursor": "txn_yyyyyyyyyyyy",
    "total_count": 1523
  }
}
```

##### POST /v1/enrichment

AI-powered transaction enrichment.

```http
POST /v1/enrichment
Content-Type: application/json
Authorization: Bearer op_live_xxxx
X-Workspace-Id: ws_xxxx

{
  "transactions": [
    {
      "id": "txn_xxxx",
      "name": "AMZN MKTP US*AB1CD2EF3",
      "amount": -156.78,
      "date": "2024-11-28"
    }
  ],
  "enrichment_types": ["merchant", "category", "tax", "recurring"]
}
```

**Response (200 OK):**
```json
{
  "results": [
    {
      "transaction_id": "txn_xxxx",
      "enrichment": {
        "merchant": {
          "name": "Amazon Marketplace",
          "logo_url": "https://logos.oppulence.io/amazon.png",
          "category": "E-commerce",
          "mcc": "5999",
          "confidence": 0.98
        },
        "category": {
          "primary": "Shopping",
          "secondary": "Online Retail",
          "plaid_category": ["Shops", "Supermarkets and Groceries"],
          "confidence": 0.92
        },
        "tax": {
          "deductible": false,
          "category": "personal",
          "requires_receipt": false,
          "confidence": 0.88
        },
        "recurring": {
          "is_recurring": false,
          "frequency": null,
          "confidence": 0.95
        }
      },
      "processed_at": "2024-11-29T10:00:00Z"
    }
  ],
  "usage": {
    "transactions_processed": 1,
    "enrichment_credits_used": 4
  }
}
```

##### POST /v1/webhooks/stripe

Receive Stripe webhooks.

```http
POST /v1/webhooks/stripe
Content-Type: application/json
Stripe-Signature: t=1234567890,v1=xxxxx
X-Workspace-Id: ws_xxxx

{
  "id": "evt_xxxx",
  "type": "invoice.payment_failed",
  "data": {
    "object": { ... }
  }
}
```

**Response (200 OK):**
```json
{
  "received": true,
  "event_id": "evt_xxxx",
  "processed": true
}
```

#### Error Responses

```json
// 400 Bad Request
{
  "error": {
    "type": "validation_error",
    "code": "invalid_parameter",
    "message": "Invalid date format for 'since' parameter",
    "param": "since",
    "doc_url": "https://docs.oppulence.io/errors/invalid_parameter"
  }
}

// 401 Unauthorized
{
  "error": {
    "type": "authentication_error",
    "code": "invalid_api_key",
    "message": "The API key provided is invalid or has been revoked"
  }
}

// 403 Forbidden
{
  "error": {
    "type": "authorization_error",
    "code": "insufficient_permissions",
    "message": "This API key does not have permission to access this resource",
    "required_scope": "transactions:read"
  }
}

// 429 Too Many Requests
{
  "error": {
    "type": "rate_limit_error",
    "code": "rate_limit_exceeded",
    "message": "Rate limit exceeded. Please retry after 60 seconds",
    "retry_after": 60
  }
}

// 500 Internal Server Error
{
  "error": {
    "type": "api_error",
    "code": "internal_error",
    "message": "An internal error occurred. Please try again.",
    "request_id": "req_xxxx"
  }
}
```

### 6.2 OpenAPI Specification

```yaml
openapi: 3.1.0
info:
  title: Oppulence Developer Platform API
  version: 1.0.0
  description: Unified Financial API for SaaS applications

servers:
  - url: https://api.oppulence.io/v1
    description: Production
  - url: https://api.staging.oppulence.io/v1
    description: Staging

security:
  - bearerAuth: []

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: API Key

  parameters:
    WorkspaceId:
      name: X-Workspace-Id
      in: header
      required: true
      schema:
        type: string
        pattern: '^ws_[a-zA-Z0-9]{20}$'

    IdempotencyKey:
      name: Idempotency-Key
      in: header
      required: false
      schema:
        type: string
        maxLength: 255

  schemas:
    Connection:
      type: object
      properties:
        id:
          type: string
          example: conn_xxxxxxxxxxxx
        provider:
          type: string
          enum: [stripe, plaid, gocardless, teller, enablebanking]
        status:
          type: string
          enum: [active, needs_reauth, disconnected, error]
        institution:
          $ref: '#/components/schemas/Institution'
        accounts:
          type: array
          items:
            $ref: '#/components/schemas/Account'
        created_at:
          type: string
          format: date-time

    Transaction:
      type: object
      properties:
        id:
          type: string
        account_id:
          type: string
        amount:
          type: number
          format: decimal
        currency:
          type: string
          minLength: 3
          maxLength: 3
        date:
          type: string
          format: date
        name:
          type: string
        merchant_name:
          type: string
          nullable: true
        category:
          type: string
        pending:
          type: boolean
        enrichment:
          $ref: '#/components/schemas/Enrichment'

    Enrichment:
      type: object
      properties:
        merchant:
          $ref: '#/components/schemas/MerchantEnrichment'
        category:
          $ref: '#/components/schemas/CategoryEnrichment'
        tax:
          $ref: '#/components/schemas/TaxEnrichment'
        recurring:
          $ref: '#/components/schemas/RecurringEnrichment'

paths:
  /connections:
    post:
      summary: Create a connection
      operationId: createConnection
      tags: [Connections]
      parameters:
        - $ref: '#/components/parameters/WorkspaceId'
        - $ref: '#/components/parameters/IdempotencyKey'
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [provider]
              properties:
                provider:
                  type: string
                  enum: [stripe, plaid, gocardless, teller, enablebanking]
                public_token:
                  type: string
                  description: Plaid public token
                institution_id:
                  type: string
                metadata:
                  type: object
      responses:
        '201':
          description: Connection created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Connection'

  /transactions:
    get:
      summary: List transactions
      operationId: listTransactions
      tags: [Transactions]
      parameters:
        - $ref: '#/components/parameters/WorkspaceId'
        - name: account_id
          in: query
          schema:
            type: string
        - name: since
          in: query
          schema:
            type: string
            format: date
        - name: until
          in: query
          schema:
            type: string
            format: date
        - name: limit
          in: query
          schema:
            type: integer
            minimum: 1
            maximum: 500
            default: 100
        - name: cursor
          in: query
          schema:
            type: string
      responses:
        '200':
          description: Transactions list
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Transaction'
                  pagination:
                    type: object
                    properties:
                      has_more:
                        type: boolean
                      next_cursor:
                        type: string
```

---

## 7. Authentication & Authorization

### 7.1 Authentication Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Edge as Edge (CF Worker)
    participant Auth as Auth Service
    participant KV as KV Store
    participant DB as Database

    Client->>Edge: Request with Bearer token

    Note over Edge: Extract token from<br/>Authorization header

    Edge->>KV: Check token cache

    alt Token cached
        KV-->>Edge: Cached API key data
    else Token not cached
        Edge->>Auth: Validate token
        Auth->>DB: Query api_keys table
        DB-->>Auth: API key record
        Auth->>Auth: Verify hash, check expiry
        Auth-->>Edge: Validated key data
        Edge->>KV: Cache for 5 min
    end

    Edge->>Edge: Check workspace match
    Edge->>Edge: Verify scopes/permissions

    alt Authorized
        Edge->>Edge: Process request
        Edge-->>Client: Response
    else Unauthorized
        Edge-->>Client: 401/403 Error
    end
```

### 7.2 API Key Structure

```
op_live_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
│   │    └── 32-character random string (base62)
│   └── Environment: live, test, sandbox
└── Prefix: op (Oppulence)

Full key: 40 characters
Stored hash: SHA-256(key)
Display prefix: op_live_xxxx (first 12 chars)
```

### 7.3 RBAC Model

```mermaid
erDiagram
    USER ||--o{ USER_WORKSPACE : belongs_to
    WORKSPACE ||--o{ USER_WORKSPACE : has
    USER_WORKSPACE ||--o{ ROLE : has
    ROLE ||--o{ PERMISSION : grants
    API_KEY ||--o{ SCOPE : has
    WORKSPACE ||--o{ API_KEY : owns

    USER {
        uuid id PK
        string email
        string name
    }

    WORKSPACE {
        uuid id PK
        string name
        string slug
    }

    USER_WORKSPACE {
        uuid user_id FK
        uuid workspace_id FK
        string role
    }

    ROLE {
        string name PK
        string description
    }

    PERMISSION {
        string resource
        string action
    }

    API_KEY {
        uuid id PK
        uuid workspace_id FK
        string key_hash
        string[] scopes
    }

    SCOPE {
        string name
        string description
    }
```

### 7.4 Permission Matrix

| Role | connections:* | transactions:* | enrichment:* | webhooks:* | settings:* |
|------|--------------|----------------|--------------|------------|------------|
| Owner | ✅ | ✅ | ✅ | ✅ | ✅ |
| Admin | ✅ | ✅ | ✅ | ✅ | ✅ |
| Member | ✅ read | ✅ read | ✅ | ✅ read | ❌ |
| Viewer | ✅ read | ✅ read | ❌ | ❌ | ❌ |
| Service | Scoped | Scoped | Scoped | Scoped | ❌ |

### 7.5 Scope Definitions

```typescript
const SCOPES = {
  // Connections
  'connections:read': 'Read connection status and accounts',
  'connections:write': 'Create, update, delete connections',

  // Transactions
  'transactions:read': 'Read transactions',
  'transactions:write': 'Trigger syncs, update metadata',
  'transactions:sync': 'Trigger transaction sync',

  // Enrichment
  'enrichment:read': 'Read enrichment data',
  'enrichment:write': 'Trigger enrichment',

  // Webhooks
  'webhooks:read': 'Read webhook events',
  'webhooks:write': 'Configure webhook endpoints',

  // Analytics
  'analytics:read': 'Read analytics data',

  // Admin
  'admin:*': 'Full administrative access',
} as const;
```

---

## 8. Security Architecture

### 8.1 Security Layers

```mermaid
flowchart TB
    subgraph Edge["Edge Security (Cloudflare)"]
        WAF["WAF Rules"]
        DDoS["DDoS Protection"]
        Bot["Bot Management"]
        TLS["TLS 1.3"]
    end

    subgraph App["Application Security"]
        Auth["Authentication"]
        AuthZ["Authorization (RBAC)"]
        Validate["Input Validation"]
        RateLimit["Rate Limiting"]
    end

    subgraph Data["Data Security"]
        Encrypt["Encryption at Rest"]
        Transit["Encryption in Transit"]
        Secrets["Secrets Management"]
        Audit["Audit Logging"]
    end

    subgraph Infra["Infrastructure Security"]
        Network["Network Isolation"]
        IAM["IAM Policies"]
        Scan["Vulnerability Scanning"]
        Rotate["Key Rotation"]
    end

    Edge --> App --> Data --> Infra
```

### 8.2 Encryption

#### At Rest

```typescript
// Provider token encryption
import { createCipheriv, createDecipheriv, randomBytes } from 'crypto';

const ALGORITHM = 'aes-256-gcm';
const KEY = Buffer.from(process.env.ENCRYPTION_KEY!, 'hex'); // 32 bytes

interface EncryptedData {
  iv: string;      // 12 bytes, base64
  data: string;    // Encrypted, base64
  tag: string;     // 16 bytes, base64
}

function encrypt(plaintext: string): EncryptedData {
  const iv = randomBytes(12);
  const cipher = createCipheriv(ALGORITHM, KEY, iv);

  let encrypted = cipher.update(plaintext, 'utf8', 'base64');
  encrypted += cipher.final('base64');

  return {
    iv: iv.toString('base64'),
    data: encrypted,
    tag: cipher.getAuthTag().toString('base64'),
  };
}

function decrypt(encrypted: EncryptedData): string {
  const decipher = createDecipheriv(
    ALGORITHM,
    KEY,
    Buffer.from(encrypted.iv, 'base64')
  );
  decipher.setAuthTag(Buffer.from(encrypted.tag, 'base64'));

  let decrypted = decipher.update(encrypted.data, 'base64', 'utf8');
  decrypted += decipher.final('utf8');

  return decrypted;
}
```

#### In Transit

- TLS 1.3 enforced
- HSTS enabled (max-age=31536000)
- Certificate pinning for provider APIs

### 8.3 Secrets Management

```mermaid
flowchart LR
    subgraph Sources["Secret Sources"]
        Env["Environment Variables"]
        KMS["AWS KMS / CF Secrets"]
        Vault["HashiCorp Vault"]
    end

    subgraph Runtime["Runtime"]
        Worker["CF Worker"]
        Service["Go Service"]
        App["Application"]
    end

    subgraph Usage["Usage"]
        Provider["Provider Tokens"]
        API["API Keys"]
        DB["DB Credentials"]
    end

    Sources --> Runtime
    Runtime --> Usage
```

### 8.4 Webhook Security

```typescript
// Stripe webhook signature verification
import Stripe from 'stripe';

async function verifyStripeWebhook(
  payload: string,
  signature: string,
  secret: string
): Promise<Stripe.Event> {
  try {
    return Stripe.webhooks.constructEvent(payload, signature, secret);
  } catch (err) {
    throw new WebhookVerificationError('Invalid signature');
  }
}

// Plaid webhook verification
import { PlaidApi, WebhookVerificationKeyGetRequest } from 'plaid';

async function verifyPlaidWebhook(
  payload: string,
  headers: Record<string, string>
): Promise<boolean> {
  const plaidSignature = headers['plaid-verification'];
  // Verify using Plaid's JWT verification
  // ...
}
```

### 8.5 Audit Logging

```typescript
interface AuditEvent {
  id: string;
  timestamp: string;
  workspace_id: string;
  actor: {
    type: 'user' | 'api_key' | 'system';
    id: string;
    ip_address?: string;
    user_agent?: string;
  };
  action: string;
  resource: {
    type: string;
    id: string;
  };
  changes?: {
    before: Record<string, unknown>;
    after: Record<string, unknown>;
  };
  metadata?: Record<string, unknown>;
}

// Example audit events
const auditEvents = [
  'connection.created',
  'connection.deleted',
  'api_key.created',
  'api_key.revoked',
  'webhook.configured',
  'workspace.settings_updated',
  'user.invited',
  'user.removed',
  'rbac.role_changed',
];
```

---

## 9. Reliability & Observability

### 9.1 SLOs & Error Budgets

| Service | SLI | SLO | Error Budget (30d) |
|---------|-----|-----|-------------------|
| API Gateway | Availability | 99.9% | 43.2 min |
| API Gateway | p99 Latency | < 400ms | N/A |
| Webhook Ingestion | Success Rate | 99.95% | 21.6 min |
| Webhook Ingestion | Latency | < 1s | N/A |
| Data Sync | Freshness | < 30s | N/A |
| Enrichment | Accuracy | > 90% | N/A |

### 9.2 Observability Stack

```mermaid
flowchart TB
    subgraph Collection["Collection"]
        OTel["OpenTelemetry SDK"]
        Prom["Prometheus Client"]
        Logs["Structured Logs (Pino)"]
    end

    subgraph Processing["Processing"]
        Collector["OTel Collector"]
        Loki["Loki"]
        Mimir["Mimir"]
    end

    subgraph Storage["Storage"]
        Tempo["Tempo (Traces)"]
        PromDB["Prometheus (Metrics)"]
        LokiDB["Loki (Logs)"]
    end

    subgraph Visualization["Visualization"]
        Grafana["Grafana"]
        Alerts["Alert Manager"]
    end

    Collection --> Processing --> Storage --> Visualization
```

### 9.3 Key Metrics

```typescript
// Prometheus metrics
import { Counter, Histogram, Gauge } from 'prom-client';

// Request metrics
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'path', 'status', 'workspace_id'],
});

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'path'],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
});

// Business metrics
const transactionsSynced = new Counter({
  name: 'transactions_synced_total',
  help: 'Total transactions synced',
  labelNames: ['provider', 'workspace_id'],
});

const enrichmentAccuracy = new Gauge({
  name: 'enrichment_accuracy',
  help: 'Enrichment accuracy score',
  labelNames: ['enrichment_type'],
});

const recoveryRate = new Gauge({
  name: 'recovery_rate',
  help: 'Payment recovery rate',
  labelNames: ['workflow_id', 'cohort'],
});

// Infrastructure metrics
const dbConnectionPool = new Gauge({
  name: 'db_connection_pool_size',
  help: 'Database connection pool size',
  labelNames: ['database', 'state'],
});

const cacheHitRate = new Gauge({
  name: 'cache_hit_rate',
  help: 'Cache hit rate',
  labelNames: ['cache_type'],
});
```

### 9.4 Distributed Tracing

```typescript
// OpenTelemetry setup
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  serviceName: 'oppulence-api',
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT,
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();

// Custom span example
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('oppulence-api');

async function enrichTransaction(transaction: Transaction) {
  return tracer.startActiveSpan('enrichTransaction', async (span) => {
    try {
      span.setAttribute('transaction.id', transaction.id);
      span.setAttribute('transaction.amount', transaction.amount);

      const enrichment = await callEnrichmentService(transaction);

      span.setAttribute('enrichment.confidence', enrichment.confidence);
      span.setStatus({ code: SpanStatusCode.OK });

      return enrichment;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message,
      });
      throw error;
    } finally {
      span.end();
    }
  });
}
```

### 9.5 Alerting Rules

```yaml
# Prometheus alerting rules
groups:
  - name: oppulence-api
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m]))
          / sum(rate(http_requests_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le)
          ) > 0.4
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High p99 latency"
          description: "p99 latency is {{ $value }}s"

      - alert: WebhookBacklog
        expr: |
          sum(pg_stat_activity_count{datname="sync_engine", state="active"}) > 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Webhook processing backlog"

      - alert: LowRecoveryRate
        expr: |
          sum(rate(recovery_events_total{event_type="recovered"}[24h]))
          / sum(rate(recovery_events_total{event_type="started"}[24h])) < 0.3
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Recovery rate below threshold"
```

---

## 10. Deployment Architecture

### 10.1 Infrastructure Diagram

```mermaid
flowchart TB
    subgraph Internet
        Users["Users / Clients"]
    end

    subgraph Cloudflare["Cloudflare Edge"]
        CF_DNS["DNS"]
        CF_CDN["CDN"]
        CF_WAF["WAF"]
        CF_Workers["Workers"]
        CF_R2["R2 Storage"]
        CF_KV["KV Store"]
        CF_DO["Durable Objects"]
        CF_Queue["Queues"]
    end

    subgraph Vercel["Vercel"]
        Polar_Web["Polar Web"]
        Canvas_Web["Canvas Web"]
    end

    subgraph K8s["Kubernetes Cluster"]
        subgraph Services["Services"]
            CP_Go["Control Plane (Go)"]
            Sync_API["Sync Engine API"]
            Sync_Worker["Sync Workers (3x)"]
            Storage_API["Storage API"]
            UBB_API["Usage Billing API"]
            UBB_Worker["Usage Workers"]
        end

        subgraph Ingress
            Nginx["Nginx Ingress"]
            Cert["Cert Manager"]
        end
    end

    subgraph Data["Data Stores"]
        subgraph Managed["Managed Services"]
            Supabase["Supabase (PG)"]
            PlanetScale["PlanetScale (MySQL)"]
            ClickHouse_Cloud["ClickHouse Cloud"]
            Upstash["Upstash (Redis)"]
        end

        subgraph Self["Self-Managed"]
            Kafka_Cluster["Kafka (MSK)"]
        end
    end

    Users --> Cloudflare
    CF_Workers --> K8s
    CF_Workers --> Managed
    Vercel --> CF_Workers
    K8s --> Managed
    K8s --> Self
```

### 10.2 Kubernetes Manifests

```yaml
# deployment.yaml - Sync Engine
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sync-engine
  namespace: oppulence
  labels:
    app: sync-engine
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sync-engine
  template:
    metadata:
      labels:
        app: sync-engine
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: sync-engine
      containers:
        - name: sync-engine
          image: ghcr.io/oppulence-engineering/sync-engine:latest
          ports:
            - name: http
              containerPort: 3000
            - name: metrics
              containerPort: 9090
          env:
            - name: NODE_ENV
              value: production
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: sync-engine-secrets
                  key: database-url
            - name: CLICKHOUSE_URL
              valueFrom:
                secretKeyRef:
                  name: sync-engine-secrets
                  key: clickhouse-url
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: sync-engine-secrets
                  key: redis-url
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: sync-engine
                topologyKey: kubernetes.io/hostname

---
# hpa.yaml - Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sync-engine
  namespace: oppulence
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sync-engine
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60

---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: sync-engine
  namespace: oppulence
spec:
  selector:
    app: sync-engine
  ports:
    - name: http
      port: 80
      targetPort: 3000
    - name: metrics
      port: 9090
      targetPort: 9090
  type: ClusterIP

---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sync-engine
  namespace: oppulence
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - sync.oppulence.io
      secretName: sync-engine-tls
  rules:
    - host: sync.oppulence.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: sync-engine
                port:
                  number: 80
```

### 10.3 Helm Chart Structure

```
charts/oppulence/
├── Chart.yaml
├── values.yaml
├── values-staging.yaml
├── values-production.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── hpa.yaml
│   ├── pdb.yaml
│   ├── serviceaccount.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── tests/
│       └── test-connection.yaml
└── charts/
    ├── postgresql/
    ├── redis/
    └── clickhouse/
```

### 10.4 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Run tests
        run: pnpm test

      - name: Run linting
        run: pnpm lint

      - name: Type check
        run: pnpm typecheck

  build:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    outputs:
      image_tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}

      - name: Deploy to staging
        run: |
          helm upgrade --install oppulence ./charts/oppulence \
            -f ./charts/oppulence/values-staging.yaml \
            --set image.tag=${{ github.sha }} \
            --namespace oppulence-staging \
            --wait --timeout 5m

      - name: Run smoke tests
        run: |
          ./scripts/smoke-test.sh https://api.staging.oppulence.io

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_PRODUCTION }}

      - name: Deploy to production
        run: |
          helm upgrade --install oppulence ./charts/oppulence \
            -f ./charts/oppulence/values-production.yaml \
            --set image.tag=${{ github.sha }} \
            --namespace oppulence \
            --wait --timeout 10m

      - name: Notify deployment
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Deployed ${{ github.sha }} to production"
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 11. Performance & Scaling

### 11.1 Performance Targets

| Operation | Target | Current |
|-----------|--------|---------|
| API p50 latency | < 50ms | ~35ms |
| API p99 latency | < 400ms | ~280ms |
| Webhook processing | < 1s | ~500ms |
| Transaction sync | < 30s freshness | ~15s |
| Enrichment (single) | < 500ms | ~350ms |
| Enrichment (batch 100) | < 5s | ~3s |
| Connection setup | < 10s | ~6s |

### 11.2 Caching Strategy

```mermaid
flowchart TB
    subgraph L1["L1: Edge Cache (CF)"]
        KV["KV Store"]
        DO["Durable Objects"]
    end

    subgraph L2["L2: Distributed Cache (Redis)"]
        Redis_Main["Redis Primary"]
        Redis_Read["Redis Replicas"]
    end

    subgraph L3["L3: Database"]
        PG_Primary["PostgreSQL Primary"]
        PG_Read["Read Replicas"]
    end

    Request --> L1
    L1 -->|Miss| L2
    L2 -->|Miss| L3
    L3 -->|Populate| L2
    L2 -->|Populate| L1
```

### 11.3 Cache Configuration

```typescript
// Cache TTLs and strategies
const CACHE_CONFIG = {
  // L1 - Edge cache (Cloudflare KV)
  edge: {
    workspace_config: {
      ttl: 300, // 5 min
      staleWhileRevalidate: 60,
    },
    api_key_validation: {
      ttl: 300,
      staleWhileRevalidate: 30,
    },
  },

  // L2 - Redis cache
  redis: {
    transaction_list: {
      ttl: 120, // 2 min
      keyPattern: 'txn:{workspace_id}:{account_id}:list',
    },
    account_balance: {
      ttl: 60, // 1 min
      keyPattern: 'balance:{workspace_id}:{account_id}',
    },
    enrichment_result: {
      ttl: 86400, // 24h
      keyPattern: 'enrich:{transaction_hash}',
    },
    connection_status: {
      ttl: 60,
      keyPattern: 'conn:{workspace_id}:{connection_id}:status',
    },
  },

  // Invalidation triggers
  invalidation: {
    'transaction.created': ['transaction_list'],
    'transaction.updated': ['transaction_list', 'enrichment_result'],
    'account.balance_updated': ['account_balance'],
    'connection.status_changed': ['connection_status'],
  },
};

// Cache-aside pattern implementation
async function getTransactions(
  workspaceId: string,
  accountId: string,
  options: QueryOptions
): Promise<Transaction[]> {
  const cacheKey = `txn:${workspaceId}:${accountId}:list:${hashOptions(options)}`;

  // Check L2 cache
  const cached = await redis.get(cacheKey);
  if (cached) {
    metrics.cacheHit.inc({ cache: 'redis', type: 'transaction_list' });
    return JSON.parse(cached);
  }

  metrics.cacheMiss.inc({ cache: 'redis', type: 'transaction_list' });

  // Query database
  const transactions = await db.transaction.findMany({
    where: {
      workspaceId,
      accountId,
      date: { gte: options.since, lte: options.until },
    },
    orderBy: { date: 'desc' },
    take: options.limit,
  });

  // Populate cache
  await redis.setex(cacheKey, CACHE_CONFIG.redis.transaction_list.ttl, JSON.stringify(transactions));

  return transactions;
}
```

### 11.4 Database Optimization

```sql
-- Partitioning for large tables
CREATE TABLE transactions (
    id UUID NOT NULL,
    workspace_id UUID NOT NULL,
    account_id UUID,
    -- ... other columns
    date DATE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (date);

-- Create monthly partitions
CREATE TABLE transactions_2024_11 PARTITION OF transactions
    FOR VALUES FROM ('2024-11-01') TO ('2024-12-01');

CREATE TABLE transactions_2024_12 PARTITION OF transactions
    FOR VALUES FROM ('2024-12-01') TO ('2025-01-01');

-- Indexes on partitioned tables
CREATE INDEX idx_transactions_workspace_date ON transactions (workspace_id, date DESC);
CREATE INDEX idx_transactions_account_date ON transactions (account_id, date DESC);

-- Partial indexes for common queries
CREATE INDEX idx_transactions_pending ON transactions (workspace_id, date DESC)
    WHERE pending = true;

CREATE INDEX idx_transactions_failed ON transactions (workspace_id, date DESC)
    WHERE status = 'failed';

-- Connection pooling settings (PgBouncer)
-- pgbouncer.ini
[databases]
oppulence = host=localhost port=5432 dbname=oppulence

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 50
min_pool_size = 10
reserve_pool_size = 10
reserve_pool_timeout = 3
max_db_connections = 100
```

### 11.5 Scaling Strategy

```mermaid
flowchart TB
    subgraph Horizontal["Horizontal Scaling"]
        API_Scale["API: Auto-scale 3-20 pods"]
        Worker_Scale["Workers: Scale with queue depth"]
        Cache_Scale["Redis: Cluster mode"]
    end

    subgraph Vertical["Vertical Scaling"]
        DB_Read["DB: Add read replicas"]
        CH_Resources["ClickHouse: Increase resources"]
    end

    subgraph Partition["Data Partitioning"]
        Time_Part["Time-based partitioning"]
        Tenant_Part["Tenant sharding (future)"]
    end

    Horizontal --> Partition
    Vertical --> Partition
```

---

## 12. Failure Modes & Recovery

### 12.1 Failure Scenarios

```mermaid
flowchart TB
    subgraph Scenarios["Failure Scenarios"]
        Provider["Provider Outage<br/>(Stripe, Plaid)"]
        DB["Database Failure"]
        Cache["Cache Failure"]
        Worker["Worker Crash"]
        Edge["Edge Failure"]
    end

    subgraph Mitigations["Mitigations"]
        Circuit["Circuit Breakers"]
        Fallback["Fallback Modes"]
        Retry["Retry with Backoff"]
        DLQ["Dead Letter Queue"]
        Multi["Multi-Region"]
    end

    Scenarios --> Mitigations
```

### 12.2 Circuit Breaker Implementation

```typescript
import CircuitBreaker from 'opossum';

// Stripe API circuit breaker
const stripeBreaker = new CircuitBreaker(
  async (operation: () => Promise<unknown>) => operation(),
  {
    timeout: 10000,          // 10s timeout
    errorThresholdPercentage: 50,  // Open at 50% errors
    resetTimeout: 30000,     // Try again after 30s
    volumeThreshold: 10,     // Min requests before calculating
  }
);

stripeBreaker.on('open', () => {
  logger.warn('Stripe circuit breaker opened');
  metrics.circuitBreakerState.set({ service: 'stripe' }, 1);
  alerting.notify('stripe-circuit-open');
});

stripeBreaker.on('halfOpen', () => {
  logger.info('Stripe circuit breaker half-open');
  metrics.circuitBreakerState.set({ service: 'stripe' }, 0.5);
});

stripeBreaker.on('close', () => {
  logger.info('Stripe circuit breaker closed');
  metrics.circuitBreakerState.set({ service: 'stripe' }, 0);
});

// Usage
async function createPaymentIntent(params: PaymentIntentParams) {
  return stripeBreaker.fire(async () => {
    return stripe.paymentIntents.create(params);
  });
}
```

### 12.3 Retry Strategy

```typescript
import { retry, exponentialDelay, handleAll } from 'cockatiel';

// Retry policy with exponential backoff + jitter
const retryPolicy = retry(handleAll, {
  maxAttempts: 5,
  backoff: exponentialDelay({
    initialDelay: 1000,
    maxDelay: 30000,
    exponent: 2,
  }),
});

// Webhook processing with retry
async function processWebhook(event: WebhookEvent) {
  return retryPolicy.execute(async (context) => {
    logger.info(`Processing webhook attempt ${context.attempt}`, {
      eventId: event.id,
      type: event.type,
    });

    try {
      await handleWebhookEvent(event);
    } catch (error) {
      if (isRetryable(error)) {
        throw error; // Will be retried
      }
      // Non-retryable error - send to DLQ
      await sendToDLQ(event, error);
    }
  });
}

function isRetryable(error: Error): boolean {
  // Retry on transient errors
  const retryableCodes = [
    'ECONNRESET',
    'ETIMEDOUT',
    'ECONNREFUSED',
    'rate_limit',
    'lock_timeout',
  ];
  return retryableCodes.some(code => error.message.includes(code));
}
```

### 12.4 Dead Letter Queue

```typescript
// DLQ schema
interface DLQMessage {
  id: string;
  originalQueue: string;
  payload: unknown;
  error: {
    message: string;
    stack?: string;
    code?: string;
  };
  attempts: number;
  firstFailedAt: Date;
  lastFailedAt: Date;
  metadata: Record<string, unknown>;
}

// DLQ processing
async function processDLQ() {
  const messages = await db.dlqMessage.findMany({
    where: {
      status: 'pending',
      lastFailedAt: {
        lt: new Date(Date.now() - 3600000), // 1 hour ago
      },
    },
    orderBy: { firstFailedAt: 'asc' },
    take: 100,
  });

  for (const message of messages) {
    try {
      await reprocessMessage(message);
      await db.dlqMessage.update({
        where: { id: message.id },
        data: { status: 'processed' },
      });
    } catch (error) {
      await db.dlqMessage.update({
        where: { id: message.id },
        data: {
          attempts: { increment: 1 },
          lastFailedAt: new Date(),
          error: serializeError(error),
        },
      });
    }
  }
}

// DLQ replay CLI
// npx ts-node scripts/dlq-replay.ts --queue webhooks --since 2024-11-28
```

### 12.5 Disaster Recovery

```mermaid
flowchart TB
    subgraph Primary["Primary Region (us-east)"]
        API_Primary["API"]
        DB_Primary["Database (Primary)"]
        Cache_Primary["Cache"]
    end

    subgraph DR["DR Region (us-west)"]
        API_DR["API (Standby)"]
        DB_DR["Database (Replica)"]
        Cache_DR["Cache"]
    end

    subgraph Backups["Backups"]
        S3_Backup["S3 Backup"]
        CH_Backup["ClickHouse Backup"]
    end

    DB_Primary -->|Streaming Replication| DB_DR
    DB_Primary -->|Daily Backup| S3_Backup

    subgraph RTO_RPO["Recovery Targets"]
        RTO["RTO: 1 hour"]
        RPO["RPO: 5 minutes"]
    end
```

---

## 13. Development Guidelines

### 13.1 Code Style

```typescript
// TypeScript configuration (tsconfig.json)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,
    "skipLibCheck": true
  }
}

// ESLint configuration
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/strict-type-checked",
    "plugin:@typescript-eslint/stylistic-type-checked"
  ],
  "rules": {
    "@typescript-eslint/no-unused-vars": ["error", { "argsIgnorePattern": "^_" }],
    "@typescript-eslint/explicit-function-return-type": "error",
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/prefer-nullish-coalescing": "error"
  }
}
```

### 13.2 Error Handling

```typescript
// Custom error classes
class AppError extends Error {
  constructor(
    public code: string,
    message: string,
    public statusCode: number = 500,
    public isOperational: boolean = true,
    public metadata?: Record<string, unknown>
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class ValidationError extends AppError {
  constructor(message: string, public field?: string) {
    super('VALIDATION_ERROR', message, 400);
  }
}

class AuthenticationError extends AppError {
  constructor(message: string = 'Authentication required') {
    super('AUTHENTICATION_ERROR', message, 401);
  }
}

class AuthorizationError extends AppError {
  constructor(message: string = 'Insufficient permissions') {
    super('AUTHORIZATION_ERROR', message, 403);
  }
}

class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super('NOT_FOUND', `${resource} with id ${id} not found`, 404);
  }
}

class RateLimitError extends AppError {
  constructor(public retryAfter: number) {
    super('RATE_LIMIT_EXCEEDED', 'Rate limit exceeded', 429);
  }
}

class ProviderError extends AppError {
  constructor(
    provider: string,
    originalError: Error,
    public retryable: boolean = false
  ) {
    super(
      'PROVIDER_ERROR',
      `${provider} error: ${originalError.message}`,
      502,
      true,
      { provider, originalError: originalError.message }
    );
  }
}

// Global error handler
function errorHandler(err: Error, req: Request, res: Response, next: NextFunction) {
  if (err instanceof AppError) {
    logger.warn('Operational error', {
      code: err.code,
      message: err.message,
      statusCode: err.statusCode,
      metadata: err.metadata,
    });

    return res.status(err.statusCode).json({
      error: {
        type: err.code.toLowerCase(),
        code: err.code,
        message: err.message,
        ...(err.metadata && { details: err.metadata }),
      },
    });
  }

  // Unexpected error
  logger.error('Unexpected error', {
    error: err.message,
    stack: err.stack,
  });

  return res.status(500).json({
    error: {
      type: 'internal_error',
      code: 'INTERNAL_ERROR',
      message: 'An unexpected error occurred',
      request_id: req.id,
    },
  });
}
```

### 13.3 Testing Strategy

```typescript
// Unit test example
import { describe, it, expect, vi } from 'vitest';
import { enrichTransaction } from './enrichment.service';

describe('EnrichmentService', () => {
  describe('enrichTransaction', () => {
    it('should enrich transaction with merchant data', async () => {
      const transaction = {
        id: 'txn_123',
        name: 'AMZN MKTP US*AB1CD2EF3',
        amount: -156.78,
        date: '2024-11-28',
      };

      const result = await enrichTransaction(transaction);

      expect(result.merchant).toEqual({
        name: 'Amazon Marketplace',
        category: 'E-commerce',
        confidence: expect.any(Number),
      });
      expect(result.merchant.confidence).toBeGreaterThan(0.9);
    });

    it('should handle unknown merchants gracefully', async () => {
      const transaction = {
        id: 'txn_456',
        name: 'UNKNOWN MERCHANT XYZ',
        amount: -50.00,
        date: '2024-11-28',
      };

      const result = await enrichTransaction(transaction);

      expect(result.merchant.name).toBeNull();
      expect(result.merchant.confidence).toBeLessThan(0.5);
    });
  });
});

// Integration test example
import { describe, it, beforeAll, afterAll } from 'vitest';
import { createTestClient } from './test-utils';

describe('Transactions API', () => {
  let client: TestClient;
  let workspaceId: string;

  beforeAll(async () => {
    client = await createTestClient();
    workspaceId = await client.createWorkspace();
    await client.createConnection(workspaceId, 'stripe');
  });

  afterAll(async () => {
    await client.cleanup();
  });

  it('should list transactions', async () => {
    const response = await client.get(`/v1/transactions`, {
      headers: { 'X-Workspace-Id': workspaceId },
    });

    expect(response.status).toBe(200);
    expect(response.data).toHaveProperty('data');
    expect(response.data).toHaveProperty('pagination');
  });

  it('should filter by date range', async () => {
    const response = await client.get(`/v1/transactions`, {
      params: {
        since: '2024-11-01',
        until: '2024-11-30',
      },
      headers: { 'X-Workspace-Id': workspaceId },
    });

    expect(response.status).toBe(200);
    response.data.data.forEach((txn: Transaction) => {
      expect(new Date(txn.date)).toBeGreaterThanOrEqual(new Date('2024-11-01'));
      expect(new Date(txn.date)).toBeLessThanOrEqual(new Date('2024-11-30'));
    });
  });
});
```

### 13.4 Local Development

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: oppulence
      POSTGRES_PASSWORD: oppulence
      POSTGRES_DB: oppulence
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./scripts/init-db.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U oppulence"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

  clickhouse:
    image: clickhouse/clickhouse-server:24.3
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse_data:/var/lib/clickhouse
      - ./clickhouse/config.xml:/etc/clickhouse-server/config.d/config.xml
    healthcheck:
      test: ["CMD", "clickhouse-client", "--query", "SELECT 1"]
      interval: 5s
      timeout: 5s
      retries: 5

  minio:
    image: minio/minio:latest
    ports:
      - "9002:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    command: server /data --console-address ":9001"
    volumes:
      - minio_data:/data

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8889:8889"   # Prometheus exporter
    volumes:
      - ./otel/config.yaml:/etc/otelcol/config.yaml
    command: ["--config=/etc/otelcol/config.yaml"]

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning

volumes:
  postgres_data:
  redis_data:
  clickhouse_data:
  minio_data:
  grafana_data:
```

---

## 14. Open Items & Roadmap

### 14.1 Current Priorities (Q4 2024)

| Priority | Item | Owner | Status |
|----------|------|-------|--------|
| P0 | Ship Developer Platform MVP | Platform | In Progress |
| P0 | Wire Canvas to Developer Platform | Apps | Not Started |
| P1 | Production deployment pipeline | DevOps | In Progress |
| P1 | Monitoring & alerting setup | DevOps | Not Started |
| P2 | Plaid integration testing | Platform | Not Started |
| P2 | Recovery workflow optimization | Apps | Not Started |

### 14.2 Future Roadmap

```mermaid
gantt
    title Oppulence Roadmap 2024-2025
    dateFormat  YYYY-MM-DD

    section Platform
    Developer Platform MVP       :2024-11-29, 30d
    Canvas Integration          :2025-01-01, 30d
    Polar Integration           :2025-02-01, 30d
    External Developer Launch   :2025-03-01, 60d

    section Features
    Usage-Based Billing         :2025-02-01, 45d
    Accounting Sync (QB/Xero)   :2025-03-15, 60d
    Banking Reconciliation      :2025-05-01, 45d

    section Infrastructure
    Multi-Region Deployment     :2025-04-01, 30d
    SOC2 Compliance            :2025-05-01, 90d
```

### 14.3 Technical Debt

- [ ] Migrate Polar from single-tenant to multi-tenant architecture
- [ ] Consolidate authentication across services (shared auth service)
- [ ] Implement proper feature flag system
- [ ] Add comprehensive E2E test suite
- [ ] Document all internal APIs

### 14.4 Security Enhancements

- [ ] Implement DLP for enrichment logs
- [ ] Add anomaly detection for API usage
- [ ] Complete threat model documentation
- [ ] SOC2 Type II certification preparation

---

## Quick Links

| Resource | Link |
|----------|------|
| Developer Platform | [oppulence-developer-platform](https://github.com/Oppulence-Engineering/oppulence-developer-platform) |
| Polar (Billing) | [oppulence](https://github.com/Oppulence-Engineering/oppulence) |
| Canvas (Recovery) | [oppulence-canvas](https://github.com/Oppulence-Engineering/oppulence-canvas) |
| Sync Engine | [oppulence-sync-engine](https://github.com/Oppulence-Engineering/oppulence-sync-engine) |
| Storage | [oppulence-storage](https://github.com/Oppulence-Engineering/oppulence-storage) |
| Usage Billing | [oppulence-usage-based-billing](https://github.com/Oppulence-Engineering/oppulence-usaged-based-billing) |

---

*This RFC is the source of truth for Oppulence architecture. Update alongside system changes.*

*Last updated: 2024-11-29*
