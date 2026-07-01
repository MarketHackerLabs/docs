# Технологический стек

## Основной стек

```
Python 3.11+
├── FastAPI          — API, OpenAPI, async
├── SQLAlchemy 2.0   — ORM
├── Alembic          — миграции
├── Pydantic v2      — валидация, settings
├── PostgreSQL 16    — основное хранилище
├── ClickHouse       — аналитика парсера (read-only из backend)
├── Redis            — сессии, rate limit, кэш, очереди
├── Celery / ARQ     — фоновые задачи (синк с MP)
└── uv               — управление зависимостями
```

## Обоснование выбора

### Python + FastAPI

- Async из коробки — важно для I/O-bound операций (запросы к API маркетплейсов).
- Автогенерация OpenAPI — контракт для расширения и будущего web-клиента.
- Pydantic v2 — строгая валидация на границах API.
- Экосистема для data/analytics задач.

### PostgreSQL

- Реляционная модель хорошо ложится на org → users → roles → marketplace accounts.
- Row Level Security (RLS) — дополнительный слой изоляции multi-tenancy.
- JSONB — для гибких метаданных без потери ACID.
- Зрелые managed-решения (RDS, Yandex Managed PostgreSQL, Supabase).

### Redis

- Хранение refresh token families / blacklist.
- Rate limiting (sliding window).
- Брокер для Celery/ARQ.
- **Response cache** — read-only данные (ClickHouse, каталоги PG) через `@cached_read`; общий для всех реплик API. См. [Кэширование](./caching.md).
- Ephemeral-токены: permissions, proxy sessions, billing counters.

### Celery / ARQ

- Синхронизация данных с маркетплейсами — длительные, периодические задачи.
- ARQ — легковесная альтернатива на старте; Celery — при росте нагрузки.

## Что сознательно не выбираем на MVP

| Технология | Причина отказа на старте |
|------------|--------------------------|
| Микросервисы | Преждевременная сложность |
| GraphQL | REST + OpenAPI достаточно для extension |
| MongoDB | Связи org/user/role/account — реляционные |

> **Parser** пишет в ClickHouse через Kafka (worker → Kafka → CH). **Backend** читает CH синхронным `clickhouse_connect` + `@cached_read(sync=True)` — без блокировки event loop. См. [Разработка парсеров](./parser-development.md), [Структура проекта](./structure.md).

## Связь с клиентами

| Клиент | Стек | Интеграция с бекендом |
|--------|------|------------------------|
| extension-chrome | React 19, TypeScript, MV3 | REST API, Bearer JWT, типы из OpenAPI |
| Web Dashboard (будущее) | TBD | Тот же API, OAuth2 PKCE |

## Инструменты разработки

| Инструмент | Назначение |
|------------|------------|
| ruff | Линтинг и форматирование Python |
| mypy | Статическая типизация |
| pytest | Тестирование |
| pre-commit | Хуки перед коммитом |
| Docker Compose | Локальная среда (API + PostgreSQL + Redis) |
| GitHub Actions | CI/CD |
