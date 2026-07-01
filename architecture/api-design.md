# Дизайн API

## Принципы

- **REST** с предсказуемыми URL и HTTP-методами.
- **Версионирование** через prefix: `/api/v1/`.
- **OpenAPI 3.1** — автогенерация из FastAPI.
- **Единый формат ошибок** на всех эндпоинтах.
- **Пагинация** через cursor-based pagination (для больших списков).

## Версионирование

```
/api/v1/...
```

При breaking changes — `/api/v2/`. Старая версия поддерживается минимум 6 месяцев.

## Группы эндпоинтов (MVP)

### Auth

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| POST | `/api/v1/auth/register` | — | Регистрация |
| POST | `/api/v1/auth/login` | — | Вход |
| POST | `/api/v1/auth/refresh` | — | Обновление токена |
| POST | `/api/v1/auth/logout` | ✓ | Выход |
| POST | `/api/v1/auth/switch-org` | ✓ | Смена организации |

### Users

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/users/me` | ✓ | Текущий пользователь |
| PATCH | `/api/v1/users/me` | ✓ | Обновление профиля |

### Organizations

| Метод | Путь | Auth | Permission | Описание |
|-------|------|:----:|------------|----------|
| GET | `/api/v1/organizations` | ✓ | — | Список org пользователя |
| POST | `/api/v1/organizations` | ✓ | — | Создание org |
| GET | `/api/v1/organizations/{id}` | ✓ | member | Детали org |
| PATCH | `/api/v1/organizations/{id}` | ✓ | `org:manage` | Обновление |
| POST | `/api/v1/organizations/{id}/members` | ✓ | `members:invite` | Приглашение |
| DELETE | `/api/v1/organizations/{id}/members/{user_id}` | ✓ | `members:manage` | Удаление участника |

### Marketplace Accounts

| Метод | Путь | Auth | Permission | Описание |
|-------|------|:----:|------------|----------|
| GET | `/api/v1/organizations/{org_id}/marketplace-accounts` | ✓ | member | Список кабинетов |
| POST | `/api/v1/organizations/{org_id}/marketplace-accounts` | ✓ | `marketplace_accounts:write` | Создание (`marketplace`, `display_name`) |
| PATCH | `/api/v1/organizations/{org_id}/marketplace-accounts/{id}` | ✓ | `marketplace_accounts:write` | Обновление `display_name` |
| DELETE | `/api/v1/organizations/{org_id}/marketplace-accounts/{id}` | ✓ | `marketplace_accounts:write` | Деактивация кабинета |
| POST | `.../capture-init` | ✓ | `marketplace_accounts:write` | Guided Connect: `capture_token`, `connect_url`, `snippet` |
| POST | `.../verify` | ✓ | `marketplace_accounts:read` | Проверка credentials |
| GET | `.../credentials-status` | ✓ | `marketplace_accounts:read` | `has_portal_session`, `portal_session_saved_at` |
| POST/DELETE | `.../access/{user_id}` | ✓ | `members:manage` | Назначение/отзыв менеджера на кабинет |
| GET/PUT | `.../section-access` | ✓ | `section_permissions:manage` | Права на разделы WB |

### Invitations

| Метод | Путь | Auth | Permission | Описание |
|-------|------|:----:|------------|----------|
| GET | `/api/v1/organizations/{org_id}/invitations` | ✓ | `members:manage` | Ожидающие приглашения |
| POST | `/api/v1/organizations/{org_id}/invitations` | ✓ | `members:invite` | Приглашение с role + account grants |
| DELETE | `/api/v1/organizations/{org_id}/invitations/{id}` | ✓ | `members:manage` | Отзыв приглашения |
| GET | `/api/v1/invitations/preview/{token}` | — | — | Превью приглашения (публично) |
| POST | `/api/v1/invitations/accept` | — | — | Принятие приглашения (существующий user) |
| POST | `/api/v1/invitations/accept-register` | — | — | Регистрация + принятие приглашения |

### WB Portal Proxy

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| POST | `/api/v1/proxy/web-handshake` | ✓ | Создать proxy-сессию, получить `callback_url` |
| GET | `/api/v1/proxy/portal/connect/{token}` | — | Начать Guided Connect (Set-Cookie `mh_wb_connect`) |
| GET | `/api/v1/proxy/portal/connect/{token}/done` | — | Страница успеха, `postMessage` в opener |
| GET | `/api/v1/proxy/portal/*` | cookie | Reverse proxy к seller.wildberries.ru |
| POST | `/api/v1/proxy/capture/{token}` | — | Одноразовый endpoint захвата (auto-script / snippet) |

### Billing

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/billing/plans` | — | Список тарифов |
| GET | `/api/v1/billing/subscription` | ✓ | Текущая подписка |
| POST | `/api/v1/billing/subscription/upgrade` | ✓ | Оформление подписки (checkout) |
| POST | `/api/v1/billing/subscription/cancel` | ✓ | Отмена подписки |
| GET | `/api/v1/billing/usage` | ✓ | Использование лимитов |
| GET | `/api/v1/billing/payment-methods` | ✓ | Сохранённые карты |
| DELETE | `/api/v1/billing/payment-methods/{id}` | ✓ | Удаление карты |
| POST | `/api/v1/billing/payments/{payment_id}/verify` | ✓ | Ручная сверка платежа ЮKassa |
| POST | `/api/v1/billing/promo/validate` | ✓ | Проверка промокода |
| POST | `/api/v1/billing/promo/redeem` | ✓ | Активация промокода (trial / free_period / limits_boost) |
| POST | `/api/v1/billing/webhooks/yookassa` | IP | Webhook ЮKassa |
| POST | `/api/v1/billing/webhooks/stripe` | Signature | Webhook Stripe |

> Подробнее: [Биллинг и оплата](./billing.md)

### Search Tags (WB)

Данные парсера из ClickHouse. **Платформенные** — без привязки к org или кабинету MP. Требуют право `search_tags:read` (все системные роли). На тарифах `pro` / `enterprise` включена фича `search_tags` в `billing_plans.features`.

| Метод | Путь | Auth | Permission | Описание |
|-------|------|:----:|------------|----------|
| GET | `/api/v1/search-tags/queries` | ✓ | `search_tags:read` | Постраничный список запросов с частотой |
| GET | `/api/v1/search-tags/queries/monthly` | ✓ | `search_tags:read` | Monthly-аналитика по точному query |

**Query-параметры списка:** `interval`, `since`, `until`, `query` (подстрока), `page`, `limit`.

**Monthly-ответ:** метрики с `MetricDelta` (value, previous, diff, diff_percent) — frequency, CTR, конверсии, orders, avg_frequency, subjects_with_orders, top_order_subject. CTR = open_to_card / frequency × 100.

Источник: таблица `wb_search_tags` (+ join `wb_subjects_dict` для названия предмета). Кэш: `@cached_read` (TTL 30 / 15 мин). См. [Кэширование](./caching.md), [Parser Service](./parser.md).

### Admin — Parser / ClickHouse

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/admin/parser/wb-search-tags` | superuser | Список строк из ClickHouse с фильтрами org/account/auth_profile |
| POST | `/api/v1/admin/parser/jobs/wb-search-tags` | superuser | Запуск job парсера (deprecated, использовать `/admin/parser/jobs`) |

### Health

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/health` | — | Liveness probe |
| GET | `/ready` | — | Readiness probe (DB + Redis) |

## Формат ответов

### Успех

```json
{
  "data": { ... },
  "meta": {
    "request_id": "uuid"
  }
}
```

### Список с пагинацией

**Offset-пагинация** (search tags, admin parser, часть admin CRUD):

```json
{
  "data": [ ... ],
  "meta": {
    "total": 1250,
    "page": 1,
    "page_size": 50,
    "has_next": true
  }
}
```

**Cursor-пагинация** (планируется для больших списков с MP-sync):

```json
{
  "data": [ ... ],
  "meta": {
    "cursor": "next_cursor_value",
    "has_more": true
  }
}
```

### Ошибка

```json
{
  "error": {
    "code": "PERMISSION_DENIED",
    "message": "Нет доступа к этому кабинету",
    "details": {}
  },
  "meta": {
    "request_id": "uuid"
  }
}
```

## Коды ошибок

| HTTP | Code | Когда |
|------|------|-------|
| 400 | `VALIDATION_ERROR` | Невалидные входные данные |
| 401 | `UNAUTHORIZED` | Нет / невалидный токен |
| 401 | `TOKEN_EXPIRED` | Access token истёк |
| 403 | `PERMISSION_DENIED` | Нет permission / доступа к ресурсу |
| 404 | `NOT_FOUND` | Ресурс не найден |
| 409 | `CONFLICT` | Дубликат (email, slug) |
| 429 | `RATE_LIMITED` | Превышен лимит запросов |
| 500 | `INTERNAL_ERROR` | Непредвиденная ошибка |

## Контракт с extension

### Генерация типов

```bash
# В extension-chrome
npx openapi-typescript https://api.markethacker.ru/openapi.json -o src/api/schema.d.ts
```

### API client в extension

```typescript
// src/api/client.ts
const API_BASE = 'https://api.markethacker.ru/api/v1';

async function apiClient<T>(path: string, options?: RequestInit): Promise<T> {
  const token = await getAccessToken();
  const response = await fetch(`${API_BASE}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${token}`,
      ...options?.headers,
    },
  });

  if (response.status === 401) {
    await refreshTokens();
    return apiClient(path, options); // retry once
  }

  if (!response.ok) {
    const error = await response.json();
    throw new ApiError(error.error);
  }

  return response.json();
}
```

## Request ID

Каждый запрос получает `X-Request-ID` (генерируется middleware, если не передан клиентом). Возвращается в `meta.request_id` — для корреляции логов и поддержки.

## OpenAPI

- Доступен по `/openapi.json` и `/docs` (Swagger UI).
- В production `/docs` отключён или защищён.
- Tags соответствуют модулям: `auth`, `users`, `organizations`, `marketplace`, `search-tags`, `billing`, `admin`.
