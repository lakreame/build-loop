# Infrastructure Security Reference

## Docker

### Multi-Stage Build (Minimal Attack Surface)
```dockerfile
# ✅ Build stage — full toolchain
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# ✅ Runtime stage — no build tools, no source
FROM node:20-alpine AS runtime
WORKDIR /app
# Run as non-root user
RUN addgroup -S app && adduser -S app -G app
USER app
COPY --from=builder --chown=app:app /app/dist ./dist
COPY --from=builder --chown=app:app /app/node_modules ./node_modules
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Hardened docker-compose.yml
```yaml
services:
  api:
    build: .
    read_only: true                  # Immutable filesystem
    tmpfs: [/tmp]                    # Allow temp writes only here
    cap_drop: [ALL]                  # Drop all Linux capabilities
    cap_add: [NET_BIND_SERVICE]      # Add only what's needed
    security_opt: [no-new-privileges:true]
    environment:
      NODE_ENV: production
      DATABASE_URL: ${DATABASE_URL}  # Injected — never hardcoded
    ports:
      - "127.0.0.1:3000:3000"       # Bind to localhost only
    networks: [backend]

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks: [backend]
    # Do NOT expose port 5432 to host in production

networks:
  backend:
    driver: bridge
    internal: true   # No external internet access for backend network

volumes:
  pgdata:
```

### Docker Image Scanning
```yaml
# Add to CI pipeline
- name: Scan image for vulnerabilities
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: ${{ env.IMAGE }}
    format: table
    exit-code: 1
    severity: CRITICAL,HIGH
```

---

## GitHub Actions — Secrets & CI Security

### Secrets Management
```yaml
# ✅ Always use GitHub Secrets — never hardcode in YAML
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  STRIPE_SECRET_KEY: ${{ secrets.STRIPE_SECRET_KEY }}

# ❌ CRITICAL — never this
env:
  API_KEY: "sk-live-abc123"
```

### Least-Privilege Permissions
```yaml
permissions:
  contents: read       # Only what the job needs
  packages: write      # For pushing to GHCR

# Or restrict at workflow level
permissions: read-all
```

### Prevent Secret Leakage in Logs
```yaml
- name: Deploy
  run: |
    # Mask any values that might appear in logs
    echo "::add-mask::${{ secrets.API_KEY }}"
    ./deploy.sh
```

### Pin Actions to SHA (Supply Chain)
```yaml
# ❌ Tag-based — tag can be moved
uses: actions/checkout@v4

# ✅ SHA-pinned — immutable
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
```

### Branch Protection Rules (via GitHub Settings)
```
Required: ✅
- Require pull request reviews (min 1 approver)
- Require status checks to pass (CI must be green)
- Require branches to be up to date
- Restrict force pushes
- Restrict deletions
```

### Dependency Review on PRs
```yaml
name: Dependency Review
on: [pull_request]
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: high
```

### Secret Scanning
```yaml
# Enable in GitHub repo settings: Security → Secret scanning → Alert
# Also add pre-commit hook locally:
# pip install detect-secrets
# detect-secrets scan > .secrets.baseline
```

---

## Vercel

### Environment Variables
- Store all secrets in Vercel Dashboard → Settings → Environment Variables
- Use different values per environment (Preview vs Production)
- Never commit `.env.local` files

### Edge Function Security Headers
```ts
// middleware.ts — apply security headers on every response
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const response = NextResponse.next()

  response.headers.set('X-Content-Type-Options', 'nosniff')
  response.headers.set('X-Frame-Options', 'DENY')
  response.headers.set('X-XSS-Protection', '1; mode=block')
  response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin')
  response.headers.set('Permissions-Policy', 'camera=(), microphone=(), geolocation=()')
  response.headers.set(
    'Content-Security-Policy',
    "default-src 'self'; script-src 'self'; connect-src 'self' https://api.example.com"
  )

  return response
}

export const config = { matcher: '/((?!_next/static|favicon.ico).*)', }
```

### vercel.json Headers
```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "Strict-Transport-Security", "value": "max-age=63072000; includeSubDomains; preload" }
      ]
    }
  ]
}
```

---

## PostgreSQL (Production Hardening)

```sql
-- Disable superuser for app account (already in backend-security.md)
-- Additional hardening:

-- Revoke public schema access from public role
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE myapp FROM PUBLIC;

-- Enable SSL connections only
-- In postgresql.conf:
-- ssl = on
-- ssl_cert_file = 'server.crt'
-- ssl_key_file = 'server.key'

-- Audit logging
ALTER SYSTEM SET log_connections = 'on';
ALTER SYSTEM SET log_disconnections = 'on';
ALTER SYSTEM SET log_statement = 'ddl';  -- or 'all' for full audit
SELECT pg_reload_conf();
```

---

## Security Headers Checklist (HTTP)

| Header | Required Value |
|--------|---------------|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` |
| `Content-Security-Policy` | Strict per-app policy (no `unsafe-eval`) |
| `X-Frame-Options` | `DENY` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Disable unused APIs |
| `Cache-Control` (API) | `no-store` for auth/sensitive endpoints |

Verify with: https://securityheaders.com or `curl -I https://yourdomain.com`
