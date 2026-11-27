# Вводный оффер подписки — контракт для клиента (расширение)

Документ для разработчика `extension-chrome`. Код расширения в этом треке не меняется — клиент подключает UI по этому контракту.

Связанные требования: [billing-intro-offer-requirements](../superpowers/specs/2026-11-27-billing-intro-offer-requirements.md).  
Общий биллинг: [architecture/billing.md](../architecture/billing.md).

## Назначение

Пользователь может купить выбранный платный тариф за символическую сумму на N дней. После периода система списывает полную **monthly**-цену тарифа, если карта сохранена и подписка не отменена.

## Идентификация клиента

Как у каталога тарифов:

```
X-MarketHacker-Client: browser_extension
```

или `?client=browser_extension` на GET планов. Для intro-эндпоинтов заголовок желателен для единообразия.

Auth: `Authorization: Bearer <accessToken>`.

Ответы — camelCase, обёртка `{ "data": …, "meta": {} }`.

## GET /api/v1/billing/intro-offer

Текущее состояние оффера для пользователя.

### Ответ `data`

| Поле | Тип | Описание |
|------|-----|----------|
| `enabled` | boolean | Оффер включён и целевой тариф настроен |
| `eligible` | boolean | Можно оформить сейчас |
| `priceRub` | string (decimal) | Цена вводного платежа, ₽ |
| `periodDays` | number | Длительность вводного периода |
| `planName` | string \| null | slug тарифа (`pro`, …) |
| `planDisplayName` | string \| null | Отображаемое имя |
| `fullPriceMonthly` | string \| null | Полная monthly-цена после периода |
| `allowReturningSubscribers` | boolean | Глобальный win-back |
| `reason` | string \| null | Код причины, если `eligible=false` (для логов/DevTools, не сырой текст пользователю) |

### Коды `reason`

| Код | Смысл для UI |
|-----|----------------|
| `offer_disabled` | Оффер выключен — не показывать CTA |
| `plan_not_configured` | Оффер выключен/не готов — не показывать CTA |
| `already_subscribed` | Уже есть действующий платный доступ — обычный upgrade |
| `returning_not_allowed` | Ранее платил подписку; без права win-back — не показывать intro CTA |

### Пример

```json
{
  "data": {
    "enabled": true,
    "eligible": true,
    "priceRub": "1.00",
    "periodDays": 7,
    "planName": "pro",
    "planDisplayName": "Личный",
    "fullPriceMonthly": "1990.00",
    "allowReturningSubscribers": false,
    "reason": null
  },
  "meta": {}
}
```

## POST /api/v1/billing/subscription/intro

Запуск оплаты вводного оффера (Hosted Checkout Launch).

### Тело

```json
{ "provider": "yookassa" }
```

Только `yookassa`. Yearly нет.

### Успех `data`

Как у `POST /billing/subscription/upgrade`:

| Поле | Тип |
|------|-----|
| `launchUrl` | string — открыть в браузере / вкладке |
| `provider` | `"yookassa"` |
| `intentId` | UUID |
| `expiresAt` | ISO datetime |

Клиент **не** открывает confirmation URL ЮKassa напрямую — только `launchUrl`.

### Ошибки

| HTTP | Когда | Что показать пользователю |
|------|-------|---------------------------|
| 400 | Не eligible / оффер выключен / ошибка провайдера | «Сейчас нельзя оформить пробный период. Обновите страницу или выберите обычный тариф.» |
| 401 | Нет/просрочен токен | Перелогин |
| 403 | Нет доступа | Общий отказ |

## Статус подписки `intro`

После успешной оплаты:

- `GET /billing/subscription` → `status: "intro"`
- Effective plan = целевой тариф оффера до `currentPeriodEnd`
- Автопродление: при сохранённой карте и без отмены после конца периода списывается `fullPriceMonthly`, статус становится `active`

Во время `intro` пользователь может вызвать обычный `POST /billing/subscription/upgrade` на тот же или другой платный тариф — полная оплата, статус → `active`.

### Подпись в UI

Рекомендуемая подпись: «Вводный период».

Для расширения: если есть allowlist «активных» статусов (`active`, `trialing`), **добавьте `intro`**, иначе UI может считать, что платного доступа нет.

## Отмена и карта

- `POST /billing/subscription/cancel` — отписка; полного списания после intro не будет.
- Удаление сохранённой карты — автосписания не будет; доступ кончится с `currentPeriodEnd`.

## Рекомендуемый UX в расширении

1. При открытии тарифов/онбординга: `GET /billing/intro-offer`.
2. Если `eligible` — показать CTA «Попробовать за {priceRub} ₽ на {periodDays} дн.» + пояснение про автосписание полной цены и возможность отменить.
3. По клику: `POST /billing/subscription/intro` → открыть `launchUrl`.
4. После возврата: `GET /billing/subscription` / entitlements; при `status=intro` показать дату конца периода и путь к отмене / полной оплате.

## Что не делать

- Не хардкодить цену 1 ₽ / 7 дней / тариф `pro` — брать из API.
- Не путать с промокодом `trial` (бесплатные дни без автосписания).
- Не менять backend-контракт ради UI без согласования.
