# Telegram: реферальная ссылка партнёра и регистрация

Дата: 2026-11-12  
Статус: approved

## Цель

Дать партнёру/админу Telegram deep link по кампании типа `referral_link` и провести полный цикл: переход → регистрация в боте → first-touch attribution → клиентский бонус кампании (где применимо) → вход в Manager Portal. Веб-поток `?ref=` не меняется.

## Решения (зафиксировано)

| Тема | Решение |
|---|---|
| Где видна ссылка | Там же, где веб-`referralUrl`: admin API/UI кампании и `/partners/me` |
| Формат | `https://t.me/{PRODUCT_TELEGRAM_BOT_USERNAME}?start=ref_{PUBLIC_CODE}` |
| Разведение с привязкой TG | Payload с префиксом `ref_` = партнёрка; иначе — текущий link-код Redis |
| Логика атрибуции | Как на вебе: first-touch на создании аккаунта; повторно не привязываем |
| Клиентский бонус | Да: тот же managed promo кампании (`public_code`) |
| Новые таблицы | Нет |
| Partner Portal UI | Вне скоупа |

## Контекст

- Веб: `CampaignService.referral_url` → `{MANAGER_PORTAL}/register?ref={code}`; click + cookie `mh_ref`; `AuthService.register` → `AttributionService.try_attribute_by_code`.
- Клиентский бонус кампании = managed `PromoCode` (`origin=partner`, код = `public_code`).
- На вебе `discount` не redeem’ится сразу: код кладётся в `localStorage` (`mh_pending_promo`) и уходит в checkout на `/billing`. `free_period` на вебе в основном показывается баннером; apply — через redeem/billing.
- Telegram `/start` сегодня: пустой payload → регистрация/вход; любой payload → только account-link. Партнёрский код как start без префикса ломает сценарий (ошибка «код привязки устарел») и не атрибутирует.

## Scope

### In

1. Генерация `telegramReferralUrl` рядом с `referralUrl` (backend schemas + admin/`/partners/me`).
2. Разбор `ref_{code}` в `TelegramService` `/start`.
3. Для нового TG-пользователя: регистрация → запись click → attribution → применение бонуса.
4. Для существующего TG-аккаунта: только login-кнопка (без re-attribute / без повторного бонуса).
5. Admin UI: копирование Telegram-ссылки рядом с веб-ссылкой кампании.
6. Краткие правки `docs/architecture/partners.md` и `docs/architecture/telegram-bot.md`.
7. Тесты разбора payload, attribution/bonus на TG-регистрации, контракт URL.

### Out

- Изменение веб-`?ref=` / cookie.
- Partner Portal UI, payouts, ClickHouse.
- Смена правил комиссии / first-touch.
- Отдельные short-token’ы вместо `public_code`.
- Поддержка `startapp` / Mini App.

## Архитектура потока

```text
Admin/partner копирует telegramReferralUrl
        │
        ▼
User открывает t.me/{bot}?start=ref_{CODE}
        │
        ▼
TelegramService._handle_start
  ├─ payload пустой        → текущий register/login
  ├─ payload = ref_{CODE}  → партнёрский путь
  └─ иначе                 → текущий account-link
        │
        ▼
Партнёрский путь
  1. normalize CODE
  2. resolve campaign + record click (существующий click/resolve слой)
  3a. новый TG user:
        register user+org+TelegramAccount
        → try_attribute_by_code(..., channel_type/source=referral_link)
        → apply client bonus (см. ниже)
        → login button
  3b. существующий TG user:
        login button only
```

Компоненты без дублирования бизнес-логики:

| Ответственность | Где |
|---|---|
| Сборка веб/TG URL | `CampaignService` (+ settings: `manager_portal_url`, `product_telegram_bot_username`) |
| Click / resolve кода | существующий partners click/public resolve |
| Attribution | `AttributionService.try_attribute_by_code` |
| Bonus | `PromoService` / pending-promo bridge (см. ниже) |
| Deep link UX | `TelegramService` |
| Копирование ссылки | admin-panel partners campaign UI |

## Контракт ссылки и API

