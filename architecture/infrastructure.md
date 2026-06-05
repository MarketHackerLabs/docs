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

## Production (этап 1)

### Компоненты

| Компонент | Решение | Примечание |
|-----------|---------|------------|
| API | Docker container | 1-2 replicas |
| Worker | Docker container | 1 replica |
| PostgreSQL | Managed (Yandex / RDS) | Автобэкапы |
| Redis | Managed Redis | Persistence AOF |
| Secrets | Env vars → Vault | По мере роста |
| DNS | `api.markethacker.ru` | |
| TLS | Let's Encrypt / cloud | |

### Минимальная конфигурация

```
┌─────────────┐     ┌─────────────┐
│   API (x2)  │────│  PostgreSQL  │
└──────┬──────┘     └─────────────┘
       │
┌──────┴──────┐     ┌─────────────┐
│  Worker (x1)│────│    Redis     │
└─────────────┘     └─────────────┘
```

### Health checks

| Probe | Путь | Проверяет |
|-------|------|-----------|
| Liveness | `GET /health` | Процесс жив |
| Readiness | `GET /ready` | DB + Redis доступны |

### Логирование

- Structured JSON logs (structlog).
- `request_id` в каждой записи.
- Уровни: DEBUG (dev), INFO (prod), ERROR (всегда).
- Агрегация: Loki / CloudWatch / Yandex Cloud Logging.

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

```bash
# .env.example
DATABASE_URL=postgresql+asyncpg://user:pass@localhost:5432/markethacker
REDIS_URL=redis://localhost:6379/0
JWT_SECRET=change-me-in-production
JWT_ACCESS_TTL_MINUTES=15
JWT_REFRESH_TTL_DAYS=30
ENCRYPTION_KEY=change-me-32-bytes-key-here!!!!
ENVIRONMENT=development
LOG_LEVEL=DEBUG
CORS_ORIGINS=["chrome-extension://extension-id"]
```
