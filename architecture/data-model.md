# Модель данных

## Принцип

Пользователь не «владеет» бизнес-данными напрямую — данные принадлежат
**Organization**. Но правами управляет именно пользователь: организацией
владеет `Organization.owner_id`, а не «роль» в членстве. Пользователь входит
в организацию через **Membership** без каких-либо ролей — это лишь факт
принадлежности к команде; фактические права на кабинеты и разделы задаются
отдельными грантами (`UserMarketplaceAccount`, `UserMarketplaceSectionAccess`).
Подробности — в [Контроле доступа](./access-control.md).

## ER-диаграмма

```mermaid
erDiagram
    User ||--o{ Organization : owns
    User ||--o{ Membership : has
    Organization ||--o{ Membership : has
    Organization ||--o{ MarketplaceAccount : owns
    Organization ||--o{ OrganizationInvitation : issues
    MarketplaceAccount ||--o| MarketplaceCredentialVault : has
    User ||--o{ UserMarketplaceAccount : assigned
    MarketplaceAccount ||--o{ UserMarketplaceAccount : has
    User ||--o{ UserMarketplaceSectionAccess : has
    MarketplaceAccount ||--o{ MarketplaceSection : defines
    User ||--o{ RefreshToken : has
    User ||--o{ AuditLog : triggers

    User {
        uuid id PK
        string email UK
        string password_hash
        bool is_active
        bool is_superadmin
        bool mfa_enabled
        datetime created_at
        datetime updated_at
    }

    Organization {
        uuid id PK
        string name
        string slug UK
        uuid owner_id FK
        bool is_active
        datetime created_at
    }

    Membership {
        uuid id PK
        uuid user_id FK
        uuid org_id FK
        bool is_active
        datetime joined_at
    }

    OrganizationInvitation {
        uuid id PK
        uuid org_id FK
        string email
        string token_hash UK
        string status
        jsonb account_grants
        datetime expires_at
    }

    MarketplaceAccount {
        uuid id PK
        uuid org_id FK
        enum marketplace
        string supplier_id
        string display_name
        enum status
        datetime last_status_changed_at
        datetime created_at
    }

    MarketplaceCredentialVault {
        uuid id PK
        uuid account_id FK "unique — один активный ряд на кабинет"
        bytes encrypted_payload
        bytes nonce
        int key_version
        datetime access_token_expires_at
        datetime last_verified_at
        datetime last_refreshed_at
        int version "optimistic concurrency"
        datetime created_at
        datetime updated_at
    }

    RefreshToken {
        uuid id PK
        uuid user_id FK
        string token_hash UK
        string family_id
        string device_id
        datetime expires_at
        datetime revoked_at
    }

    AuditLog {
        uuid id PK
        uuid user_id FK
        uuid org_id FK
        string action
        string resource_type
        uuid resource_id
        jsonb metadata
        datetime created_at
    }
```

## Сущности

### User

Глобальная учётная запись. Один пользователь может состоять в нескольких организациях.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | UUID | Первичный ключ |
| `email` | string | Уникальный, используется для входа |
| `password_hash` | string | bcrypt / argon2 |
| `is_active` | bool | Деактивация аккаунта |
| `mfa_enabled` | bool | Включена ли двухфакторная аутентификация |

### Organization

Тенант — изолированная единица данных. Все marketplace accounts и аналитика привязаны к org. Тариф организации определяется биллинг-планом её `owner_id`; лимиты org (участники, API) могут быть увеличены докупками владельца (см. [Биллинг](./billing.md)).

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | UUID | Первичный ключ |
| `name` | string | Отображаемое имя |
| `slug` | string | URL-safe идентификатор |
| `owner_id` | UUID | FK → User. Единолично управляет org (см. [Контроль доступа](./access-control.md)) |
| `is_active` | bool | Деактивация без удаления |

### Membership

Связь user ↔ organization. Не хранит роль и не даёт прав — только факт членства в команде; см. [Контроль доступа](./access-control.md).

| Поле | Тип | Описание |
|------|-----|----------|
| `user_id` | UUID | FK → User |
| `org_id` | UUID | FK → Organization |
| `is_active` | bool | Деактивация без удаления |

### OrganizationInvitation

Приглашение по email с заранее заданными грантами на кабинеты (`account_grants`), которые применяются при принятии приглашения.

| Поле | Тип | Описание |
|------|-----|----------|
| `org_id` | UUID | FK → Organization |
| `email` | string | Email приглашённого |
| `token_hash` | string | Хэш одноразового токена |
| `status` | string | `pending` / `accepted` / `revoked` / `expired` |
| `account_grants` | jsonb | Список `{marketplace_account_id, sections}` |
| `expires_at` | datetime | TTL 7 дней |

### MarketplaceAccount

Привязанный кабинет продавца на маркетплейсе.