### Deep link

- Префикс: `ref_` (латиница, нижний регистр).
- Код: `normalize_public_code` → `A-Z0-9_-`, длина как сейчас (3–64).
- Пример: `https://t.me/mhcatdev_bot?start=ref_ABC123`
- Лимит Telegram start payload: 64 символа → `ref_` (4) + code ≤ 60; текущий max code 64 теоретически может не влезть. **Правило:** при сборке URL, если `len("ref_" + public_code) > 64`, поле `telegramReferralUrl` всё равно возвращается со значением `null`, веб-ссылка остаётся; в админке показать, что TG-ссылка недоступна из‑за длины кода. Генератор кодов в этой задаче не меняем.

### API

В ответах кампании (admin + partner me), где уже есть `referralUrl`:

| Поле | Тип | Обязательность | Смысл |
|---|---|---|---|
| `referralUrl` | string \| null | как сейчас | веб-регистрация |
| `telegramReferralUrl` | string \| null | да в JSON, значение nullable | TG deep link; `null` если нет `PRODUCT_TELEGRAM_BOT_USERNAME` или payload > 64 |

`CampaignService.telegram_referral_url(campaign, bot_username) -> str | None`.

### Admin UI

На карточке партнёра / кампании рядом с копированием веб-ссылки — копирование Telegram-ссылки (если не `null`). Без новых разделов меню.

## Клиентский бонус

После **успешной** first-touch attribution на TG-регистрации:

| `client_bonus_type` | Поведение |
|---|---|
| `none` | ничего |
| `free_period` | `PromoService.redeem_promo(user_id, public_code)` сразу (с `plan_name` из `bonus_target_plan`, если нужен API) |
| `discount` | **не** вызывать `redeem_promo` (он отклоняет discount). Положить `public_code` в Redis-payload login-токена (поле вроде `pending_promo_code`). `POST /auth/telegram/exchange` возвращает его клиенту; страница `/login/telegram` вызывает существующий `setPendingPromoCode` — дальше тот же путь, что веб-`mh_pending_promo` на `/billing`. Без новой таблицы и без query в URL |

Сбой бонуса: регистрация и attribution **не** откатываются; ошибка только в лог; пользователю — обычный welcome/login без технических деталей.

Повторный `/start ref_` тем же пользователем бонус не применяет повторно (нет новой attribution; promo `max_uses_per_user=1` / first-touch).

## Ошибки и краевые случаи

| Ситуация | Поведение |
|---|---|
| Невалидный / неизвестный / неактивный код | Новый user: обычная TG-регистрация **без** attribution/bonus + короткое сообщение, что приглашение не сработало. Существующий: обычный login |
| Партнёр не `active` / кампания вне окна | Как `try_attribute_by_code` / resolve сейчас — без bind |
| Self-referral | Тихий skip attribution (как сейчас) |
| Уже есть attribution | First-touch wins, тихо |
| Нет bot username в settings | `telegramReferralUrl = null`; бот-хендлер `ref_` всё равно должен уметь атрибутировать, если webhook жив |
| Коллизия `ref_` с link-кодом | Нет: link-коды без префикса `ref_`; партнёрские только с префиксом |

## Тестирование

1. Парсинг start: пустой / `ref_CODE` / link-code.
2. Новый user + валидный ref → click + attribution; `free_period` → redemption; `discount` → pending code доходит до portal helper.
3. Повторный start → login only, без второй attribution/bonus.
4. Битый ref → регистрация без attribution.
5. `telegramReferralUrl` с username; `null` без username; `null` при слишком длинном code.
6. Admin UI показывает/копирует TG-ссылку.
7. Ручной smoke: deep link → TG register → attributed user в admin; бонус по типу кампании.

## Документация

- `docs/architecture/partners.md` — момент bind: регистрация через Telegram `ref_`.
- `docs/architecture/telegram-bot.md` — таблица start payload + партнёрский путь.

## Не менять

- Схему БД партнёров.
- Правила комиссии и ledger.
- Поведение веб-регистрации и cookie `mh_ref`.
- Account-link flow без префикса `ref_`.
