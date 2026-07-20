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
- Внутренние заметки и лог действий (system_event) клиенту не отдаются.
- Теги — справочник в БД, CRUD любым сотрудником поддержки.

Клиенты: Manager Portal (виджет), Admin Panel (инбокс), Extension (по [интеграции](../integrations/support-client.md)), Telegram (webhook).

---

## Стек

```
clients → FastAPI modules/support (REST + WS + Telegram webhook)
        → PostgreSQL, Redis pub/sub, S3/local attachments
```

Код: `SupportService` + `SupportAccess` + `SupportRepository` + `SupportRealtimeBroker`.

Миграции: `20260720_0030` (схема), `0031` (без org), `0032` (один чат на customer).

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
| Передать | `POST .../assign` `{ assigneeId }` | `assign` |
| Отпустить | `POST .../unassign` | `assign` |

Optimistic claim: `UPDATE … WHERE assignee_id IS NULL`.

---

## Права

Каталог: `support:conversations:read|read_assigned|read_all|reply|close|assign`, `attachments:read`, `settings:manage`, `staff:manage`.

Гранты: `support_staff_grants`. Суперпользователь — полный набор. Вход в админку: superuser **или** активный grant.

Вход агента: `POST /admin/auth/login`. Отказ в `require_superuser` — **403** (не 401), иначе клиент сбрасывает сессию.

---

## Realtime

WS: `/api/v1/support/ws?access_token=…`  
События: `message.created|updated`, `conversation.updated|assigned|created`, `unread.updated`.  
Системные события пишутся в сообщения (`content_type=system_event`) с `actor` (id/email/name).  
Звук уведомления — Web Audio на клиенте (админка / виджет).

---

## Admin UI

- `/support` — инбокс, ответ, теги, приоритет, заметки, лог, claim/transfer
- `/support/staff` — гранты (модалка), удаление
- `/support/settings` — Telegram (токен в env)

Фильтр «Мои чаты» → `assignee=me`.

---

## Manager Portal

Плавающий виджет справа внизу. `/support` редиректит на dashboard.  
WS всегда на (бейдж непрочитанных + звук).

---

## Telegram

Env: `SUPPORT_TELEGRAM_BOT_TOKEN`, `SUPPORT_TELEGRAM_WEBHOOK_SECRET`,  
`SUPPORT_TELEGRAM_API_BASE_URL` / `SUPPORT_TELEGRAM_FILE_BASE_URL` (прокси).  
Webhook: `POST /support/channels/telegram/webhook`.

---

## Хранилище вложений

S3-совместимое (`SUPPORT_S3_*`) или локальный путь (`SUPPORT_LOCAL_STORAGE_PATH`).

---

## Индексы (основные)

- `support_conversations (customer_user_id)` unique  
- `support_conversations (assignee_id, status, last_message_at)`  
- `support_messages (conversation_id, created_at, id)`

---

## Связанные документы

- [Клиентская интеграция](../integrations/support-client.md)
- OpenAPI: `/docs` tag `support`