| Поле | Тип | Описание |
|------|-----|----------|
| `marketplace` | enum | `wildberries`, `ozon` |
| `supplier_id` | string | Реальный идентификатор продавца на маркетплейсе (для WB — `x-supplier-id`); пусто до успешного Guided Connect (`draft`/`connecting`) |
| `display_name` | string | Человекочитаемое имя кабинета (отображается в UI и в прокси WB) |
| `status` | enum | Жизненный цикл: `draft` → `connecting` → `active` / `expired` / `revoked` → `archived`. Источник истины для `wb_gateway` (см. [WB Gateway & Guided Connect](./wb-portal-proxy.md#жизненный-цикл-кабинета-cabinetstatus)) |
| `last_status_changed_at` | datetime | Момент последнего перехода статуса |

**Ограничение:** частичный unique index `(org_id, marketplace, supplier_id)` — применяется только к живым кабинетам (`status != 'archived'`) с уже известным `supplier_id` (`supplier_id != ''`). Несколько незавершённых `draft`/`connecting` попыток подключения одновременно разрешены осознанно (пользователь мог открыть форму в двух вкладках); дублей по одинаковому `supplier_id` в одной организации после подключения быть не может.

### UserMarketplaceAccount

Привязка пользователя к кабинету MP (уровень 2 доступа).

| Поле | Тип | Описание |
|------|-----|----------|
| `user_id` | UUID | FK → User |
| `marketplace_account_id` | UUID | FK → MarketplaceAccount |
| `is_default` | bool | Кабинет по умолчанию для пользователя |

### UserMarketplaceSectionAccess

Гранулярные права на разделы кабинета (уровень 3).

| Поле | Тип | Описание |
|------|-----|----------|
| `user_id` | UUID | FK → User |
| `marketplace_account_id` | UUID | FK → MarketplaceAccount |
| `section_key` | string | `growth`, `products`, `shipments`, `analytics`, `promotion`, `finances` (WB) |
| `can_read` | bool | Доступ на чтение раздела |
| `can_write` | bool | Доступ на изменение (где применимо) |

См. [Модель доступа к кабинетам MP](./marketplace-access-model.md).

### MarketplaceCredentialVault

Зашифрованные credentials WB portal-сессии кабинета. В отличие от предыдущей модели (append-лог `MarketplaceCredential`) — **один активный ряд на кабинет**, обновляемый на месте.

| Поле | Тип | Описание |
|------|-----|----------|
| `encrypted_payload` | bytes | AES-256-GCM ciphertext (`authorizev3`, `cookies`, `local_storage`) |
| `nonce` | bytes | Nonce для GCM |
| `key_version` | int | Версия ключа шифрования |
| `access_token_expires_at` | datetime \| null | Срок жизни `authorizev3` из JWT `exp`; у многих WB JWT `exp` нет → `NULL`, refresh по возрасту (`last_refreshed_at` / `updated_at`, `wb_token_max_age_seconds`) |
| `last_verified_at` / `last_refreshed_at` | datetime | Для `credentials-status`; `last_refreshed_at` — якорь для slide-v3 без `exp` |
| `version` | int | Optimistic concurrency: обновление — `UPDATE ... WHERE account_id = :id AND version = :v`; 0 affected rows означает конкурентную запись, вызывающий код обязан повторить операцию |

Подробности шифрования и жизненного цикла — [WB Gateway & Guided Connect](./wb-portal-proxy.md#хранение-credentials-marketplacecredentialvault).

### Биллинг (кратко)

Подробности — в [Биллинг и оплата](./billing.md). Основные таблицы PostgreSQL:

| Таблица | Назначение |
|---------|------------|
| `billing_plans` | Каталог тарифов |
| `billing_clients` | Справочник клиентов платформы (ключи для `X-MarketHacker-Client`) |
| `billing_plan_client_visibility` | M:N: тариф виден в клиенте (`plan_id`, `client_id`). Пустая связь = тариф не в каталоге клиента |
| `billing_subscriptions` | Подписка пользователя (1:1 с `users`) |
| `billing_payments` | Платежи ЮKassa / Stripe |
| `billing_limit_addon_products` | Каталог продуктов докупки лимитов |
| `billing_limit_addon_entitlements` | Активные докупки пользователя |
| `promo_codes` / `promo_code_redemptions` | Промокоды и их использование |

Миграции: `20260704_0019_limit_addons`, `20260704_0020_limit_addon_billing_modes`, `20260708_0023_plan_client_visibility`.

## Индексы

```sql
-- Быстрый поиск membership
CREATE INDEX idx_membership_user_org ON memberships(user_id, org_id);

-- Фильтрация аккаунтов по организации
CREATE INDEX idx_marketplace_account_org ON marketplace_accounts(org_id);

-- Audit log по организации и времени
CREATE INDEX idx_audit_log_org_created ON audit_logs(org_id, created_at DESC);

-- Refresh tokens по family (для rotation detection)
CREATE INDEX idx_refresh_token_family ON refresh_tokens(family_id);
```

## Row Level Security

PostgreSQL RLS как дополнительный слой изоляции:

```sql
ALTER TABLE marketplace_accounts ENABLE ROW LEVEL SECURITY;

CREATE POLICY org_isolation ON marketplace_accounts
    USING (org_id = current_setting('app.current_org_id')::uuid);
```

`app.current_org_id` устанавливается зависимостью `require_org_path_context` из
`org_id` в пути запроса (не из JWT — токен не содержит org-контекста, см.
[Контроль доступа](./access-control.md)).

## Миграции

- Alembic для версионирования схемы.
- Каждая миграция — атомарная, с `upgrade` и `downgrade`.
