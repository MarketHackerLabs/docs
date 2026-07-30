# Support Chat

> Статус: Implemented (2026-07-20). Модуль `modules/support`.

Единый чат поддержки для Manager Portal, Admin Panel, Browser Extension и Telegram.
Один REST/WS API (`/api/v1/support/*`), без отдельного микросервиса.

---

## Модель

- **Один чат на пользователя** (`customer_user_id`, unique). Закрытие (`closed`) только убирает из активной очереди; следующее сообщение снова открывает тот же тред.
- **Один assignee** на чат. Если назначен — отвечать может только он. Свободный чат: первый ответ агента берёт чат себе.
- **Сотрудник = клиент**: в своём чате (`customer_user_id == me`) пишет как `customer`, даже с grant поддержки.
- Текст + картинки (jpeg/png/webp/gif ≤ 10 МБ). Без HTML.
- Внутренние заметки и лог действий (`system_event`) клиенту не отдаются.
- Автоответ вне часов: `sender_type=system`, `content_type=text` — виден клиенту; не путать с audit `system_event`.
- Теги — справочник в БД, CRUD любым сотрудником поддержки.

Клиенты: Manager Portal (виджет), Admin Panel (инбокс), Extension (по [интеграции](../integrations/support-client.md)), Telegram (webhook).

---

## Стек

```
clients → FastAPI modules/support (REST + WS + Telegram webhook)
        → PostgreSQL, Redis pub/sub, S3/local attachments
```

Код: `SupportService` + `SupportAccess` + `SupportRepository` + `SupportRealtimeBroker`.

Миграции: `20260720_0030` (схема), `0031` (без org), `0032` (один чат на customer), `0033` (индекс тегов для фильтра), `0039` (объявления, `support_settings`, автоответы).

---

## Статусы

| status | Смысл |
|--------|--------|
| `new` | Новый / без активного агента |
| `in_progress` | В работе |
| `waiting_user` | Ждём клиента |
| `closed` | Решён (вне активной очереди) |

Переходы: claim / первый ответ → `in_progress`; сообщение клиента из `waiting_user`/`closed` → `in_progress`; close → `closed`.

---

## Назначение

| Действие | Endpoint | Право |
|----------|----------|--------|
| Взять | `POST .../claim` | `assign` |
| Передать / забрать себе | `POST .../assign` `{ assigneeId }` | `assign` |
| Отпустить / открепить | `POST .../unassign` | `assign` |

Optimistic claim: `UPDATE … WHERE assignee_id IS NULL`.  
Забрать у другого агента — `POST …/claim` (force, если уже назначен) или `assign` на себя.  
Открепить чужое назначение — `unassign` при праве `assign` (или superuser).  
Суперпользователей нельзя добавить в staff grants.

---

## Права

Каталог: `support:conversations:read|read_assigned|read_all|reply|close|assign`, `attachments:read`, `settings:manage`, `staff:manage`.

Гранты: `support_staff_grants`. Суперпользователь — полный набор. Вход в админку: superuser **или** активный grant.

Вход агента: `POST /admin/auth/login`. Отказ в `require_superuser` — **403** (не 401), иначе клиент сбрасывает сессию.

---

## Realtime

WS: `/api/v1/support/ws` (Bearer предпочтительно; query `access_token` — fallback).

**Публичный протокол v1** — единый envelope `{ v, event, ts, requestId, correlationId, payload }`.  
Доменные события на wire:

- `conversation.updated` — единственный sync треда (`conversation` + опционально `message` + `change.kind`)
- `unread.updated` — персональные счётчики

Control: `ping`/`pong`, `subscribe`/`unsubscribe`/`subscribed`, `error`.

Внутренние audit (`system_event` в БД) **не** являются отдельными WS `event`; они могут приезжать staff-only внутри `payload.message`. Клиенту `message` с `system_event` не отдаётся.

Маппинг: `PublicWsEventMapper` (`modules/support/infrastructure/ws_protocol.py`) → Redis `support:events` с `_routing` → fanout в `api/ws.py` (strip `_routing`).

Правила добавления событий:

1. Не публиковать сырые dict из сервиса — только через mapper.
2. Новое доменное изменение → расширить `change.kind` или additive поля в payload при том же `v`.
3. Rename/remove/смена семантики → bump `v`.
4. Внутренние DomainEvent не должны автоматически становиться WS event.

Звук уведомления — Web Audio на клиенте (админка / виджет).

---

## Admin UI

- `/support` — инбокс, ответ, теги, приоритет, заметки, лог, claim/transfer/reclaim/unassign
- `/support/staff` — гранты (модалка), удаление
- `/support/announcements` — CRUD объявлений; одно закреплённое (`is_pinned` + `is_active`) в шапке виджета Portal
- `/support/settings` — вкладки Telegram | Режим работы (`SupportSettingsNav`)
  - Telegram: флаги в `platform_settings` (токен в env)
  - Режим работы: часы в `support_settings.business_hours` + шаблон автоответа `off_hours`
  (в UI один переключатель «Автоответ в нерабочее время» синхронизирует `businessHours.enabled` и `autoReply.isActive`)

