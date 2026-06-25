# Инфраструктура

## Локальная разработка

### Docker Compose

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    volumes:
      - ./src:/app/src
    environment:
      - DATABASE_URL=postgresql+asyncpg://markethacker:markethacker@postgres:5432/markethacker
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: markethacker
      POSTGRES_PASSWORD: markethacker
      POSTGRES_DB: markethacker
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U markethacker"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  worker:
    build: .
    command: arq markethacker.infrastructure.jobs.WorkerSettings
    environment:
      - DATABASE_URL=postgresql+asyncpg://markethacker:markethacker@postgres:5432/markethacker
      - REDIS_URL=redis://redis:6379/0
    depends_on:
      - postgres
      - redis

volumes:
  pgdata:
```

### Быстрый старт

```bash
cd backend
cp .env.example .env
docker compose up -d
alembic upgrade head
uvicorn markethacker.main:app --reload
```

## CI/CD

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: ["5432:5432"]
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]

    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v4
      - run: uv sync
      - run: ruff check .
      - run: ruff format --check .
      - run: mypy src/
      - run: pytest --cov=markethacker
      - run: pip-audit
```

### Pipeline

```mermaid
flowchart LR
    PUSH[git push] --> LINT[Lint + Type Check]
    LINT --> TEST[Tests]
    TEST --> AUDIT[Dependency Audit]
    AUDIT --> BUILD[Docker Build]
    BUILD --> MIGRATE[Alembic Migrate]
    MIGRATE --> DEPLOY[Deploy]
```

## Production

### Топология (текущая)

Все компоненты развёрнуты на одном VPS. Caddy работает в host-network, проксируя по доменному имени.

```
Internet :443
    │
    ▼
 Caddy (host network, TLS Let's Encrypt)
    ├── api.markethacker.ru       → 127.0.0.1:8000  (FastAPI backend)
    ├── admin.markethacker.ru     → 127.0.0.1:3001  (Admin Panel, Next.js)
    ├── team.markethacker.ru      → 127.0.0.1:3002  (Manager Portal, Next.js)
    └── wb-proxy.markethacker.ru  → 127.0.0.1:8000  (WB Portal Proxy, FastAPI)

Docker-сети (межсервисное):
    backend api/worker ──markethacker_apps──► parser-api:8010, clickhouse:8123
    backend api/worker ──markethacker_backend_infra──► postgres, pgbouncer, redis
    parser api/worker  ──markethacker_parser_infra──► postgres, pgbouncer, redis, clickhouse
```

> `wb-proxy.markethacker.ru` указывает на тот же FastAPI backend, Caddy переписывает путь:
> `GET /something` → `GET /api/v1/proxy/portal/something`.

### Docker Compose стеки

Каждый сервис разворачивается отдельным стеком из своего каталога:

| Стек | Каталог | Compose-файл | Порт (127.0.0.1) |
|------|---------|--------------|------------------|
| Сети (bootstrap) | `infra/` | `make networks` | — |
| Backend infra | `backend/` | `docker-compose.yml` | — |
| Backend app | `backend/` | `docker-compose.prod.yml` | 8000 |
| Parser infra | `parser/` | `docker-compose.yml` | — |
| Parser app | `parser/` | `docker-compose.prod.yml` | 8010 |
| Admin Panel | `admin-panel/` | `docker-compose.prod.yml` | 3001 |
| Manager Portal | `manager-portal/` | `docker-compose.prod.yml` | 3002 |
| Caddy | `caddy/` | `docker-compose.yml` | 80/443 |

### Порядок деплоя на VPS

```bash
cd infra && make networks
cd backend && make infra-up-prod && make prod-migrate && make prod-up
cd parser && make infra-up-prod && make prod-migrate && make prod-up
cd admin-panel && docker compose -f docker-compose.prod.yml up -d
cd manager-portal && docker compose -f docker-compose.prod.yml up -d
cd caddy && make up
```

### Компоненты

| Компонент | Решение | Примечание |
|-----------|---------|------------|
| API (FastAPI) | Docker container | + WB Portal Proxy |
| Worker (ARQ) | Docker container | Фоновые задачи |
| Admin Panel | Docker container (Next.js) | `admin.markethacker.ru` |
| Manager Portal | Docker container (Next.js) | `team.markethacker.ru` |
| Caddy | Docker (host network) | TLS, reverse proxy |
| PostgreSQL | Managed или Docker | Автобэкапы |
| Redis | Managed или Docker | Persistence AOF |

### Health checks

| Сервис | Probe | Путь |
|--------|-------|------|
| API | Liveness | `GET /health` |
| API | Readiness | `GET /ready` |
| Admin Panel | Docker | `GET /login` (порт 3001) |
| Manager Portal | Docker | `GET /login` (порт 3002) |

### Логирование

- Structured JSON logs (structlog).
- `request_id` в каждой записи.
- Уровни: DEBUG (dev), INFO (prod), ERROR (всегда).

### Мониторинг

| Инструмент | Назначение |
|------------|------------|
| Prometheus + Grafana | Метрики API, workers |
| Sentry | Error tracking |
| Uptime monitoring | Внешняя проверка `/health` |

## Масштабирование (будущее)

| Этап | Действие |
|------|----------|
| API bottleneck | Horizontal scaling API replicas |
| DB bottleneck | Read replicas, connection pooling (PgBouncer) |
| Worker bottleneck | Больше worker replicas, per-account queues |
| Межсервисное | Выделение analytics / sync в отдельные сервисы |

## Переменные окружения

### Backend (`backend/.env`)

```bash
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/markethacker
REDIS_URL=redis://localhost:6379/0
JWT_SECRET=change-me-in-production
JWT_ACCESS_TTL_MINUTES=15
JWT_REFRESH_TTL_DAYS=30
ENCRYPTION_KEY=change-me-32-bytes-key-here!!!!
ENVIRONMENT=production
LOG_LEVEL=INFO
CORS_ORIGINS=["https://team.markethacker.ru","https://wb-proxy.markethacker.ru","https://admin.markethacker.ru"]

# WB Portal Proxy (production)
WB_PORTAL_PUBLIC_BASE_URL=https://wb-proxy.markethacker.ru
WB_PORTAL_COOKIE_PATH=/
WB_PORTAL_COOKIE_SECURE=true
```

> ⚠️ `WB_PORTAL_COOKIE_PATH=/` обязателен в production. Caddy rewrite убирает `/api/v1/proxy/portal`, поэтому cookie должна устанавливаться на корневой путь.

### Manager Portal (`manager-portal/.env`)

```bash
NEXT_PUBLIC_API_URL=https://api.markethacker.ru/api/v1
MANAGER_PORTAL_PORT=3002
FULL_IMAGE=markethacker-manager-portal:latest
```
