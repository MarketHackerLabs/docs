# Admin: начисление Мурликов пользователю

Дата: 2026-11-09  
Статус: approved  

## Цель

Дать администратору возможность начислить пользователю Мурлики через админку: увеличить лимит текущего billing-периода, при необходимости отозвать начисление, видеть остаток и историю грантов в карточке пользователя.

## Решения (зафиксировано)

| Тема | Решение |
|---|---|
| Смысл «начислить» | +N к лимиту периода (`effective max_mh_credits_per_period`), ledger `used` не трогаем |
| Срок | Только текущий billing-период пользователя; после смены периода грант сам перестаёт влиять |
| Отзыв | Да: активный грант можно отозвать до конца периода |
| Комментарий | Опционален при выдаче и при отзыве |
| Цель начисления | Effective plan пользователя в карточке; если пользователь — owner org, потолок seat/`shared_pool` растёт автоматически (лимит берётся из плана owner) |
| Подход хранения | Отдельная таблица admin-grant, не докупка и не промокод |
| UI | Блок на странице пользователя в admin-panel; отдельный пункт сайдбара не нужен |

## Контекст модели Мурликов

- Остаток: `remaining = max(0, limit - used)` при конечном лимите; `null` = безлимит.
- `limit` = тариф + промо-бусты + докупки (+ **admin-grants** после этой задачи).
- `used` = `reserved + consumed` в `billing_mh_credits_ledger` за `period_start`.
- Org seat / `shared_pool` используют лимит **owner** effective plan, не личный ledger участника.

## Scope

### In

1. Миграция и ORM-модель таблицы admin-грантов Мурликов.
2. Учёт суммы активных грантов текущего периода в `get_effective_plan` / `limit_adjustments`.
3. Admin API: статус Мурликов пользователя, создание гранта, отзыв гранта + audit-лог.
4. Admin UI: блок «Мурлики» на `users/[id]`, модалка начисления, отзыв с подтверждением.
5. Unit/API-тесты на учёт грантов в effective plan и на grant/revoke.
6. При необходимости — короткое дополнение в `docs/architecture/billing.md`.

### Out

- Корректировка ledger `used` / «вернуть потраченные» Мурлики.
- Гранты с произвольным сроком или бессрочные.
- Явный выбор пула (личный vs org) в UI.
- Seed пакета докупки Мурликов и правки багов формы тарифов/промо (отдельные задачи).
- Изменение пользовательского `GET /billing/mh-credits` (уже читает effective plan).
- Отдельный экран/пункт меню только для Мурликов.

## Данные

Таблица (имя в стиле модуля billing, например `billing_mh_credits_admin_grants`):

| Поле | Тип | Обязательность | Смысл |
|---|---|---|---|
| `id` | UUID | да | PK |
| `user_id` | UUID → users | да | Кому начислено |
| `units` | int (> 0) | да | Сколько Мурликов к лимиту |
| `period_start` | datetime | да | Начало billing-периода на момент выдачи |
| `period_end` | datetime | да | Конец billing-периода на момент выдачи |
| `comment` | text / null | нет | Комментарий выдачи |
| `granted_by_admin_id` | UUID | да | Кто выдал |
| `revoked_at` | datetime / null | нет | Когда отозвано; `null` = активен |
| `revoked_by_admin_id` | UUID / null | нет | Кто отозвал |
| `revoke_comment` | text / null | нет | Комментарий отзыва |
| `created_at` | datetime | да | Когда выдали |

**Активный грант:** `revoked_at is null` и «сейчас» ∈ `[period_start, period_end)` (сравнение в той же семантике, что у подписки/addons).

**Вклад в лимит:**

```text
effective_limit = plan + promo_boosts + purchased_addons + active_admin_grants
```

После grant/revoke — инвалидация кэша effective plan пользователя (как при смене подписки / докупке).

**Отзыв и used > limit:** уже committed usage не откатываем; новые ИИ-действия режутся проверками reserve (как при снижении тарифа).

## API

Все эндпоинты — admin auth, префикс как у существующих `/admin/users/{user_id}/…`.

### `GET /admin/users/{user_id}/mh-credits`

Назначение: остаток и гранты текущего периода для карточки пользователя.

Ответ (смысл полей):

- `limit`, `used`, `remaining`, `periodStart`, `periodEnd`
- `adminGrantUnits` — сумма активных admin-грантов, входящих в `limit`
- `grants[]` — id, units, comment, createdAt, grantedByAdminId, revokedAt, revokedByAdminId, revokeComment, periodStart, periodEnd

### `POST /admin/users/{user_id}/mh-credits/grants`

Тело:

- `units: number` — целое > 0, обязательный
- `comment?: string` — опциональный

Поведение: создать грант с `period_start`/`period_end` текущего billing-периода пользователя; audit `admin.mh_credits.granted`; инвалидация кэша плана.

Ошибки:

- пользователь не найден;
- нет активного расчётного периода / подписки для привязки окна;
- безлимитный тариф — начисление отклоняется («лимит не ограничен»);
- невалидные `units`.

### `POST /admin/users/{user_id}/mh-credits/grants/{grant_id}/revoke`

Тело:

- `comment?: string` — опциональный

Поведение: отозвать активный грант этого пользователя; audit `admin.mh_credits.revoked`; инвалидация кэша.

Ошибки:

- грант не найден, чужой user, или уже отозван.

Пользовательский `GET /billing/mh-credits` не меняем.

## UI (admin-panel)

Страница: `admin-panel/src/app/(admin)/users/[id]/page.tsx`.

Блок «Мурлики»:

- период, лимит, использовано, остаток;
- вклад admin-грантов в лимит;
- кнопка «Начислить» (скрыта при безлимите);
- таблица грантов текущего периода со статусом и «Отозвать» для активных.

Модалка начисления: `units` + optional comment → POST → toast → reload.  
Отзыв: ConfirmDialog (+ optional comment) → POST revoke → toast → reload.

Паттерны: `useApi` / `useMutation` / `Modal` / `ConfirmDialog` / `useToast`, по аналогии с `GrantSubscriptionModal`.

Тексты для админа — продуктовые («Начислено», «Отозвано», «Не удалось …»), без внутренних имён сервисов и SQL.

## Аудит

| Action | Когда |
|---|---|
| `admin.mh_credits.granted` | успешная выдача |
| `admin.mh_credits.revoked` | успешный отзыв |

В payload: user_id, grant_id, units, admin_id (и comment при наличии — без секретов).

## Тесты

1. Unit: активные гранты текущего периода увеличивают `max_mh_credits_per_period` в effective plan; отозванные и гранты другого периода — нет.
2. API: grant → `remaining` растёт на `units`; revoke → падает; повторный revoke — ошибка; кэш плана инвалидируется.
3. Смоук в админке: карточка пользователя, начисление и отзыв.

## Риски и краевые случаи

| Случай | Поведение |
|---|---|
| Owner org получил грант | Потолок seat/`shared_pool` его org растёт (limit из owner plan) |
| Member (не owner) получил грант | Растёт только личный пул participant |
| Отзыв при used > нового limit | Новые действия блокируются; история списаний цела |
| Смена billing-периода | Старые гранты вне окна не суммируются |
| Безлимит | UI без кнопки начисления; API отклоняет grant |

## Зависимости от существующих мест

- `BillingService.get_effective_plan` + `limit_adjustments.py`
- `MhCreditsService` (period / status) — для чтения статуса в admin GET
- `admin/api/router.py`, `admin/application/service.py` (audit по образцу subscription grant)
- `admin-panel` user detail + `lib/api.ts`