Фильтр «Мои чаты» → `assignee=me`.  
Фильтр по тегам → `tag_id=<uuid>` (повтор параметра для нескольких; OR — чат с любым из выбранных). Совместим с `status` / `assignee` / `q` / `priority` / `source`.

В инбоксе: опция «Поднимать чаты с новыми сообщениями» (localStorage, по умолчанию вкл.) —
при `message_created` чат уходит в начало списка.

Индекс: `ix_support_conv_tags_tag_id` на `support_conversation_tags(tag_id)`.

Очистка клиентских чатов у staff (одноразово):  
`uv run python scripts/purge_support_staff_customer_chats.py --dry-run`  
затем без `--dry-run`.

Снять grant у суперпользователей:  
`uv run python scripts/purge_superuser_support_grants.py --dry-run`

Снять assignee у бывших сотрудников:  
`uv run python scripts/clear_orphaned_support_assignees.py --dry-run`

Суперпользователей в `support_staff_grants` добавлять нельзя — доступ у них через `is_superuser`.

---

## Объявления, часы работы, автоответчик

**Объявления** (`support_announcements`): CRUD при `support:settings:manage`.  
`GET /support/announcements/pinned` — любой auth; только активное закреплённое. Показ — шапка виджета Manager Portal.

**Часы работы** — singleton `support_settings` (id=1), поле `business_hours` JSONB.  
Не хранятся в `platform_settings`. Чтение/запись через `GET/PATCH /support/settings` (`businessHours`).

```json
{
  "enabled": false,
  "timezone": "Europe/Moscow",
  "schedule": {
    "mon": [{"start": "09:00", "end": "18:00"}],
    "sat": [],
    "sun": []
  }
}
```

Если `enabled=false` — считаем всегда открыто. Пустой список интервалов на день = выходной.

**Автоответчик** (`support_auto_replies`): шаблоны по `trigger_key`. Сейчас `off_hours`.  
Провайдеры: `OffHoursAutoReplyProvider` → `CompositeAutoReplyProvider` (точка расширения под ИИ).  
Волна: `conversation.meta.autoReply.off_hours.suppressUntilAgent` — один ответ до ответа агента.  
Хук после входящего от `customer`/`channel` (REST и Telegram).

API: `GET/PATCH /support/auto-replies/{trigger_key}`.

---

## Manager Portal

Плавающий виджет справа внизу. `/support` редиректит на dashboard.

- **Клиент** (без grant поддержки): чат + WS (бейдж непрочитанных + звук); закреплённое объявление в шапке; статус «Работаем / Не работаем» и интервал на сегодня (`GET /support/hours/status`), если расписание включено.
- **Сотрудник поддержки** (`isSupportStaff`): клиентский чат не открывается — виджет
  ведёт в Admin Panel (`NEXT_PUBLIC_ADMIN_URL` + `/support`), чтобы не смешивать роли
  агента и клиента в одном аккаунте.

---

## Telegram

Env: `SUPPORT_TELEGRAM_BOT_TOKEN`, `SUPPORT_TELEGRAM_WEBHOOK_SECRET`,  
`TELEGRAM_API_BASE_URL` / `TELEGRAM_FILE_BASE_URL` (общий прокси Bot API).  
Webhook: `POST /support/channels/telegram/webhook`.

Ops-уведомления (отдельный бот, супергруппа с топиками):  
`NOTIFY_TELEGRAM_BOT_TOKEN`, `NOTIFY_TELEGRAM_ENABLED`, `NOTIFY_TELEGRAM_CHAT_ID`,  
`NOTIFY_TELEGRAM_PAYMENTS_TOPIC_ID`, `NOTIFY_TELEGRAM_SUPPORT_TOPIC_ID`.  
Триггеры: успешная активация оплаты (без `admin_test`); входящие `customer` / `channel`.

---

## Хранилище вложений

S3-совместимое (`SUPPORT_S3_*`) или локальный путь (`SUPPORT_LOCAL_STORAGE_PATH`).  
В production без S3: named volume `markethacker_support_attachments` → `/app/data/support-attachments` (`docker-compose.prod.yml`).

---

## Индексы (основные)

- `support_conversations (customer_user_id)` unique  
- `support_conversations (assignee_id, status, last_message_at)`  
- `support_messages (conversation_id, created_at, id)`
- `support_conversation_tags (tag_id)` — фильтр инбокса по тегам

---

## Связанные документы

- [Клиентская интеграция](../integrations/support-client.md)
- OpenAPI: `/docs` tag `support`
