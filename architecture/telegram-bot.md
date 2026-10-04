# Product Telegram-бот

Отдельный бот для регистрации/входа по Telegram id, привязки к аккаунту MarketHacker и рассылок из Admin Panel. Не связан с support- и notify-ботами.

## Env

| Переменная | Назначение |
|---|---|
| `PRODUCT_TELEGRAM_BOT_TOKEN` | Токен бота (только env) |
| `PRODUCT_TELEGRAM_WEBHOOK_SECRET` | `secret_token` для `setWebhook` и проверки входящих |
| `PRODUCT_TELEGRAM_BOT_USERNAME` | Username без `@` (для deep-link) |
| `PRODUCT_TELEGRAM_MEDIA_PATH` | Локальное хранилище загруженных фото (default `data/telegram-media`). В prod — `/app/data/telegram-media` на общем volume api+worker; после первой успешной отправки файл удаляется, дальше используется `photo_file_id` |
| `API_PUBLIC_BASE_URL` | Публичный HTTPS base API; из него собирается webhook URL |
| `TELEGRAM_API_BASE_URL` | Общий base Bot API (можно прокси для РФ) |
| `MANAGER_PORTAL_URL` | База ссылок входа (`/login/telegram?token=…`) |

## Webhook

При старте API (lifespan), если задан `PRODUCT_TELEGRAM_BOT_TOKEN`, приложение само вызывает Bot API `setWebhook`:

`{API_PUBLIC_BASE_URL}/telegram/webhook`

Пример: `https://api.markethacker.ru/api/v1/telegram/webhook`.

Условия:
- URL должен быть **HTTPS** (иначе регистрация пропускается с логом);
- в production обязателен `PRODUCT_TELEGRAM_WEBHOOK_SECRET`;
- ошибка `setWebhook` не роняет процесс — пишется warning.

Входящие: `POST /api/v1/telegram/webhook`, заголовок `X-Telegram-Bot-Api-Secret-Token`.  
Идемпотентность — таблица `telegram_updates`.

## Потоки бота

| Команда | Поведение |
|---|---|
| `/start CODE` | Привязка к user_id из Redis (код из `POST /auth/telegram/link`) |
| `/start` (новый TG) | Создаёт `User` (email/password = null) + org + `TelegramAccount`, шлёт кнопку входа |
| `/start` (уже связан) | Кнопка входа (one-time token) |
| `/login` | То же: one-time token → Manager Portal |

Токены:

- link: Redis `telegram:link:{code}`, TTL 15 мин
- login: Redis `telegram:login:{token}`, TTL 5 мин

## Auth API

| Метод | Путь | Auth | Назначение |
|---|---|---|---|
| POST | `/auth/telegram/link` | JWT | Deep-link привязки |
| DELETE | `/auth/telegram/link` | JWT | Отвязка |
| GET | `/auth/telegram/status` | JWT | `linked`, `needsProfileCompletion` |
| POST | `/auth/telegram/exchange` | public | One-time token → JWT (+ MFA gate) |
| POST | `/auth/telegram/complete-profile` | JWT | Email + пароль для TG-only аккаунта |

Email/password login не заменяется. MFA при входе через TG соблюдается.

`needsProfileCompletion = email is null OR password_hash is null` — после первого входа клиент обязан показать экран дозаполнения.

Контракт для клиентов: [telegram-auth-client.md](../integrations/telegram-auth-client.md).

## Admin

Право: `telegram:manage`.

| Метод | Путь |
|---|---|
| GET | `/admin/telegram/accounts` |
| POST | `/admin/telegram/messages` |
| POST | `/admin/telegram/broadcasts` |
| GET | `/admin/telegram/broadcasts` |
| GET | `/admin/telegram/broadcasts/{id}` |
| POST | `/admin/telegram/broadcasts/{id}/cancel` |
| POST | `/admin/telegram/media` |

Список рассылок: query `offset`, `limit`, опционально `status` (фильтр по статусу).

Рассылка выполняется ARQ job `send_telegram_broadcast` (timeout 1800s). Стабильный `job_id`: `telegram-broadcast:{broadcast_uuid}`.

### Статусы рассылки

