# Модель доступа к кабинетам маркетплейсов

Документ описывает трёхуровневую модель контроля доступа к кабинетам Wildberries, Ozon и других маркетплейсов в MarketHacker.

## Обзор

MarketHacker использует **трёхуровневую** систему доступа, а не классический «пользователь → организация → роль»:

```mermaid
flowchart TB
    subgraph tier1 [Уровень 1 — Организация]
        MEM[Membership + Role]
        PERM[Permissions]
    end

    subgraph tier2 [Уровень 2 — Кабинет MP]
        ACC[Привязка к кабинету]
        ACC --> MPA[marketplace_account_id]
        ACC --> HIER[Иерархия org / partner]
    end

    subgraph tier3 [Уровень 3 — Разделы кабинета]
        SEC[section_permissions]
        SEC --> BALANCE[balance]
        SEC --> FIN[fin_analytics]
        SEC --> ADS[ads]
        SEC --> OTHER[supplies, brands, ...]
    end

    USER[User] --> tier1
    USER --> tier2
    USER --> tier3
```

| Уровень | Что контролирует | Где проверяется |
|---------|------------------|-----------------|
| **1. Org RBAC** | Доступ к функциям MarketHacker, управление командой | Backend API, web admin |
| **2. Account binding** | К какому кабинету продавца привязан пользователь | `marketplace_account_id` в JWT |
| **3. Section permissions** | Какие разделы кабинета MP видит менеджер | Extension / MP Proxy + `section_permissions` |

---

## Уровень 1: RBAC в организации

Роли и permissions на уровне организации — см. [Контроль доступа](./access-control.md).

| Роль | Назначение |
|------|------------|
| `owner` | Полный доступ, биллинг, удаление org |
| `admin` | Управление командой, кабинетами MP, section permissions |
| `manager` | Работа с аналитикой и рекламой в назначенных кабинетах |
| `viewer` | Только чтение |

### Иерархия партнёров / агентств

Для модели «агентство управляет клиентами»:

- **Parent org** — агентство; видит sub-orgs или linked accounts.
- **Sub-org** — клиент агентства.
- Admin агентства назначает менеджеров на конкретные кабинеты клиентов.

---

## Уровень 2: Привязка к кабинету продавца

Пользователь привязывается к одному или нескольким кабинетам MP через `user_marketplace_accounts`:

```json
{
  "user_id": "uuid",
  "org_id": "uuid",
  "marketplace_account_id": "uuid",
  "marketplace": "wildberries",
  "display_name": "ООО \"Звезда\"",
  "section_permissions": ["fin_analytics", "ads"]
}
```

| Поле | Смысл |
|------|-------|
| `marketplace_account_id` | ID кабинета MP в MarketHacker |
| `marketplace` | `wildberries`, `ozon`, ... |
| `display_name` | Отображаемое имя (юр. лицо) |

Назначение кабинета:

```
PUT /api/v1/members/{user_id}/marketplace-accounts
{ "marketplace_account_id": "uuid" }
```

Без привязки к кабинету пользователь **не может** работать с данными этого MP — UI показывает ошибку «Кабинет не назначен».

### MarketplaceAccount

Кабинет хранит credentials для доступа к MP:

- API-ключ / OAuth-токены
- Cookies сессии (сбор через extension)
- Проверка валидности credentials: `POST /marketplace-accounts/{id}/verify`

---

## Уровень 3: Section permissions (разделы кабинета)

Гранулярные права **внутри** кабинета MP. Назначаются отдельно от org roles:

```
PUT /api/v1/members/{user_id}/section-permissions
{ "marketplace_account_id": "uuid", "sections": ["fin_analytics", "ads", "supplies"] }
```

### Разделы Wildberries

| section_key | Раздел кабинета |
|-------------|-----------------|
| `balance` | Баланс |
| `fin_analytics` | Финансовая аналитика |
| `prices_and_discounts` | Цены и скидки |
| `ads` | Реклама |
| `supplies` | Поставки |
| `brands` | Бренды |
| `jam_subscription` | Подписка Jam |
| `store_showcase` | Витрина магазина |
| `documents` | Документы |
| `reviews` | Отзывы и вопросы |
| `review_points` | Баллы за отзывы |
| `pinned_reviews` | Закреплённые отзывы |

**Пустой список `section_permissions`** — полный доступ ко всем разделам назначенного кабинета (policy по умолчанию для `owner` / `admin`).

Для Ozon — отдельный набор `marketplace_sections`, т.к. структура кабинета другая.

---

## Механизм MP Proxy: ограничение видимости разделов

Менеджер может работать с кабинетом MP **без прямого доступа** к credentials продавца — через reverse proxy:

```mermaid
sequenceDiagram
    participant M as Менеджер
    participant EXT as Extension / Web UI
    participant PROXY as wb-proxy.markethacker.ru
    participant API as api.markethacker.ru
    participant MP as seller.wildberries.ru

    M->>EXT: Login
    EXT->>API: POST /auth/login
    API-->>EXT: JWT + section_permissions

    M->>EXT: Открыть кабинет WB
    EXT->>PROXY: Открыть proxy с JWT

    PROXY->>API: Validate session, load permissions
    PROXY->>PROXY: Load MP credentials for marketplace_account_id

    M->>PROXY: GET /finances/...
    PROXY->>MP: Forward request with seller credentials
    MP-->>PROXY: HTML/JSON response
    PROXY->>PROXY: Filter by section_permissions
    PROXY-->>M: Modified page (only allowed sections)
```

