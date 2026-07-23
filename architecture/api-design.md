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
| POST | `/api/v1/auth/login` | — | Вход (при MFA → 403 `MFA_REQUIRED`) |
| POST | `/api/v1/auth/mfa/complete` | — | Завершение входа с TOTP |
| POST | `/api/v1/auth/refresh` | — | Обновление токена |
| POST | `/api/v1/auth/logout` | — | Выход (refresh + опционально access) |

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
| DELETE | `/api/v1/organizations/{id}/members/me` | ✓ | участник (не владелец) | Покинуть организацию |
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
| POST | `/api/v1/invitations/accept/{token}` | ✓ | приглашённый | Принятие (email аккаунта = invitation) |
| POST | `/api/v1/invitations/accept-register/{token}` | — | — | Регистрация + принятие; email из invitation |

### WB Gateway (уже подключённый кабинет)

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| POST | `/api/v1/wb-gateway/handshake` | ✓ | Открыть кабинет: JWT-сессия (jti в Redis), `callbackUrl`, `gatewaySessionToken` (для extension) |
| GET | `/api/v1/wb-gateway/auth/callback` | — (одноразовый token) | Обмен callback-токена на cookie `mh_gw_session` |
| DELETE | `/api/v1/wb-gateway/session` | cookie | Logout — отзыв текущей gateway-сессии |
| ANY | `/api/v1/wb-gateway/*` | cookie `mh_gw_session` | Reverse proxy к seller.wildberries.ru (default-deny ACL) |

> Browser-сессия кабинета: после `handshake` + `auth/callback` устанавливается
> httpOnly cookie `mh_gw_session`. Отдельный `Authorization: Bearer` к wb-proxy
> **не используется** (ранее описанный `gatewaySessionToken` снят из контракта).
> Upstream-заголовки и cookies WB подставляет сервер из vault.

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
| GET | `/api/v1/billing/plans` | — | Список тарифов (фильтр по `X-MarketHacker-Client`) |
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

#### Заголовок `X-MarketHacker-Client`

Клиенты платформы передают ключ приложения при запросе каталога тарифов,
оформлении подписки и promo-эндпоинтах:

```
X-MarketHacker-Client: manager_portal
GET /api/v1/billing/plans?client=manager_portal
```

Приоритет: заголовок, затем query `?client=`. Допустимые ключи — из справочника
`billing_clients` (admin API). Без контекста каталог не фильтруется (legacy).

| Ключ (seed) | Приложение |
|-------------|------------|
| `manager_portal` | Manager-portal (уже интегрирован) |
| `browser_extension` | Chromium-расширение (**нужно добавить заголовок на фронте**) |

При попытке оформить тариф, скрытый для клиента, API вернёт `400` с текстом в поле
`detail` (стандартный формат FastAPI `HTTPException`).

### Extension (browser)

Контракт для Chromium-расширения: агрегированные entitlements и статическая конфигурация клиента.

| Метод | Путь | Auth | Требование | Описание |
|-------|------|:----:|------------|----------|
| GET | `/api/v1/extension/entitlements` | ✓ | фича `browser_extension` | Доступные capabilities, статус подписки |
| GET | `/api/v1/extension/config` | — | — | API URLs, min version, poll interval |

Аутентификация расширения — стандартный JWT (`/auth/login` с `deviceId`). Доступ к
самому расширению проверяет guard эндпоинта (`browser_extension`): личный план
**или** seat от тарифа владельца org (см. [Контроль доступа](./access-control.md)).
Поле `capabilities` содержит только подфичи расширения (сейчас — `search_tags`).
WB Gateway и организации сюда не входят — отдельная зона manager-portal.
См. [Аутентификация](./authentication.md).

**Пример `GET /extension/entitlements`:**

```json
{
  "data": {
    "account": { "status": "active" },
    "subscription": {
      "planName": "pro",
      "status": "active",
      "currentPeriodEnd": "2026-08-07T00:00:00Z",
      "cancelledAt": null,
      "trialEndsAt": null,
      "isInGracePeriod": false
    },
    "features": ["browser_extension", "search_tags"],
    "capabilities": {
      "search_tags": { "enabled": true }
    },
    "cacheTtlSeconds": 60,
    "serverTime": "2026-07-08T00:00:00Z"
  }
}
```

`features` — эффективный набор (`resolve_user_feature_keys`); `subscription.planName` —
личный план пользователя (источник billing), даже если часть фич пришла через seat.

