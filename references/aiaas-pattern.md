# AIaaS Pattern Reference

## Core Architecture

```
[Client] → [API Gateway] → [Inference Service] → [Model]
                ↓                   ↓
          [Rate Limit]        [Queue / Batch]
                ↓                   ↓
           [Postgres]         [Result Store]
```

## Model Serving Patterns

### HuggingFace (Transformers)

**Local inference (Python / FastAPI):**
```python
from transformers import pipeline
from fastapi import FastAPI

app = FastAPI()
model = pipeline("text-classification", model="distilbert-base-uncased-finetuned-sst-2-english")

@app.post("/predict")
async def predict(text: str):
    return model(text)[0]
```

**Hosted inference (HuggingFace API):**
```python
import httpx

async def call_hf(model_id: str, inputs: str, hf_token: str):
    async with httpx.AsyncClient() as client:
        r = await client.post(
            f"https://api-inference.huggingface.co/models/{model_id}",
            headers={"Authorization": f"Bearer {hf_token}"},
            json={"inputs": inputs}
        )
        return r.json()
```

### YOLOv8 (Object Detection)

**Inference service wrapper:**
```python
from ultralytics import YOLO
from fastapi import FastAPI, UploadFile
import io, cv2, numpy as np

app = FastAPI()
model = YOLO("yolov8n.pt")  # or custom weights

@app.post("/detect")
async def detect(file: UploadFile):
    contents = await file.read()
    nparr = np.frombuffer(contents, np.uint8)
    img = cv2.imdecode(nparr, cv2.IMREAD_COLOR)
    results = model(img)
    return {"detections": results[0].tojson()}
```

### Alpaca / LLaMA (llama.cpp)

**llama.cpp server mode:**
```bash
./server -m models/alpaca-7b.gguf --host 0.0.0.0 --port 8080 -c 4096
```

**Python client wrapper:**
```python
import httpx

async def alpaca_complete(prompt: str, max_tokens: int = 256):
    async with httpx.AsyncClient() as client:
        r = await client.post("http://localhost:8080/completion", json={
            "prompt": prompt,
            "n_predict": max_tokens,
            "stop": ["\n###"]
        })
        return r.json()["content"]
```

## Async Inference Queue

For slow models (>2s inference), use a job queue:

```sql
CREATE TABLE inference_jobs (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id   UUID REFERENCES tenants(id),
  model       TEXT NOT NULL,
  input       JSONB NOT NULL,
  status      TEXT DEFAULT 'queued' CHECK (status IN ('queued','running','done','failed')),
  result      JSONB,
  error       TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ
);
```

**Queue worker (Node.js + BullMQ):**
```ts
import { Worker } from 'bullmq'

new Worker('inference', async (job) => {
  const result = await runModel(job.data)
  await db.inferenceJobs.update({ where: { id: job.data.jobId }, data: { status: 'done', result } })
}, { connection: redis })
```

## Standard Folder Layout (AIaaS)

```
/
├── inference/
│   ├── models/       # Model loader + wrapper per model type
│   ├── queue/        # BullMQ or pg-queue workers
│   └── cache/        # Result caching layer
├── api/
│   ├── routes/       # FastAPI or Fastify routers
│   └── openapi.yaml
├── web/              # React/TS dashboard (usage stats, API keys)
├── db/
│   └── migrations/
├── docker-compose.yml
└── .github/workflows/ci.yml
```

## Docker Compose (AIaaS GPU-optional)

```yaml
services:
  api:
    build: ./api
    ports: ["8000:8000"]
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/aiaas
    depends_on: [db]

  inference:
    build: ./inference
    # Uncomment for GPU:
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - capabilities: [gpu]
    volumes:
      - ./models:/app/models

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: aiaas
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

## Observability

Always instrument inference endpoints:
```python
# Track: latency, token count, model version, tenant
import time

@app.middleware("http")
async def track_inference(request, call_next):
    start = time.time()
    response = await call_next(request)
    latency = time.time() - start
    await db.log_inference(latency=latency, path=request.url.path)
    return response
```

## Pricing Model (Stripe metered billing)

```ts
// Report usage after each inference
await stripe.billing.meterEvents.create({
  event_name: 'inference_tokens',
  payload: {
    stripe_customer_id: tenant.stripeId,
    value: String(tokenCount),
  },
})
```