| Статус | Значение |
|---|---|
| `draft` | Черновик (резерв модели; create из админки сразу ставит очередь или план) |
| `scheduled` | Отложена: `scheduledAt` в будущем, deliveries уже созданы |
| `queued` | В очереди на немедленную отправку |
| `sending` | Идёт отправка |
| `done` | Завершена |
| `failed` | Ошибка на уровне рассылки |
| `cancelled` | Запланированная рассылка отменена до старта |

### Создание и отложенная отправка

`POST /admin/telegram/broadcasts` — те же поля контента и аудитории, плюс опциональный **`scheduledAt`** (ISO datetime, timezone-aware; naive трактуется как UTC):

| `scheduledAt` | Поведение |
|---|---|
| не передан или `null` | `status=queued`, job без отложки |
| в будущем | `status=scheduled`, колонка `scheduled_at`, deliveries создаются **сразу** (состав аудитории фиксируется на момент create), `enqueue_job(..., defer_by=scheduledAt−now)` |
| ≤ now | `ValidationError`: время отправки уже прошло |

Локальный файл по `photoStorageKey` на volume **не** удаляется при create scheduled — только после первой успешной доставки, когда получен `photo_file_id` (как для немедленной рассылки).

### Отмена

`POST /admin/telegram/broadcasts/{id}/cancel`:

- только если `status=scheduled`;
- иначе `ValidationError` (в т.ч. если уже `sending`);
- успех → `status=cancelled`.

Отложенный ARQ job при срабатывании вызывает `process_broadcast`: для `cancelled` — no-op (лог, без отправки). Отмена не удаляет job из Redis.

### Жизненный цикл job

```
create (scheduledAt в будущем) → scheduled + deferred job
create (без scheduledAt)       → queued + job сразу
scheduled + due / cron         → process_broadcast → sending → done|failed
cancel из scheduled            → cancelled → job no-op
```

`process_broadcast` принимает вход только при `status ∈ {queued, sending, scheduled}`; при старте переводит в `sending`.

### Cron (safety net)

ARQ cron **`enqueue_due_telegram_broadcasts`** — **каждую минуту** (`unique=True`, timeout 120s):

- выборка `status=scheduled` и `scheduled_at ≤ now()`;
- для каждой — повторный `enqueue_job` с тем же `job_id` `telegram-broadcast:{id}`.

Нужно пережить рестарт Redis/worker без потери просроченных отложенных рассылок; дубликаты отправки сдерживаются стабильным `job_id` и сменой статуса на `sending`/`done`.

### Аудитории

- `all_linked`
- `active_subscription`
- `recently_active` + `audienceParams.days` (default 30)
- `user_ids` + `audienceParams.userIds`
- `telegram_registered` (нет email или password)

### Контент

- `parseMode`: `HTML` | `MarkdownV2`
- текст / caption, photo URL или `photoStorageKey`, inline keyboard

Ответ create/list/get: `scheduledAt`, счётчики доставки, `errorSummary` (без списка deliveries в v1).

## Admin Panel (UI)

Страница `/telegram` (право `telegram:manage`):

- **Композер:** после upload файла превью — `<img>` с `URL.createObjectURL(file)`; при смене или удалении фото — `URL.revokeObjectURL`. Параллельно на API уходит `photoStorageKey`.
- **Отправка:** «Сразу» / «Запланировать» (`datetime-local`, МСК в UI как у новостей); в API — `scheduledAt`. Подсказка: состав аудитории фиксируется при создании, не в момент send.
- **Рассылки:** таблица с фильтром по статусу, pagination; клик по строке — modal (статус, аудитория, текст, кнопки, статистика, даты, `scheduledAt` для запланированных).
- **«Повторить»:** клон в композер (новая рассылка) — заполнение формы и переход на вкладку «Сообщение», **без** автоматической отправки.
- **«Отменить»:** только для `scheduled` → `POST …/broadcasts/{id}/cancel`.
- **Подписчики:** ссылка на `/users/{userId}`.

## Модели

- `telegram_accounts` — user ↔ telegram_user_id / chat_id
- `telegram_broadcasts` (`scheduled_at`, статусы включая `scheduled` / `cancelled`) / `telegram_broadcast_deliveries`
- `telegram_updates`
- `users.email`, `users.password_hash` — nullable
