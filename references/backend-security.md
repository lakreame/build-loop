# Backend Security Reference

## Injection

### SQL Injection
- **Never** interpolate user input into SQL strings
- Node.js: use parameterised queries (`pg`: `$1, $2`) or an ORM (Drizzle, Prisma)
- Python: SQLAlchemy parameterised, never `f"SELECT * FROM users WHERE id={id}"`
- .NET: EF Core parameterised or `SqlParameter` — never string concat in raw SQL

```ts
// ✅ Safe (Node.js / pg)
const result = await db.query('SELECT * FROM users WHERE id = $1', [userId])

// ❌ CRITICAL
const result = await db.query(`SELECT * FROM users WHERE id = '${userId}'`)
```

```python
# ✅ Safe (SQLAlchemy)
stmt = select(User).where(User.id == user_id)

# ❌ CRITICAL
db.execute(f"SELECT * FROM users WHERE id = '{user_id}'")
```

### NoSQL / JSON Injection
- Validate and type-check all JSONB inputs before storing
- Whitelist allowed keys; reject unknown fields

### Command Injection
- Never pass user input to `exec`, `spawn`, `subprocess.run(..., shell=True)`
- Use argument arrays, not shell strings: `spawn('cmd', [arg1, arg2])`

---

## Authentication (Backend)

### Token Security
```ts
// ✅ Node.js — sign with env secret, short expiry
import jwt from 'jsonwebtoken'

const token = jwt.sign(
  { sub: userId, tenantId },
  process.env.JWT_SECRET!,   // ≥256 bits, never hardcoded
  { expiresIn: '15m' }
)
```

```csharp
// ✅ .NET — JWT Bearer
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(opts => {
        opts.TokenValidationParameters = new() {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"]!)),
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    });
```

### Password Hashing
```ts
import bcrypt from 'bcrypt'
const ROUNDS = 12

const hash   = await bcrypt.hash(password, ROUNDS)
const valid  = await bcrypt.compare(password, hash)
```

```python
from passlib.context import CryptContext
pwd_ctx = CryptContext(schemes=["argon2"], deprecated="auto")
hashed = pwd_ctx.hash(password)
valid  = pwd_ctx.verify(password, hashed)
```

### Rate Limiting Auth Endpoints
```ts
// Node.js / Fastify
import rateLimit from '@fastify/rate-limit'
await app.register(rateLimit, {
  max: 10,
  timeWindow: '1 minute',
  keyGenerator: (req) => req.ip,
  errorResponseBuilder: () => ({ error: 'Too many requests' })
})
```

```python
# FastAPI / slowapi
from slowapi import Limiter
from slowapi.util import get_remote_address
limiter = Limiter(key_func=get_remote_address)

@app.post("/auth/login")
@limiter.limit("10/minute")
async def login(request: Request, body: LoginRequest): ...
```

---

## API Security

### CORS
```ts
// Only allow your own origins — never wildcard in production
await app.register(cors, {
  origin: [process.env.WEB_ORIGIN!],
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true
})
```

### API Key Validation
```ts
// Hash the key at rest; compare with constant-time comparison
import { timingSafeEqual } from 'crypto'

async function validateApiKey(rawKey: string): Promise<Tenant | null> {
  const keyHash = sha256(rawKey)
  const record  = await db.apiKeys.findOne({ keyHash })
  if (!record) return null
  await db.apiKeys.update({ id: record.id }, { lastUsed: new Date() })
  return record.tenant
}
```

---

## Database Security (PostgreSQL)

### Row Level Security (multi-tenant)
```sql
-- Enable RLS on tenant-shared tables
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON documents
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Set in every request context
SET LOCAL app.tenant_id = '<tenant-uuid>';
```

### Least Privilege DB User
```sql
-- App user: no DDL, no superuser
CREATE ROLE app_user LOGIN PASSWORD 'strong-password';
GRANT CONNECT ON DATABASE myapp TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
```

### Sensitive Column Encryption
```sql
-- PGP symmetric encryption for PII columns
UPDATE users SET email = pgp_sym_encrypt(email, current_setting('app.enc_key'))
WHERE email IS NOT NULL;
```

---

## Stripe / Payments Security

### Always Verify Webhook Signatures
```ts
// Node.js
import Stripe from 'stripe'
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)

app.post('/webhooks/stripe', { config: { rawBody: true } }, async (req, reply) => {
  const sig = req.headers['stripe-signature'] as string
  let event: Stripe.Event

  try {
    event = stripe.webhooks.constructEvent(
      req.rawBody!,
      sig,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch (err) {
    return reply.status(400).send(`Webhook Error: ${err}`)
  }

  // Handle event...
})
```

```python
# FastAPI
import stripe

@app.post("/webhooks/stripe")
async def stripe_webhook(request: Request):
    payload = await request.body()
    sig = request.headers.get("stripe-signature")
    try:
        event = stripe.Webhook.construct_event(
            payload, sig, os.environ["STRIPE_WEBHOOK_SECRET"]
        )
    except stripe.error.SignatureVerificationError:
        raise HTTPException(status_code=400, detail="Invalid signature")
```

### Never Trust Client-Side Amount
- Always retrieve price/amount from Stripe or your DB, never from the request body
- Validate the PaymentIntent amount server-side before fulfilling

---

## AI / ML Security

### Model Input Sanitisation
- Validate and clip numeric inputs (image dimensions, tensor shapes) before inference
- Reject oversized files before loading them into memory (max_size check)
- For LLMs: sanitise prompt inputs to prevent prompt injection

### Prompt Injection (Alpaca / LLaMA)
```python
# Never directly interpolate user input into system prompts
# ❌ CRITICAL
prompt = f"You are a helpful assistant. User says: {user_input}"

# ✅ Use a structured delimiter
prompt = f"""<|system|>You are a helpful assistant.</s>
<|user|>{html.escape(user_input)}</s>
<|assistant|>"""
```

### Model File Security
- Verify model file checksums before loading (SHA256)
- Never load `.pkl` or `.pt` files from untrusted sources (arbitrary code execution)
- Use ONNX or safetensors format when possible

### Inference Rate Limiting
- Rate-limit per tenant/API key — inference is expensive and abusable
- Set max token / max inference time limits per request

### Output Filtering
- Filter model outputs for PII before returning to client
- Log inputs/outputs for abuse detection (with appropriate data retention policies)