### Product promotions (баннеры)

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/promotions/active` | ✓ | Активные баннеры (`placement`, опционально `orgId`) |
| POST | `/api/v1/promotions/{key}/dismiss` | ✓ | Скрыть баннер |
| CRUD | `/api/v1/admin/promotions` | superuser | Управление кампаниями |

Placement: `{client}.{slot}` (`manager_portal.all` / `.dashboard` / `.team`; далее —
`browser_extension.*`). Подробнее: [Продуктовые промо](./product-promotions.md).

### Партнёры

| Метод | Путь | Auth | Описание |
|-------|------|------|----------|
| GET/POST | `/api/v1/partners/public/r/{code}` | — | Resolve + click + cookie; client bonus без партнёрских % |
| GET | `/api/v1/partners/me` | ✓ partner | Профиль (API; UI-кабинета в Manager Portal нет) |
| GET | `/api/v1/partners/me/campaigns` | ✓ partner | Мои кампании |
| GET | `/api/v1/partners/me/analytics` | ✓ partner | Сводка |
| CRUD | `/api/v1/admin/partners` | superuser | Admin Panel: партнёры и кампании |

Клиентский путь в Manager Portal: `?ref=` на `/register` + промокод на `/billing`.
Подробнее: [Партнёры](./partners.md).

**Поля ответа entitlements:**

| Поле | Описание |
|------|----------|
| `features` | Эффективные ключи (`resolve_user_feature_keys`: личный ∪ org seat) |
| `capabilities` | Подфичи расширения: `enabled` + `reason` при отказе |
| `subscription` | Личный план / статус / grace period после отмены |

**Capabilities (расширяемый каталог в коде):**

| Ключ | Требует (billing features) | Описание |
|------|---------------------------|----------|
| `search_tags` | `browser_extension` + `search_tags` | Поисковые запросы WB в расширении |

При `403` на `/extension/entitlements` — нет фичи `browser_extension` (лично и через seat)
или истёк оплаченный период после отмены.

#### Использование во frontend (manager-portal, admin-panel)

**Manager-portal** — для UX-гейтинга достаточно `GET /users/me`:

```typescript
import { planHasFeature, PLAN_FEATURE_BROWSER_EXTENSION } from "@/lib/plan-features";

const hasExtension = planHasFeature(me?.session.planFeatures, PLAN_FEATURE_BROWSER_EXTENSION);
```

`session.planFeatures` и `features` в entitlements — один и тот же набор ключей;
в web-клиенте удобнее брать из `/users/me`.

Для UI статуса подписки / grace period — `GET /extension/entitlements`
(поле `subscription`).

**Admin-panel** — фича в `FeaturesPicker` через `GET /admin/billing/features`, без доработок UI.

### Search Tags (WB)

Данные парсера из ClickHouse. **Платформенные** — без привязки к org или кабинету MP. Доступ определяется **личным тарифом пользователя**, а не ролью: требуется фича `search_tags` в `billing_plans.features` (включена на тарифах `pro` / `enterprise`). См. [Контроль доступа](./access-control.md).

| Метод | Путь | Auth | Требование | Описание |
|-------|------|:----:|------------|----------|
| GET | `/api/v1/search-tags/queries` | ✓ | фича `search_tags` | Постраничный список запросов с частотой |
| GET | `/api/v1/search-tags/queries/monthly` | ✓ | фича `search_tags` | Monthly-аналитика по точному query |
| GET | `/api/v1/search-tags/queries/frequency-trends` | ✓ | фича `search_tags` | Тренды частотности по точному query |

**Query-параметры списка:** `interval`, `since`, `until`, `query` (подстрока), `page`, `limit`.

**Monthly-ответ:** метрики с `MetricDelta` (`value`, `previous`, `diff`, `diffPercent`) — `frequency`, `ctr`, конверсии, `orders`, `avgFrequency`, `subjectsWithOrders`, `topOrderSubject`. CTR = `openToCard` / `frequency` × 100.

**Frequency-trends:** `query` (обязательный), `series` (`days_30` по умолчанию | `months_12`). Ответ: `yesterday`, `latestMonth.frequency` (`MetricDelta` из WB `frequency`/`frequency_prev`), dense `points` (`frequency: null` на пропуски), `granularity` = `day`|`month`. Без данных по query — `404` (`NOT_FOUND`), как у monthly. Значения только из готовых агрегатов WB (`interval=yesterday` / `month`), без суммирования дневных в месячные.

Пример:

```http
GET /api/v1/search-tags/queries/frequency-trends?query=кроссовки&series=days_30
```

```json
{
  "query": "кроссовки",
  "series": "days_30",
  "granularity": "day",
  "yesterday": { "periodStart": "2026-07-20", "frequency": 4210 },
  "latestMonth": {
    "periodStart": "2026-06-01",
    "periodEnd": "2026-06-30",
    "parsedAt": "2026-07-01T09:00:00Z",
    "frequency": { "value": 118782, "previous": 113660, "diff": 5122, "diffPercent": 4.51 }
  },
  "points": [
    { "periodStart": "2026-06-21", "frequency": null },
    { "periodStart": "2026-07-20", "frequency": 4210 }
  ]
}
```

Источник: таблица `wb_search_tags` (+ join `wb_subjects_dict` для названия предмета в monthly). Кэш: `@cached_read` (TTL 30 / 15 мин; trends — 15 мин). См. [Кэширование](./caching.md), [Parser Service](./parser.md).

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
- Tags соответствуют модулям: `auth`, `users`, `organizations`, `marketplace`, `search-tags`, `billing`, `promotions`, `partners`, `extension`, `admin`.
