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
| POST | `/admin/telegram/media` |

Рассылка ставится в ARQ job `send_telegram_broadcast` (timeout 1800s).

### Аудитории

- `all_linked`
- `active_subscription`
- `recently_active` + `audienceParams.days` (default 30)
- `user_ids` + `audienceParams.userIds`
- `telegram_registered` (нет email или password)

### Контент

- `parseMode`: `HTML` | `MarkdownV2`
- текст / caption, photo URL или `photoStorageKey`, inline keyboard

## Модели

- `telegram_accounts` — user ↔ telegram_user_id / chat_id
- `telegram_broadcasts` / `telegram_broadcast_deliveries`
- `telegram_updates`
- `users.email`, `users.password_hash` — nullable
