# Дизайн API

## Принципы

- **REST** с предсказуемыми URL и HTTP-методами.
- **Версионирование** через prefix: `/api/v1/`.
- **OpenAPI 3.1** — автогенерация из FastAPI.
- **camelCase в JSON** — все поля тел запросов и ответов (см. [Соглашение об именовании](#соглашение-об-именовании-json)).
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

Эндпоинта смены организации нет — токен не привязан к организации, см. [Контроль доступа](./access-control.md).

### Users

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/users/me` | ✓ | Текущий пользователь |
| PATCH | `/api/v1/users/me` | ✓ | Обновление профиля |

### Organizations

Прав в смысле permission-строк нет — либо пользователь владелец (`owner_id`), либо просто член. См. [Контроль доступа](./access-control.md).

| Метод | Путь | Auth | Кто может | Описание |
|-------|------|:----:|-----------|----------|
| GET | `/api/v1/organizations` | ✓ | любой | Список org пользователя (владелец + член) |
| POST | `/api/v1/organizations` | ✓ | любой | Создание org (лимит по тарифу) |
| GET | `/api/v1/organizations/{id}` | ✓ | член | Детали org |
| PATCH | `/api/v1/organizations/{id}` | ✓ | владелец | Обновление |
| DELETE | `/api/v1/organizations/{id}` | ✓ | владелец | Удаление org |
| GET | `/api/v1/organizations/{id}/members` | ✓ | член (фича `team_management`) | Список участников |
| DELETE | `/api/v1/organizations/{id}/members/{user_id}` | ✓ | владелец | Удаление участника |

### Marketplace Accounts

| Метод | Путь | Auth | Кто может | Описание |
|-------|------|:----:|-----------|----------|
| GET | `/api/v1/organizations/{org_id}/marketplace-accounts` | ✓ | член | Список кабинетов |
| GET | `.../marketplace-accounts/mine` | ✓ | член | Кабинеты, к которым привязан текущий пользователь |
| POST | `/api/v1/organizations/{org_id}/marketplace-accounts` | ✓ | владелец | Создание (`marketplace`, `displayName`) |
| PATCH | `/api/v1/organizations/{org_id}/marketplace-accounts/{id}` | ✓ | владелец | Обновление `displayName` |
| DELETE | `/api/v1/organizations/{org_id}/marketplace-accounts/{id}` | ✓ | владелец | Деактивация кабинета |
| POST | `.../capture-init` | ✓ | владелец | Guided Connect: `captureToken`, `connectUrl`, `suppliersUrl`, `snippet` |
| POST | `.../select-supplier` | ✓ | владелец | Явный выбор организации WB для уже привязанной сессии |
| GET | `.../credentials-status` | ✓ | владелец | `status`, `hasActiveSession`, `supplierId`, `lastVerifiedAt` |
| GET/POST/DELETE | `.../access/{user_id}` | ✓ | владелец | Список/выдача/отзыв доступа к кабинету |
| GET/PUT | `.../access/{user_id}/sections/{key}` | ✓ | владелец (себя может смотреть сам участник) | Права на разделы WB |

### Invitations

| Метод | Путь | Auth | Кто может | Описание |
|-------|------|:----:|-----------|----------|
| GET | `/api/v1/organizations/{org_id}/invitations` | ✓ | владелец | Ожидающие приглашения |
| POST | `/api/v1/organizations/{org_id}/invitations` | ✓ | владелец | Приглашение с `accountGrants` (кабинеты + разделы) |
| DELETE | `/api/v1/organizations/{org_id}/invitations/{id}` | ✓ | владелец | Отзыв приглашения |
| GET | `/api/v1/invitations/preview/{token}` | — | — | Превью приглашения (публично) |
| POST | `/api/v1/invitations/accept` | — | — | Принятие приглашения (существующий user) |
| POST | `/api/v1/invitations/accept-register` | — | — | Регистрация + принятие приглашения |

### WB Gateway (уже подключённый кабинет)

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| POST | `/api/v1/wb-gateway/handshake` | ✓ | Открыть кабинет: JWT-сессия (jti в Redis), `callbackUrl`, `gatewaySessionToken` (для extension) |
| GET | `/api/v1/wb-gateway/auth/callback` | — (одноразовый token) | Обмен callback-токена на cookie `mh_gw_session` |
| DELETE | `/api/v1/wb-gateway/session` | cookie | Logout — отзыв текущей gateway-сессии |
| ANY | `/api/v1/wb-gateway/*` | cookie **или** `Authorization: Bearer <gatewaySessionToken>` | Reverse proxy к seller.wildberries.ru (default-deny ACL) |

> **Browser extension:** после `handshake` сохраните `gatewaySessionToken` и вызывайте WB API через `https://wb-proxy.markethacker.ru/__content__/ns/...` с заголовком `Authorization: Bearer <gatewaySessionToken>` и `Origin: chrome-extension://<id>` (id должен быть в `CORS_ORIGINS`). Upstream-заголовки и cookies WB подставляет сервер из vault — extension не передаёт секреты WB.

### WB Connect (Guided Connect — привязка кабинета)

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/wb-connect/{token}` | — (opaque token) | Начать Guided Connect (Set-Cookie `mh_wb_connect`), редирект на onboarding-поддомен |
| GET | `/api/v1/wb-connect/{token}/done` | — | Страница успеха, `postMessage` в opener |
| GET | `/api/v1/wb-connect/{token}/suppliers` | — (opaque token) | Список организаций WB, обнаруженных во время connect |
| POST | `/api/v1/wb-connect/capture/{token}` | — (opaque token, CORS ограничен) | Одноразовый endpoint захвата (auto-script / DevTools-сниппет) |
| ANY | `*.wb-connect.markethacker.ru/*` | cookie `mh_wb_connect` | Onboarding subdomain-прокси (Host-based dispatch, не FastAPI-роутинг) |

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

Данные парсера из ClickHouse. **Платформенные** — без привязки к org или кабинету MP. Доступ определяется **личным тарифом пользователя**, а не ролью: требуется фича `search_tags` в `billing_plans.features` (включена на тарифах `pro` / `enterprise`). См. [Контроль доступа](./access-control.md).

| Метод | Путь | Auth | Требование | Описание |
|-------|------|:----:|------------|----------|
| GET | `/api/v1/search-tags/queries` | ✓ | фича `search_tags` | Постраничный список запросов с частотой |
| GET | `/api/v1/search-tags/queries/monthly` | ✓ | фича `search_tags` | Monthly-аналитика по точному query |

**Query-параметры списка:** `interval`, `since`, `until`, `query` (подстрока), `page`, `limit`.

**Monthly-ответ:** метрики с `MetricDelta` (`value`, `previous`, `diff`, `diffPercent`) — `frequency`, `ctr`, конверсии, `orders`, `avgFrequency`, `subjectsWithOrders`, `topOrderSubject`. CTR = `openToCard` / `frequency` × 100.

Источник: таблица `wb_search_tags` (+ join `wb_subjects_dict` для названия предмета). Кэш: `@cached_read` (TTL 30 / 15 мин). См. [Кэширование](./caching.md), [Parser Service](./parser.md).

### Admin — Parser / ClickHouse

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/admin/parser/wb-search-tags` | superuser | Список строк из ClickHouse с фильтрами org/account/auth_profile |
| POST | `/api/v1/admin/parser/jobs` | superuser | Постановка задачи парсера (`job_type`: `wb_search_tags`, `wb_market_niche`, `wb_subjects`, …) |
| POST | `/api/v1/admin/parser/jobs/wb-search-tags` | superuser | Deprecated — использовать `/admin/parser/jobs` |

Типы задач и поля формы: `GET /api/v1/admin/parser/job-types` (прокси к parser-сервису).  
Parser-сервис напрямую: `POST /api/v1/jobs/wb-market-niche`, `POST /api/v1/jobs/wb-search-tags`.

### Health

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/health` | — | Liveness probe |
| GET | `/ready` | — | Readiness probe (DB + Redis) |

## Соглашение об именовании JSON

Все **тела запросов и ответов** REST API сериализуются в **camelCase**. В Python-коде поля Pydantic-схем остаются в snake_case; преобразование выполняется автоматически через базовый `CamelModel` (`alias_generator=to_camel`, `populate_by_name=True`).

| Контекст | Стиль | Пример |
|----------|-------|--------|
| JSON request/response | camelCase | `accessToken`, `pageSize`, `fullName` |
| Query-параметры URL | snake_case | `page_size`, `date_from`, `sort_by` |
| Значения enum / status | как в коде | `past_due`, `limits_boost`, `trialing` |
| JWT payload (claims) | snake_case для кастомных | `is_superadmin` |
| PostgreSQL / Python domain | snake_case | `display_name`, `owner_id` |

**Обратная совместимость:** благодаря `populate_by_name=True` клиенты могут отправлять тела запросов и в snake_case, и в camelCase. Ответы API всегда в camelCase.

**Реализация (backend):**

```python
# markethacker/shared/schemas.py
class CamelModel(BaseModel):
    model_config = ConfigDict(
        alias_generator=to_camel,
        populate_by_name=True,
    )
```

FastAPI: `response_model_by_alias=True` на уровне приложения. Ошибки и ручная сериализация — через `serialize_model()` с `by_alias=True`.

**Клиенты:** `admin-panel`, `manager-portal` и будущее extension используют camelCase в TypeScript-типах (`accessToken`, `createdAt`, …). Query-параметры в admin-panel конвертируются в snake_case при сборке URL (`buildQuery`).

## Формат ответов

### Успех

```json
{
  "data": { ... },
  "meta": {
    "requestId": "uuid"
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
    "pageSize": 50,
    "hasNext": true
  }
}
```

**Cursor-пагинация** (планируется для больших списков с MP-sync):

```json
{
  "data": [ ... ],
  "meta": {
    "cursor": "next_cursor_value",
    "hasMore": true
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
    "requestId": "uuid"
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

Пример ответа `POST /auth/login`:

```json
{
  "accessToken": "eyJ...",
  "refreshToken": "opaque-refresh-token",
  "tokenType": "bearer"
}
```

## Request ID

Каждый запрос получает `X-Request-ID` (генерируется middleware, если не передан клиентом). Возвращается в `meta.requestId` — для корреляции логов и поддержки.

## OpenAPI

- Доступен по `/openapi.json` и `/docs` (Swagger UI).
- В production `/docs` отключён или защищён.
- Tags соответствуют модулям: `auth`, `users`, `organizations`, `marketplace`, `search-tags`, `billing`, `admin`.
