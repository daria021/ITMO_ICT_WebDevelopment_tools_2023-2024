🕷️ parser_service – модульный микросервис‑парсер

Новая структура: шесть маленьких файлов в корне.  Никаких подпакетов и «core» – только то, что попросил.

• main.py – точка входа, создание FastAPI, lifecycle, запуск redis_listener.
• routes.py – один APIRouter с ручкой /fetch.
• requests.py – Pydantic‑модели входных запросов (здесь только FetchQuery).
• responses.py – Pydantic‑ответ FetchResponse.
• parser.py – функция fetch_html_and_title(url).
• redis_listener.py – асинхронный consumer очереди parser:tasks.

📁 Структура каталога
```
parser_service/
├── Dockerfile
├── requirements.txt
├── main.py
├── routes.py
├── requests.py
├── responses.py
├── parser.py
└── redis_listener.py
```
🔧 Dockerfile
```Dockerfile

FROM python:3.12-slim AS runtime

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential libxml2-dev libxslt1-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src/app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV PYTHONUNBUFFERED=1 \
    REDIS_URL=redis://redis:6379/0 \
    TASK_QUEUE=parser:tasks \
    RESULT_PREFIX=parser:results:

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001"]
```

📦 requirements.txt
```
fastapi==0.110.0
uvicorn[standard]==0.29.0
httpx[http2]==0.27.0
redis==5.0.5
beautifulsoup4==4.12.3
lxml==5.2.1
python-dotenv==1.0.1
```

🐍 main.py – точка входа (lifespan‑подход)
```python
from __future__ import annotations

import asyncio
import os
from contextlib import asynccontextmanager

import redis.asyncio as aioredis
from fastapi import FastAPI

from redis_listener import start_listener
from routes import router

REDIS_URL = os.getenv("REDIS_URL", "redis://localhost:6379/0")

@asynccontextmanager
async def lifespan(app: FastAPI):  # noqa: D401
    """Создаём Redis‑клиент и запускаем listener во время жизни приложения."""
    redis = aioredis.from_url(REDIS_URL, decode_responses=True)
    # кладём клиент в state — может пригодиться в ручках
    app.state.redis = redis
    listener_task = asyncio.create_task(start_listener(redis))
    try:
        yield  # приложение запущено
    finally:
        listener_task.cancel()
        try:
            await listener_task
        except asyncio.CancelledError:
            pass
        await redis.close()


app = FastAPI(title="Parser Service", version="1.0.0", lifespan=lifespan)
app.include_router(router)
```
🐍 routes.py – HTTP‑ручки
```python
from __future__ import annotations

from fastapi import APIRouter, HTTPException, Query

from parser import fetch_html_and_title
from requests import FetchQuery
from responses import FetchResponse

router = APIRouter()


@router.get("/fetch", response_model=FetchResponse, summary="Скачать страницу и вернуть HTML")
async def fetch(query: FetchQuery = Query(...)):
    try:
        title, html = await fetch_html_and_title(str(query.url))
    except Exception as exc:  # noqa: BLE001
        raise HTTPException(status_code=502, detail=str(exc)) from exc
    return FetchResponse(url=query.url, title=title, html=html)
```
🐍 requests.py – входные модели
```python
from __future__ import annotations

from pydantic import BaseModel, HttpUrl, Field


class FetchQuery(BaseModel):
    """Модель query‑параметра для /fetch."""

    url: HttpUrl = Field(..., example="https://example.com")
```
🐍 responses.py – модели ответа
```python
from __future__ import annotations

from pydantic import BaseModel, HttpUrl


class FetchResponse(BaseModel):
    url: HttpUrl
    title: str
    html: str
```
🐍 parser.py – логика парсинга
```python
from __future__ import annotations

from typing import Tuple

import httpx
from bs4 import BeautifulSoup


async def fetch_html_and_title(url: str, timeout: float = 10.0) -> Tuple[str, str]:
    """Скачивает страницу и извлекает <title>."""
    async with httpx.AsyncClient(follow_redirects=True, timeout=timeout) as client:
        resp = await client.get(url)
        resp.raise_for_status()
        html_text: str = resp.text
    soup = BeautifulSoup(html_text, "lxml")
    title_tag = soup.find("title")
    title = title_tag.text.strip() if title_tag else ""
    return title, html_text
```
🐍 redis_listener.py – consumer очереди
```python
from __future__ import annotations

import asyncio
import json
import os
from typing import Any, Dict

import redis.asyncio as aioredis

from parser import fetch_html_and_title

TASK_QUEUE = os.getenv("TASK_QUEUE", "parser:tasks")
RESULT_PREFIX = os.getenv("RESULT_PREFIX", "parser:results:")


async def start_listener(redis: aioredis.Redis) -> None:
    """Запускает бесконечный BLPOP‑цикл."""
    print("[listener] started, waiting for tasks…")
    while True:
        try:
            task = await redis.blpop(TASK_QUEUE, timeout=5)
            if task is None:
                continue  # нет заданий
            _, raw = task
            payload: Dict[str, Any] = json.loads(raw)
            req_id = payload["id"]
            url = payload["url"]
            print(f"[listener] got task id={req_id} url={url}")

            try:
                title, html = await fetch_html_and_title(url)
                result = {"status": "success", "title": title, "html": html}
            except Exception as err:  # noqa: BLE001
                print("[listener] task failed", err)
                result = {"status": "failure", "error": str(err)}

            await redis.set(f"{RESULT_PREFIX}{req_id}", json.dumps(result), ex=3600)
            print(f"[listener] stored result for id={req_id}")
        except Exception as exc:  # noqa: BLE001
            print("[listener] error:", exc)
            await asyncio.sleep(1)
```
🐳 docker-compose (фрагмент остаётся прежним)
```yaml
services:
  parser:
    build: ./parser_service
    container_name: parser_service
    environment:
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      redis:
        condition: service_healthy
    networks: [backend]

  redis:
    image: redis:7-alpine
    container_name: redis
    healthcheck:
      test: ["CMD", "redis-cli", "PING"]
      interval: 5s
      retries: 5
    networks: [backend]
```
Готово.  Всё лежит в одном каталоге – ровно те файлы, что тебе нужны.

