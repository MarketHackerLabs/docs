# Design: вводный оффер подписки (N ₽ / N дней)

Дата: 2026-11-27  
Статус: implemented  
Требования: [2026-11-27-billing-intro-offer-requirements.md](./2026-11-27-billing-intro-offer-requirements.md)

## Цель

Глобально настраиваемый платный вводный период выбранного тарифа: оплата N ₽ → статус `intro` на N дней → автосписание полной monthly-цены тарифа (карта + без отписки). Win-back: глобальный флаг + точечные гранты админа.

## Архитектура

### Конфиг (singleton)

Ключи в `platform_settings` (как welcome subscription):

| Ключ | Тип | Default |
|------|-----|---------|
| `intro_offer_enabled` | bool | `false` |
| `intro_offer_price_rub` | Decimal/number | `1` |
| `intro_offer_period_days` | int | `7` |
| `intro_offer_plan_id` | UUID string | `""` (обязателен при enabled) |
| `intro_offer_allow_returning_subscribers` | bool | `false` |

Один активный оффер = эти ключи. Целевой тариф любой платный (`price_monthly > 0`), не зашит в `pro`.

### Статус подписки

Новый статус: **`intro`**.

| Путь | Поведение |
|------|-----------|
| Entitlement | `intro` ∈ period-bound → доступ к плану до `current_period_end` |
| Renewals due | `intro` включается наряду с `active` / `past_due` |
| Успешный renewal | статус → `active`, период +30 дней (monthly) |
| Expire stale | `intro` в кандидатах как `active`/`trialing`/`past_due` |
| Cancel | как обычно → `cancelled`, без автосписания |
| Admin/MRR | считать «активной» вместе с `active`/`trialing` (доступ есть) |

Не переиспользовать `trialing` (бесплатный промо-trial без автосписания).

### Платёж

`payment_type = "subscription_intro"`.

Активация:

- `status = "intro"`
- `period_days` из `intro_offer_period_days` (не 30)
- `billing_period` в meta = `"monthly"`
- `autopay_enabled` по `yookassa_recurrent_enabled`
- сохранение карты как у `subscription_initial`

После intro renewal (`subscription_renewal`) → обычный `active`.

### Eligibility

Оффер доступен, если:

1. `intro_offer_enabled` и валидный целевой план;
2. у пользователя нет текущей «живой» платной подписки с доступом (`intro` / `active` / `trialing` / `past_due` / `cancelled` в grace) на платный план — иначе только через обычный upgrade (уже платит);
3. одно из:
   - **нет** успешных оплат подписки (`subscription_initial` / `subscription_renewal` / `subscription_intro`);
   - **или** `intro_offer_allow_returning_subscribers`;
   - **или** есть неиспользованный (не revoked, `consumed_at IS NULL`) грант win-back.

После успешной intro-оплаты грант помечается `consumed_at`. Повтор без глобального флага — только новой выдачей админом.

Докупки лимитов в «уже покупал» **не** входят.

### Checkout

- `POST /billing/subscription/intro` → checkout intent с типом intro (monthly only).
- Во время `intro` обычный `POST /billing/subscription/upgrade` на тот же или другой платный план разрешён → `subscription_initial` → `active` (прерывает intro-период полной оплатой).

Публичное чтение оффера:

- `GET /billing/intro-offer` → `{ enabled, eligible, priceRub, periodDays, planName, planDisplayName, reason? }`  
  (и дублирование ключевых полей в ответах subscription/plans по необходимости для portal).

### Win-back grants

Таблица `billing_intro_offer_grants`:

| Поле | Смысл |
|------|--------|
| `id` | UUID PK |
| `user_id` | кому |
| `granted_by_admin_id` | кто выдал |
| `comment` | опционально |
| `consumed_at` | после успешной intro-оплаты |
| `revoked_at` / `revoked_by_admin_id` / `revoke_comment` | отзыв до использования |
| timestamps | created/updated |

Активный грант: `revoked_at IS NULL AND consumed_at IS NULL`. Несколько активных не нужно — при выдаче предыдущие неconsumed revoked или запрет второй активной.

Admin API на карточке пользователя (паттерн MH credits grants).

### Клиенты

| Клиент | Работа |
|--------|--------|
| Admin | settings: блок Intro offer; user: grant/revoke |
| Manager-portal | карточка/CTA оффера на `/billing`, если `eligible` |
| Extension | только API + `docs/integrations/billing-intro-offer-client.md`; код расширения не трогаем |

### Out

- Yearly intro
- Отдельный тариф в каталоге
- Правки `extension-chrome`
- Замена промо `trial`/`discount`

## Риски

- Все фильтры статусов (entitlement, renewals, expire, admin counts, extension entitlements) должны явно знать `intro`.
- Intro payment activation не должна брать 30 дней из monthly default.
- `user_had_paid_subscription` / eligibility должны учитывать `subscription_intro` после первой покупки.
