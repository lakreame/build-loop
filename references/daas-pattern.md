# DaaS Pattern Reference

## Core Architecture

```
[Ingest] → [Transform] → [Postgres] → [OpenAPI / Scalar] → [Consumer]
               ↓
          [Validation]
               ↓
          [Rate Limiting]
```

## Ingestion Patterns

**Batch (scheduled)**
- Python + `schedule` or Node.js + cron
- Pull from: CSV, REST APIs, S3, webhooks
- Write to staging table → validate → promote to clean table

**Streaming (real-time)**
- HTTP webhook endpoint → queue (pg-queue or BullMQ) → worker
- Idempotency key on every ingest record

**Staging → Clean pattern (Postgres)**:
```sql
CREATE TABLE ingest_raw (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source       TEXT NOT NULL,
  payload      JSONB NOT NULL,
  received_at  TIMESTAMPTZ DEFAULT NOW(),
  processed    BOOLEAN DEFAULT FALSE
);

CREATE TABLE data_clean (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  raw_id       UUID REFERENCES ingest_raw(id),
  -- typed columns here
  created_at   TIMESTAMPTZ DEFAULT NOW()
);
```

## Transformation

- Python: pandas / polars for batch; pure SQL CTEs for in-DB transforms
- Node.js: Zod schema validation before write
- Always: reject-and-log invalid records, never silently drop

## API Layer

**OpenAPI-first design:**
1. Write `openapi.yaml` first
2. Generate server stubs (openapi-generator or `@fastify/swagger`)
3. Scalar for hosted docs UI

**Rate Limiting:**
- Token bucket per API key (stored in Postgres or memory)
- Node.js: `@fastify/rate-limit`
- .NET: `AspNetCoreRateLimit`
- Python: `slowapi`

**API Key Management:**
```sql
CREATE TABLE api_keys (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id   UUID REFERENCES tenants(id),
  key_hash    TEXT UNIQUE NOT NULL,  -- bcrypt hash
  label       TEXT,
  rate_limit  INTEGER DEFAULT 1000,  -- requests/hour
  last_used   TIMESTAMPTZ,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

## Standard Folder Layout (Python DaaS)

```
/
├── ingestion/
│   ├── sources/      # One file per data source
│   ├── validators/   # Pydantic models for each source
│   └── workers/      # Queue consumers
├── api/
│   ├── routes/       # FastAPI routers
│   ├── schemas/      # Response models
│   └── openapi.yaml  # Spec-first source of truth
├── db/
│   └── migrations/   # Flyway SQL migrations
├── docker-compose.yml
└── .github/workflows/ci.yml
```

## Scalar Docs Setup (.NET)

```csharp
// Program.cs
builder.Services.AddOpenApi();
app.MapOpenApi();
app.MapScalarApiReference(); // NuGet: Scalar.AspNetCore
```

## Scalar Docs Setup (Node.js / Fastify)

```ts
import fastifySwagger from '@fastify/swagger'
import scalar from '@scalar/fastify-api-reference'

await app.register(fastifySwagger, { openapi: { info: { title: 'DaaS API', version: '1.0' } } })
await app.register(scalar, { routePrefix: '/docs' })
```

## Key Metrics to Expose

Always include a `/metrics` or `/health` endpoint:
```json
{
  "status": "ok",
  "ingest_lag_seconds": 4,
  "records_today": 140200,
  "api_requests_hour": 3421,
  "db_connections": 12
}
```
