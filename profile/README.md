# Oppulence Engineering

> Revenue Operations OS - The first platform managing the complete revenue lifecycle from customer acquisition through payment recovery.

## What We're Building

Oppulence is a **Platform + Apps** architecture for SMB SaaS companies:

```
┌──────────────────────────────────────────────────┐
│         DEVELOPER PLATFORM (Foundation)          │
│   Unified Financial API: Stripe, Plaid, Banks    │
└──────────────────────────────────────────────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │  POLAR   │   │  CANVAS  │   │  FUTURE  │
   │ Billing  │   │ Dunning  │   │   APPS   │
   └──────────┘   └──────────┘   └──────────┘
```

## Repositories

| Repository | Purpose | Stack |
|------------|---------|-------|
| [oppulence-developer-platform](https://github.com/Oppulence-Engineering/oppulence-developer-platform) | Unified Financial API | Go + TypeScript |
| [oppulence](https://github.com/Oppulence-Engineering/oppulence) | Billing/Storefront | Python/FastAPI |
| [oppulence-canvas](https://github.com/Oppulence-Engineering/oppulence-canvas) | Dunning/Recovery | TypeScript/Next.js |
| [oppulence-sync-engine](https://github.com/Oppulence-Engineering/oppulence-sync-engine) | Stripe Data Sync | TypeScript/Fastify |

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full technical architecture.

---

*Building the financial infrastructure for the autonomous economy.*
