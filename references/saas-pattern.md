# SaaS Pattern Reference

## Core Architecture

```
[React/TS or Blazor] → [REST / tRPC API] → [Postgres] → [Stripe]
                              ↓
                       [Auth Layer]
                              ↓
                      [Tenant Isolation]
```

## Multi-Tenancy Strategy

Default: **Schema-per-tenant** (Postgres)
- Each tenant gets isolated schema: `tenant_{id}.users`, `tenant_{id}.data`
- Connection pool with schema-aware middleware
- Fallback: Row-level tenancy via `tenant_id` column + RLS policies

## Auth Stack

- **Node.js / Python**: Better Auth or Lucia Auth (JWT + refresh tokens)
- **.NET**: ASP.NET Core Identity + JWT Bearer
- Session storage: Postgres-backed (not Redis unless user requests)
- MFA: TOTP via `otpauth` (Node) or `OtpNet` (.NET)

## RBAC Model

```sql
-- Roles: owner | admin | member | viewer
-- Stored per-tenant
CREATE TABLE tenant_{id}.role_assignments (
  user_id UUID,
  role    TEXT CHECK (role IN ('owner','admin','member','viewer')),
  PRIMARY KEY (user_id)
);
```

## Billing (Stripe)

Boilerplate sequence:
1. Customer created on org signup → `stripe.customers.create`
2. Subscription attached to plan → `stripe.subscriptions.create`
3. Webhook handler for: `invoice.paid`, `customer.subscription.deleted`, `payment_intent.payment_failed`
4. Usage metering if DaaS/AIaaS hybrid → `stripe.billing.meterEvents.create`

## Standard Folder Layout (Node.js SaaS)

```
/
├── apps/
│   ├── web/          # React + TypeScript (Vite)
│   └── api/          # Node.js / Fastify
├── packages/
│   ├── db/           # Postgres client + migrations
│   ├── auth/         # Auth logic
│   └── stripe/       # Billing helpers
├── docker-compose.yml
├── .env.example
└── github/workflows/ci.yml
```

## Standard Folder Layout (.NET SaaS)

```
/
├── src/
│   ├── Api/          # .NET 8 Minimal API
│   ├── Domain/       # Entities + interfaces
│   ├── Infrastructure/ # EF Core, Stripe, Auth
│   └── Web/          # Blazor WASM or React
├── tests/
│   └── Api.Tests/
├── docker-compose.yml
└── .github/workflows/ci.yml
```

## Key Migrations (Postgres)

```sql
-- V1: Core tenant table
CREATE TABLE tenants (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name       TEXT NOT NULL,
  slug       TEXT UNIQUE NOT NULL,
  stripe_id  TEXT,
  plan       TEXT DEFAULT 'free',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- V2: Users
CREATE TABLE users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id   UUID REFERENCES tenants(id) ON DELETE CASCADE,
  email       TEXT UNIQUE NOT NULL,
  password_hash TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

## GitHub Actions CI (SaaS)

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20' }
      - run: npm ci
      - run: npm test
      - run: npx playwright test
```
