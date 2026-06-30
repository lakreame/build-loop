# Python Pattern Reference

FastAPI-first, async-by-default Python backend patterns for SaaS, DaaS, and AIaaS projects.

---

## Project Structure

```
/
├── app/
│   ├── main.py              # FastAPI app entrypoint
│   ├── core/
│   │   ├── config.py        # Pydantic Settings
│   │   ├── security.py      # JWT, password hashing
│   │   └── deps.py          # Shared FastAPI dependencies
│   ├── api/
│   │   └── v1/
│   │       ├── routes/      # One router per resource
│   │       └── api.py       # Router aggregation
│   ├── models/               # SQLAlchemy ORM models
│   ├── schemas/               # Pydantic request/response models
│   ├── services/              # Business logic layer
│   ├── db/
│   │   ├── session.py        # Async engine + session factory
│   │   └── migrations/       # Alembic migrations
│   └── workers/                # Background task / queue consumers
├── tests/
│   ├── conftest.py
│   ├── unit/
│   └── integration/
├── pyproject.toml
├── alembic.ini
├── Dockerfile
└── .github/workflows/ci.yml
```

---

## Dependency & Project Management — uv (recommended) or Poetry

### pyproject.toml (uv)
```toml
[project]
name = "myapp-api"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "fastapi[standard]>=0.115.0",
    "pydantic>=2.9.0",
    "pydantic-settings>=2.5.0",
    "sqlalchemy[asyncio]>=2.0.35",
    "asyncpg>=0.29.0",
    "alembic>=1.13.0",
    "passlib[argon2]>=1.7.4",
    "pyjwt>=2.9.0",
    "httpx>=0.27.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=5.0.0",
    "ruff>=0.6.0",
    "mypy>=1.11.0",
]

[tool.ruff]
line-length = 100
target-version = "py312"

[tool.mypy]
strict = true
plugins = ["pydantic.mypy"]
```

```bash
uv venv
uv sync --all-extras
uv run uvicorn app.main:app --reload
```

---

## FastAPI — Core Conventions

### App Entrypoint
```python
# app/main.py
from contextlib import asynccontextmanager
from fastapi import FastAPI
from app.api.v1.api import api_router
from app.core.config import settings
from app.db.session import engine

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    yield
    # Shutdown
    await engine.dispose()

app = FastAPI(
    title=settings.PROJECT_NAME,
    version="1.0.0",
    lifespan=lifespan,
    docs_url="/docs" if settings.ENV != "production" else None,
)

app.include_router(api_router, prefix="/api/v1")
```

### Settings (Pydantic Settings)
```python
# app/core/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    PROJECT_NAME: str = "MyApp API"
    ENV: str = "development"
    DATABASE_URL: str
    JWT_SECRET: str
    JWT_EXPIRE_MINUTES: int = 15
    STRIPE_SECRET_KEY: str | None = None
    STRIPE_WEBHOOK_SECRET: str | None = None

settings = Settings()
```

### Router Pattern
```python
# app/api/v1/routes/users.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.deps import get_db, get_current_user
from app.schemas.user import UserResponse, UserCreate
from app.services.user_service import UserService

router = APIRouter(prefix="/users", tags=["users"])

@router.get("/me", response_model=UserResponse)
async def read_current_user(current_user=Depends(get_current_user)):
    return current_user

@router.post("/", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(payload: UserCreate, db: AsyncSession = Depends(get_db)):
    service = UserService(db)
    existing = await service.get_by_email(payload.email)
    if existing:
        raise HTTPException(status_code=409, detail="Email already registered")
    return await service.create(payload)
```

### Pydantic Schemas
```python
# app/schemas/user.py
from pydantic import BaseModel, EmailStr, ConfigDict
from uuid import UUID
from datetime import datetime

class UserCreate(BaseModel):
    email: EmailStr
    password: str

class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: UUID
    email: EmailStr
    created_at: datetime
```

### Dependency Injection (DB session + auth)
```python
# app/core/deps.py
from typing import AsyncGenerator
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.ext.asyncio import AsyncSession

from app.db.session import async_session
from app.core.security import decode_token

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        yield session

async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db),
):
    payload = decode_token(token)
    if payload is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    user = await db.get(User, payload["sub"])
    if user is None:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
    return user
```

---

## Database — SQLAlchemy 2.0 (Async)

### Engine & Session
```python
# app/db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker
from app.core.config import settings

engine = create_async_engine(settings.DATABASE_URL, pool_size=10, max_overflow=20)
async_session = async_sessionmaker(engine, expire_on_commit=False)
```

