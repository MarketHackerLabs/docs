# Структура проекта

## Дерево каталогов

```
backend/
├── src/
│   └── markethacker/
│       ├── main.py                 # FastAPI app factory
│       ├── config/                 # Settings (pydantic-settings)
│       ├── shared/                 # Общие утилиты, exceptions, types
│       ├── infrastructure/         # DB, Redis, encryption, external clients
│       │   ├── database/
│       │   ├── cache/                # generic: redis, json_cache, @cached_read
│       │   ├── clickhouse/
│       │   ├── security/
│       │   └── jobs/
│       └── modules/                # Bounded contexts
│           ├── auth/
│           ├── users/
│           ├── organizations/
│           ├── marketplace_accounts/
│           ├── search_tags/
│           ├── billing/
│           ├── admin/
│           └── proxy/
├── alembic/
├── tests/
├── docker/
├── pyproject.toml
└── README.md
```

## Структура модуля (bounded context)

Каждый модуль в `modules/` следует единому шаблону:

```
modules/auth/
├── api/              # routers, request/response schemas
├── application/      # use cases (services)
├── domain/           # entities, value objects, domain events
└── infrastructure/   # repositories, cached/*, clickhouse/queries
```

Для кэшируемых read-операций политика объявляется **рядом с источником данных** (`@cached_read` в query-файле или `infrastructure/cached/<op>.py`), а не в общем `cache.py` модуля. Подробнее: [Кэширование](./caching.md).

### Слои и зависимости

```mermaid
flowchart BT
    API[api] --> APP[application]
    APP --> DOM[domain]
    INF[infrastructure] --> DOM
    APP --> INF

    style DOM fill:#e8f5e9
    style API fill:#e3f2fd
    style APP fill:#fff3e0
    style INF fill:#fce4ec
```

| Слой | Ответственность | Зависит от |
|------|-----------------|------------|
| `domain` | Бизнес-правила, сущности | Ничего (чистый слой) |
| `application` | Use cases, оркестрация | `domain` |
| `infrastructure` | БД, внешние API, кэш | `domain` (реализует интерфейсы) |
| `api` | HTTP, валидация входа/выхода | `application` |

**Правило:** `domain` не импортирует `infrastructure` и `api`.

## Модули и их зона ответственности

| Модуль | Зона ответственности |
|--------|----------------------|
| `auth` | Login, logout, refresh, MFA, управление сессиями |
| `users` | Профиль пользователя, настройки |
| `organizations` | CRUD организаций, роли, приглашения, membership |
| `marketplace_accounts` | Привязка WB/Ozon, хранение credentials |
| `search_tags` | Поисковые запросы WB (ClickHouse, read-only) |
| `billing` | Подписки, тарифы, лимиты, промокоды, ЮKassa/Stripe |
| `admin` | Админ-панель, управление тарифами и парсером |
| `proxy` | WB Portal reverse proxy |

## Shared и Infrastructure

### `shared/`

- Базовые исключения (`NotFoundError`, `PermissionDeniedError`)
- Общие типы (`UUID`, пагинация)
- Утилиты (datetime, id generation)

### `infrastructure/`

- `database/` — SQLAlchemy engine, session factory, base model
- `cache/` — Redis client, JSON cache-aside, декоратор `@cached_read` (см. [Кэширование](./caching.md))
- `clickhouse/` — read-only клиент для данных парсера
- `security/` — JWT, encryption, password hashing
- `jobs/` — ARQ worker

## Конфигурация

Настройки через `pydantic-settings`, загрузка из env:

```python
# config/settings.py
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env")

    database_url: str
    redis_url: str
    jwt_secret: str
    jwt_access_ttl_minutes: int = 15
    jwt_refresh_ttl_days: int = 30
    encryption_key: str
    cache_enabled: bool = True   # TTL — в @cached_read у каждой операции
```

Секреты никогда не коммитятся — только `.env.example` с описанием переменных.

## Тесты

```
tests/
├── unit/           # domain, application (без БД)
├── integration/    # API + PostgreSQL (testcontainers)
└── conftest.py     # fixtures
```
