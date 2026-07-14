# Продуктовые промо (баннеры / CTA)

Универсальный механизм управляемых объявлений в клиентах платформы
(Manager Portal сейчас; расширение — через те же placements в будущем).

Не путать с **промокодами биллинга** (`/billing/promo/*`) и с remote config
расширения (`/extension/config/{namespace}` — DOM-селекторы).

## Модель

Таблицы: `product_promotions`, `product_promotion_dismissals`,
`product_promotion_assignments` (миграции `20260714_0027`, `20260714_0028`).

| Поле | Назначение |
|------|------------|
| `key` | Стабильный идентификатор кампании (`install_browser_extension`) |
| `placement` | Слот показа: `{client}.{slot}` |
| `requires_features` | Баннер только если у пользователя есть эти billing-фичи (личный ∪ org seat) |
| `requires_org_features` | Дополнительно — фичи тарифа владельца текущей org (`orgId` в запросе) |
| `audience` | `all` / `owner` / `member` |
| `experiment_key` + `variant` | Sticky A/B раздача вариантов (**без сбора метрик**) |
| `dismissible` / `dismiss_ttl_days` | Скрытие крестиком; `NULL` TTL = навсегда |

### Placement — client-scoped

Формат: `{client}.{slot}`. Сейчас:

| Placement | Где показывается |
|-----------|------------------|
| `manager_portal.all` | Любая страница Manager Portal (layout) |
| `manager_portal.dashboard` | Только Dashboard |
| `manager_portal.team` | Только «Команда» |

Дальше без смены схемы можно добавить, например,
`browser_extension.popup` / `browser_extension.overlay` — потребитель
запрашивает свой placement. Для overlay на чужих сайтах правила host/URL
кладутся в `extra` (читает клиент расширения), а не в отдельную
licensing-систему.

## API

### Клиент (Manager Portal / будущие клиенты)

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/promotions/active?placement=…&orgId=` | ✓ | Активные баннеры для пользователя |
| POST | `/api/v1/promotions/{key}/dismiss` | ✓ | Скрыть баннер |

### Admin

| Метод | Путь | Описание |
|-------|------|----------|
| GET/POST | `/api/v1/admin/promotions` | Список / создание |
| GET/PATCH/DELETE | `/api/v1/admin/promotions/{id}` | Детали / обновление / удаление |

UI: **Продвижение → Баннеры** (`/promotions/banners`).

## Targeting

Порядок фильтров при `GET /promotions/active`:

1. `placement` + `is_active` + даты `starts_at` / `ends_at`
2. `audience` (для `owner`/`member` нужен `orgId`)
3. `requires_features` ⊆ эффективные фичи пользователя
4. `requires_org_features` ⊆ фичи плана владельца org
5. Не истёкший dismiss
6. A/B: при общем `experiment_key` остаётся один sticky `variant`

Эффективные фичи — `BillingService.resolve_user_feature_keys` (см.
[Контроль доступа](./access-control.md), [Биллинг](./billing.md)).

## A/B

Поля `experiment_key` / `variant` только закрепляют вариант за пользователем.
**Показы, клики и конверсии не пишутся.** Полноценный эксперимент появится
после событий analytics — до тех пор блок в админке вспомогательный.

## Seed

Кампания `install_browser_extension` на `manager_portal.all` с
`requires_features: ["browser_extension"]` — CTA установки расширения
тем, у кого уже есть доступ (лично или через seat организации).

## Клиент Manager Portal

- `PromotionSlot` в layout — `manager_portal.all`
- Слоты на страницах Dashboard / Team — page-specific placements
- Dismiss — крестик; локальный cache + `POST …/dismiss`
- Опциональный ping `chrome.runtime.sendMessage({ type: "mh.ping" })`
  для авто-скрытия install-баннера (контракт для автора расширения)

## Связанные документы

| Документ | Связь |
|----------|--------|
| [Дизайн API](./api-design.md) | Сводка эндпоинтов |
| [Модель данных](./data-model.md) | Таблицы |
| [Аутентификация](./authentication.md) | Контракт extension + entitlements |
| [Биллинг](./billing.md) | Промокоды (скидки) — другая подсистема |