### Models
```python
# app/models/user.py
import uuid
from datetime import datetime
from sqlalchemy import String, DateTime, func
from sqlalchemy.orm import Mapped, mapped_column
from app.db.base import Base

class User(Base):
    __tablename__ = "users"

    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    email: Mapped[str] = mapped_column(String, unique=True, index=True)
    password_hash: Mapped[str] = mapped_column(String)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

### Alembic Migrations
```bash
alembic init app/db/migrations
alembic revision --autogenerate -m "create users table"
alembic upgrade head
```

---

## Auth — Argon2 + JWT

```python
# app/core/security.py
import jwt
from datetime import datetime, timedelta, timezone
from passlib.context import CryptContext
from app.core.config import settings

pwd_context = CryptContext(schemes=["argon2"], deprecated="auto")

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(subject: str) -> str:
    expire = datetime.now(timezone.utc) + timedelta(minutes=settings.JWT_EXPIRE_MINUTES)
    return jwt.encode({"sub": subject, "exp": expire}, settings.JWT_SECRET, algorithm="HS256")

def decode_token(token: str) -> dict | None:
    try:
        return jwt.decode(token, settings.JWT_SECRET, algorithms=["HS256"])
    except jwt.InvalidTokenError:
        return None
```

---

## Background Tasks & Queues

### Simple FastAPI BackgroundTasks (low-volume)
```python
from fastapi import BackgroundTasks

@router.post("/send-report")
async def send_report(background_tasks: BackgroundTasks):
    background_tasks.add_task(generate_and_email_report)
    return {"status": "queued"}
```

### Celery (production queue, AIaaS/DaaS workloads)
```python
# app/workers/celery_app.py
from celery import Celery
from app.core.config import settings

celery_app = Celery("worker", broker=settings.REDIS_URL, backend=settings.REDIS_URL)
celery_app.conf.task_routes = {"app.workers.tasks.*": {"queue": "default"}}
```

```python
# app/workers/tasks.py
from app.workers.celery_app import celery_app

@celery_app.task(bind=True, max_retries=3)
def run_inference(self, model_id: str, payload: dict):
    try:
        return run_model(model_id, payload)
    except Exception as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
```

---

## Testing — pytest + httpx

### conftest.py
```python
# tests/conftest.py
import pytest
import pytest_asyncio
from httpx import AsyncClient, ASGITransport
from app.main import app
from app.db.session import async_session

@pytest_asyncio.fixture
async def client():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac

@pytest_asyncio.fixture
async def db_session():
    async with async_session() as session:
        yield session
        await session.rollback()
```

### Example Tests
```python
# tests/integration/test_users.py
import pytest

@pytest.mark.asyncio
async def test_create_user(client):
    response = await client.post("/api/v1/users/", json={
        "email": "test@example.com",
        "password": "supersecret123",
    })
    assert response.status_code == 201
    assert response.json()["email"] == "test@example.com"

@pytest.mark.asyncio
async def test_duplicate_email_rejected(client):
    payload = {"email": "dup@example.com", "password": "pw123456"}
    await client.post("/api/v1/users/", json=payload)
    response = await client.post("/api/v1/users/", json=payload)
    assert response.status_code == 409
```

### pytest config
```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
addopts = "--cov=app --cov-report=term-missing --cov-fail-under=80"
```

---

## Linting & Type Checking — Ruff + mypy

```bash
ruff check app/ tests/ --fix
ruff format app/ tests/
mypy app/
```

---

## Docker

```dockerfile
# Dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen --no-dev

FROM python:3.12-slim AS runtime
WORKDIR /app
RUN useradd -m appuser
COPY --from=builder /app/.venv /app/.venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
USER appuser
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## GitHub Actions CI

```yaml
name: Python CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --all-extras
      - run: uv run ruff check app/ tests/
      - run: uv run mypy app/
      - run: uv run alembic upgrade head
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/test
      - run: uv run pytest
        env:
          DATABASE_URL: postgresql+asyncpg://postgres:postgres@localhost:5432/test
          JWT_SECRET: test-secret-key-for-ci-only
```

---

## AIaaS Tie-In

When `python` is combined with `huggingface` / `yolov8` / `alpaca`, route to
`references/aiaas-pattern.md` for the model-serving code and `references/backend-security.md`
for AI/ML-specific security (model checksum validation, prompt injection prevention, inference
rate limiting). This file (`python-pattern.md`) covers the surrounding API/service layer; the
AIaaS file covers the model layer itself.
