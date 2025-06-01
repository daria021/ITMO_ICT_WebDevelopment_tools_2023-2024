# 🌐 **api\_service** – основной API‑шлюз

> Переписано: один Celery‑воркер, взаимодействие с парсером **только через Redis**.
> Каталог без подпапок – ровно столько файлов, сколько нужно: `main.py`, `routes.py`, `requests.py`, `responses.py`, `tasks.py`, `models.py`, `database.py`, `celery_app.py`.

## 📁 Структура

```
api_service/
├── Dockerfile
├── requirements.txt
├── main.py
├── routes.py
├── requests.py
├── responses.py
├── tasks.py
├── celery_app.py
├── models.py
├── database.py
├── status_enum.py      # 🆕 enum вынесен отдельно
└── repository.py       # 🆕 инкапсулированные операции БД
```

api\_service/
├── Dockerfile
├── requirements.txt
├── main.py
├── routes.py
├── requests.py
├── responses.py
├── tasks.py
├── celery\_app.py
├── models.py
└── database.py

````

---

## 🔧 Dockerfile
```dockerfile
FROM python:3.12-slim AS runtime

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONUNBUFFERED=1 \
    DATABASE_URL=postgresql+asyncpg://user:pass@db:5432/app_db \
    REDIS_URL=redis://redis:6379/0 \
    TASK_QUEUE=parser:tasks \
    RESULT_PREFIX=parser:results:

# 📝 Не задаём CMD здесь. Каждому сервису в docker-compose
# будет передана своя команда: Uvicorn для "api" и Celery для "api_worker".
````

---

## 📦 requirements.txt

```text
fastapi==0.110.0
uvicorn[standard]==0.29.0
httpx[http2]==0.27.0
redis==5.0.5                    # client для очереди
celery[redis]==5.4.0
sqlalchemy[asyncio]==2.0.30
asyncpg==0.29.0
alembic==1.13.1
pydantic==2.7.1
python-dotenv==1.0.1
```

---

### 🐍 **celery\_app.py** – конфигурация Celery

```python
from __future__ import annotations

import os
from celery import Celery

REDIS_URL = os.getenv("REDIS_URL", "redis://redis:6379/0")

celery_app = Celery(
    "api_service",
    broker=REDIS_URL,
    backend=REDIS_URL,
    include=["tasks"],
)

celery_app.conf.task_default_queue = "api"
celery_app.conf.timezone = "UTC"
```

---

### 🐍 **status\_enum.py** – перечисление статусов

```python
from enum import StrEnum


class StatusEnum(StrEnum):
    pending = "pending"
    processing = "processing"
    success = "success"
    failure = "failure"
```

---

### 🐍 **models.py** – SQLAlchemy ORM (декларативный стиль)

```python
from __future__ import annotations

from datetime import datetime
from typing import Optional

from sqlalchemy import Enum, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

from status_enum import StatusEnum


class Base(DeclarativeBase):
    pass