### Компоненты proxy-слоя

| Компонент | Роль |
|-----------|------|
| **wb-proxy.markethacker.ru** | Reverse proxy перед `seller.wildberries.ru` |
| **Credentials MP** | Хранятся на сервере для `marketplace_account_id`; менеджер их не видит |
| **JWT пользователя** | Идентификация менеджера и его `section_permissions` |
| **Фильтрация** | Proxy скрывает пункты меню, блокирует URL, режет HTML/JS/API-ответы |

Без авторизации proxy редиректит на login (302).

### Сбор credentials (onboarding кабинета)

1. Admin создаёт `marketplace_account` с API-ключом или через OAuth.
2. Extension собирает cookies сессии MP (content script + background worker).
3. Extension отправляет credentials на backend через одноразовый **collection-token**.
4. Backend сохраняет credentials, привязанные к account.
5. Proxy использует эти credentials для всех менеджеров этого кабинета.

---

## API управления доступами

| Группа | Эндпоинты |
|--------|-----------|
| Auth | `POST /auth/login`, `POST /auth/logout`, `GET /auth/me` |
| Members | `GET /members`, `POST/DELETE .../roles` |
| Account binding | `PUT .../marketplace-accounts` |
| Section permissions | `PUT .../section-permissions` |
| Marketplace Accounts | `GET/POST /marketplace-accounts`, `verify`, `check-credentials` |

---

## Архитектура клиентов

MarketHacker имеет **Chromium-расширение** с `host_permissions` на `wildberries.ru` и `ozon.ru`.

```mermaid
flowchart TB
    subgraph backend [Backend MarketHacker]
        ORG[Organization]
        USER[User + Membership]
        MPA[MarketplaceAccount]
        MPR[MarketplaceSectionPermission]
    end

    subgraph clients [Клиенты]
        EXT[Chrome Extension]
        WEB[Web Admin — будущее]
        PROXY[MP Proxy — опционально]
    end

    USER --> ORG
    USER --> MPA
    USER --> MPR
    EXT -->|JWT + permissions| backend
    EXT -->|filter UI on MP pages| MP[wildberries.ru / ozon.ru]
    PROXY -->|alternative path| MP
```

### Два пути enforcement

#### Вариант A: Extension-first (рекомендуется для MVP)

1. Пользователь логинится в extension → получает JWT с `marketplace_account_id` и `section_permissions`.
2. Content script на `wildberries.ru` / `ozon.ru`:
   - скрывает пункты меню и виджеты по `section_permissions`;
   - блокирует навигацию на запрещённые URL;
   - перехватывает XHR/fetch и отклоняет запросы к запрещённым API MP.
3. Backend API дублирует проверки для своих эндпоинтов.

**Плюсы:** не нужен отдельный proxy, работает с нативным UI маркетплейса.  
**Минусы:** enforcement на клиенте; для строгой изоляции нужен server-side proxy.

#### Вариант B: MP Proxy

Отдельный сервис `wb-proxy.markethacker.ru` / `ozon-proxy.markethacker.ru`:

1. Reverse proxy перед кабинетом MP.
2. Credentials seller account хранятся на сервере.
3. Proxy фильтрует HTML/JS/API по `section_permissions`.
4. Extension или web UI открывает proxy URL.

**Плюсы:** строгий контроль, менеджер не имеет прямого доступа к seller credentials.  
**Минусы:** сложная инфраструктура, поддержка при изменении UI MP.

#### Рекомендация

| Этап | Подход |
|------|--------|
| MVP | **Extension-first** — permissions с backend, UI filtering в content script |
| v2 | **Proxy для WB** — для клиентов, которым нужна строгая изоляция credentials |
| v3 | **Proxy для Ozon** — marketplace-specific adapters |

---

## Модель данных

```sql
CREATE TABLE marketplace_sections (
    id           UUID PRIMARY KEY,
    marketplace  VARCHAR(20) NOT NULL,
    section_key  VARCHAR(50) NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    UNIQUE (marketplace, section_key)
);

CREATE TABLE user_marketplace_section_access (
    id                      UUID PRIMARY KEY,
    user_id                 UUID NOT NULL REFERENCES users(id),
    marketplace_account_id  UUID NOT NULL REFERENCES marketplace_accounts(id),
    section_key             VARCHAR(50) NOT NULL,
    granted_by              UUID REFERENCES users(id),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (user_id, marketplace_account_id, section_key)
);
```

### JWT payload

```json
{
  "sub": "user_uuid",
  "org_id": "org_uuid",
  "marketplace_account_id": "account_uuid",
  "marketplace": "wildberries",
  "section_permissions": ["fin_analytics", "ads", "supplies"],
  "permissions": ["analytics:read"]
}
```

---

## Связь с другими документами

| Документ | Содержание |
|----------|------------|
| [Контроль доступа](./access-control.md) | Org RBAC, permissions, алгоритм проверки |
| [Модель данных](./data-model.md) | ER-диаграмма, таблицы |
| [Аутентификация](./authentication.md) | JWT, refresh tokens |