class ParseRequest(Base):
    """Таблица запросов на парсинг."""

    __tablename__ = "parse_requests"

    id: Mapped[int] = mapped_column(primary_key=True)
    url: Mapped[str]
    status: Mapped[StatusEnum] = mapped_column(Enum(StatusEnum), default=StatusEnum.pending)
    title: Mapped[Optional[str]]
    html_content: Mapped[Optional[str]]

    created_at: Mapped[datetime] = mapped_column(default=datetime.utcnow, server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column(
        default=datetime.utcnow,
        onupdate=datetime.utcnow,
        server_default=func.now(),
    )
```

---

### 🐍 **database.py** – async engine + helpers

```python
from __future__ import annotations

import os
from contextlib import asynccontextmanager

from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

DATABASE_URL = os.getenv("DATABASE_URL", "postgresql+asyncpg://user:pass@db:5432/app_db")

engine = create_async_engine(DATABASE_URL, echo=False)
AsyncSessionLocal = sessionmaker(engine, expire_on_commit=False, class_=AsyncSession)


@asynccontextmanager
async def get_session():  # Dependency для FastAPI
    async with AsyncSessionLocal() as session:
        yield session
```

---

### 🐍 **requests.py** – входные схемы

```python
from __future__ import annotations

from pydantic import BaseModel, HttpUrl, Field


class ParseAsyncCreate(BaseModel):
    url: HttpUrl = Field(..., example="https://example.com")


class ParseSyncQuery(BaseModel):
    url: HttpUrl = Field(..., example="https://example.com")
```

---

### 🐍 **responses.py** – выходные схемы

```python
from __future__ import annotations

from typing import Optional

from pydantic import BaseModel, HttpUrl

from status_enum import StatusEnum


class ParseAsyncResponse(BaseModel):
    request_id: int
    status: StatusEnum


class StatusResponse(BaseModel):
    request_id: int
    status: StatusEnum


class ResultResponse(BaseModel):
    request_id: int
    status: StatusEnum
    title: Optional[str] = None
    html: Optional[str] = None


class SyncResult(BaseModel):
    title: str
    html: str
    url: HttpUrl
```

---

### 🐍 **tasks.py** – Celery‑задача парсинга

```python
from __future__ import annotations

import asyncio
import json
import os
from time import time

import redis
from celery import shared_task
from sqlalchemy import select, update

from celery_app import celery_app
from database import AsyncSessionLocal, engine
from models import ParseRequest, StatusEnum

REDIS_URL = os.getenv("REDIS_URL", "redis://redis:6379/0")
TASK_QUEUE = os.getenv("TASK_QUEUE", "parser:tasks")
RESULT_PREFIX = os.getenv("RESULT_PREFIX", "parser:results:")


@shared_task(name="api.parse_url", queue="api")
def parse_url_task(request_id: int, url: str) -> None:
    """Кладёт задание в Redis, ждёт результат, затем пишет его в БД."""
    r = redis.Redis.from_url(REDIS_URL, decode_responses=True)

    # 1) Отправляем задачу парсеру
    payload = json.dumps({"id": request_id, "url": url})
    r.lpush(TASK_QUEUE, payload)

    # 2) Блокирующе ждём результат (макс 30 c)
    res_key = f"{RESULT_PREFIX}{request_id}"
    deadline = time() + 30
    result_data: dict[str, str] | None = None
    while time() < deadline:
        data = r.get(res_key)
        if data:
            result_data = json.loads(data)
            r.delete(res_key)
            break
        r.sleep(1)  # Redis >= 5 client helper

    async def _write(status: StatusEnum, title: str | None = None, html: str | None = None):
        async with AsyncSessionLocal() as session:
            stmt = (
                update(ParseRequest)
                .where(ParseRequest.id == request_id)
                .values(status=status, title=title, html_content=html)
            )
            await session.execute(stmt)
            await session.commit()

    if not result_data:
        asyncio.run(_write(StatusEnum.failure, title=None, html="Timeout waiting result"))
        return

    if result_data.get("status") == "success":
        asyncio.run(_write(StatusEnum.success, result_data.get("title"), result_data.get("html")))
    else:
        asyncio.run(_write(StatusEnum.failure, html=result_data.get("error")))
```

---

### 🐍 **repository.py** – CRUD‑операции

```python
from __future__ import annotations

from sqlalchemy import select, update
from sqlalchemy.ext.asyncio import AsyncSession

from models import ParseRequest
from status_enum import StatusEnum


class ParseRepository:
    """Инкапсулирует доступ к таблице parse_requests."""

    @staticmethod
    async def create(session: AsyncSession, url: str) -> ParseRequest:
        req = ParseRequest(url=url, status=StatusEnum.pending)
        session.add(req)
        await session.commit()
        await session.refresh(req)
        return req

    @staticmethod
    async def by_id(session: AsyncSession, request_id: int) -> ParseRequest | None:
        res = await session.execute(select(ParseRequest).where(ParseRequest.id == request_id))
        return res.scalar_one_or_none()

    @staticmethod
    async def update_status(
        session: AsyncSession,
        request_id: int,
        status: StatusEnum,
        *,
        title: str | None = None,
        html: str | None = None,
    ) -> None:
        stmt = (
            update(ParseRequest)
            .where(ParseRequest.id == request_id)
            .values(status=status, title=title, html_content=html)
        )
        await session.execute(stmt)
        await session.commit()
```

---

### 🐍 **routes.py** – HTTP‑эндпойнты (без прямого SQLAlchemy)

```python
from __future__ import annotations

import httpx
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.ext.asyncio import AsyncSession

from database import get_session
from repository import ParseRepository as Repo
from requests import ParseAsyncCreate, ParseSyncQuery
from responses import (
    ParseAsyncResponse,
    StatusResponse,
    ResultResponse,
    SyncResult,
)
from status_enum import StatusEnum
from tasks import parse_url_task

PARSER_HTTP = "http://parser:8001/fetch"

router = APIRouter()


@router.post("/parse_async", response_model=ParseAsyncResponse, status_code=status.HTTP_202_ACCEPTED)
async def parse_async(payload: ParseAsyncCreate, session: AsyncSession = Depends(get_session)):
    req = await Repo.create(session, str(payload.url))
    parse_url_task.delay(req.id, str(payload.url))
    return ParseAsyncResponse(request_id=req.id, status=req.status)


@router.get("/status/{request_id}", response_model=StatusResponse)
async def status(request_id: int, session: AsyncSession = Depends(get_session)):
    obj = await Repo.by_id(session, request_id)
    if not obj:
        raise HTTPException(status_code=404, detail="Not found")
    return StatusResponse(request_id=obj.id, status=obj.status)


@router.get("/result/{request_id}", response_model=ResultResponse)
async def result(request_id: int, session: AsyncSession = Depends(get_session)):
    obj = await Repo.by_id(session, request_id)
    if not obj:
        raise HTTPException(status_code=404, detail="Not found")
    return ResultResponse(
        request_id=obj.id,
        status=obj.status,
        title=obj.title,
        html=obj.html_content,
    )


@router.get("/parse_sync", response_model=SyncResult)
async def parse_sync(query: ParseSyncQuery):
    async with httpx.AsyncClient(timeout=10, follow_redirects=True) as client:
        try:
            resp = await client.get(PARSER_HTTP, params={"url": query.url})
            resp.raise_for_status()
        except httpx.HTTPError as exc:
            raise HTTPException(status_code=502, detail=str(exc)) from exc
    data = resp.json()
    return SyncResult(url=query.url, title=data["title"], html=data["html"])
```

---

### 🐍 **main.py** – FastAPI + lifespan

```python
from __future__ import annotations

from contextlib import asynccontextmanager

from fastapi import FastAPI

from routes import router


@asynccontextmanager
async def lifespan(app: FastAPI):  # noqa: D401
    # Здесь можно подключить мониторинг / логи, но пока просто yield
    yield


app = FastAPI(title="API Service", version="1.0.0", lifespan=lifespan)
app.include_router(router)
```

---

## 🔄 Alembic (как и раньше)

* `alembic init alembic`, в `env.py`: `target_metadata = models.Base.metadata`
* `alembic revision --autogenerate -m "create parse_requests"`

---

## 🐳 docker-compose — итоговый файл

```yaml
version: "3.9"

services:
  # --- DB & Redis -------------------------------------------------
  db:
    image: postgres:15-alpine
    container_name: db
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app_db
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "user"]
      interval: 5s
      retries: 5
    volumes:
      - dbdata:/var/lib/postgresql/data
    networks: [backend]

  redis:
    image: redis:7-alpine
    container_name: redis
    healthcheck:
      test: ["CMD", "redis-cli", "PING"]
      interval: 5s
      retries: 5
    networks: [backend]

  # --- Parser microservice ---------------------------------------
  parser:
    build:
      context: ./parser_service
    container_name: parser_service
    environment:
      REDIS_URL: redis://redis:6379/0
      TASK_QUEUE: parser:tasks
      RESULT_PREFIX: parser:results:
    depends_on:
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8001/health" ]  # опционально, если добавишь эндпойнт
      interval: 10s
      retries: 3
    networks: [backend]

  # --- API (HTTP) -------------------------------------------------
  api:
    build:
      context: ./api_service
    container_name: api_service
    command: uvicorn main:app --host 0.0.0.0 --port 8000
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@db:5432/app_db
      REDIS_URL: redis://redis:6379/0
      TASK_QUEUE: parser:tasks
      RESULT_PREFIX: parser:results:
    depends_on:
      db:
        condition: service_healthy
      migrate:
        condition: service_completed_successfully
      redis:
        condition: service_healthy
      parser:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/docs" ]
      interval: 10s
      retries: 3
    networks: [backend]

  # --- Celery worker ---------------------------------------------
  api_worker:
    build:
      context: ./api_service
    container_name: api_worker
    command: celery -A celery_app worker --loglevel=info --concurrency=4 -Q api
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@db:5432/app_db
      REDIS_URL: redis://redis:6379/0
      TASK_QUEUE: parser:tasks
      RESULT_PREFIX: parser:results:
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
      parser:
        condition: service_healthy
    networks: [backend]

  # --- Alembic migrations (one‑shot) ------------------------------
  migrate:
    build:
      context: ./api_service
    container_name: migrate
    command: alembic upgrade head
    environment:
      DATABASE_URL: postgresql+asyncpg://user:pass@db:5432/app_db
    depends_on:
      db:
        condition: service_healthy
    restart: "no"
    networks: [backend]

# ---------------------------------------------------------------
volumes:
  dbdata:

networks:
  backend:
    driver: bridge
```
