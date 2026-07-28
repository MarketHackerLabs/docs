# Подключение клиента к чату поддержки

Базовый URL: `{API}/api/v1`  
OpenAPI: `{API}/docs` (tag `support`)

Авторизация: `Authorization: Bearer <accessToken>` (обычный login).  
Тела — camelCase.

Агенты: `POST /admin/auth/login`, права — `GET /support/staff/me`.

В Manager Portal у `isSupportStaff` клиентский виджет не открывает чат — ведёт в Admin Panel `/support`.

---

## Модель для клиента

Один постоянный чат на пользователя. Extension и Manager Portal пишут в одну переписку.  
`closed` / «Решено» только убирает из очереди агентов; следующее сообщение клиента снова открывает тот же чат.  
`source` — откуда открыли/написали, не отдельная ветка.

Системные события (claim, теги и т.п.) клиенту в ленте не отдаются (ни REST, ни WebSocket `message`).

---

## 1. Открыть чат

```http
POST /api/v1/support/conversations
{ "source": "browser_extension" }
```

`source`: `manager_portal` | `browser_extension` | `telegram` | `api`.

Вернётся существующий чат (в т.ч. закрытый — статус станет активным при необходимости) или создастся новый.

---

## 2. Список

```http
GET /api/v1/support/conversations?limit=50
```

Без прав поддержки — только свои чаты.

Агенты: `status`, `assignee=me|unassigned|{uuid}`, `q`, cursor.

---

## 3. Сообщения

```http
GET /api/v1/support/conversations/{id}/messages?limit=50
POST /api/v1/support/conversations/{id}/messages
{
  "body": "Текст",
  "clientMessageId": "uuid",
  "attachmentIds": []
}
```

`clientMessageId` — идемпотентность. Текст без HTML.

Сотрудник в **чужом** чате пишет как agent (если чат свободен или назначен на него).  
В **своём** чате — всегда как customer.

---

## 4. Картинки

```http
POST /api/v1/support/conversations/{id}/attachments
Content-Type: multipart/form-data
file: <файл>
```

jpeg/png/webp/gif, до 10 МБ → затем сообщение с `attachmentIds`.  
URL: `GET /support/attachments/{id}/url` → для local storage вернёт `requiresAuth: true`  
и абсолютный `url` на `/content`; клиент должен скачать с `Authorization: Bearer`  
(в UI — blob URL). S3 — signed URL без auth.

---

## 5. WebSocket (протокол v1)

```
WS {API}/api/v1/support/ws
```

Авторизация: предпочтительно `Authorization: Bearer <accessToken>` при handshake.  
Допускается query `?access_token=` (менее безопасно — утечки в логах/Referer).  
Ошибка auth → закрытие с кодом `4401` до `accept`.

### Единый envelope

Все кадры (C→S и S→C):

```json
{
  "v": 1,
  "event": "conversation.updated",
  "ts": "2026-07-28T06:00:00.000Z",
  "requestId": null,
  "correlationId": null,
  "payload": {}
}
```

- `v` — обязателен, только `1`. Иначе `event: "error"`.
- `event` — имя события (не `type`).
- Доменные данные только в `payload`.
- `requestId` — клиент может передать на C→S; сервер echo в ответе.

Обратная совместимость со старым `{ "type": "..." }` **не поддерживается**.

### Client → Server

| event | payload |
|-------|---------|
| `ping` | `{}` |
| `subscribe` | `{ "conversationId": "<uuid>" }` |
| `unsubscribe` | `{ "conversationId": "<uuid>" }` |

Отправка сообщений и смена статуса — **только REST**. Ping ~25 с.

### Server → Client (control)

| event | payload |
|-------|---------|
| `pong` | `{}` |
| `subscribed` | `{ "conversationId": "<uuid>" }` |
| `error` | `{ "code": "PERMISSION_DENIED" \| "VALIDATION_ERROR" \| "NOT_FOUND" \| "UNAUTHORIZED", "message": "...", "details": null }` |

Неуспешный `subscribe` → `error`, локальную подписку ставить только после `subscribed`.

### Server → Client (domain)

Только два события:

#### `conversation.updated`

```json
{
  "v": 1,
  "event": "conversation.updated",
  "ts": "...",
  "requestId": null,
  "correlationId": "...",
  "payload": {
    "conversation": { "...ConversationDTO без notes и unreadCount..." },
    "message": null,
    "change": { "kind": "message_created" }
  }
}
```

- `conversation` — всегда.
- `message` — опционально (новое / edit / delete / system_event для staff).
- `change.kind`: `created` | `message_created` | `message_updated` | `assigned` | `unassigned` | `status` | `priority` | `tags` | `closed` | `reopened`.

Клиент: merge `conversation`; если есть `message` и открыт этот чат — upsert в ленту по `id` (учитывать `deletedAt`).

Владелец чата получает события без `subscribe`.  
`system_event` в `message` приходит **только сотрудникам**; клиент получает тот же кадр с `message: null`.

#### `unread.updated`

```json
{
  "v": 1,
  "event": "unread.updated",
  "payload": {
    "total": 2,
    "conversations": [{ "conversationId": "...", "count": 2 }]
  }
}
```

Персонально текущему пользователю.

### После reconnect

1. Переподключиться и снова `subscribe`.
2. Догрузить историю сообщений и unread через REST.

### Пример кадра

```json
{
  "v": 1,
  "event": "ping",
  "ts": "2026-07-28T06:00:00.000Z",
  "requestId": "req-1",
  "correlationId": null,
  "payload": {}
}
```

---

## 6. Статусы

| status | Для клиента |
|--------|-------------|
| `new` | Новый |
| `in_progress` | В работе |
| `waiting_user` | Ждём ваш ответ |
| `closed` | Закрыт |

Закрытие: `POST .../conversations/{id}/close` (история сохраняется).

---

## 7. Непрочитанные

```http
GET /api/v1/support/unread
POST /api/v1/support/conversations/{id}/read
{ "messageId": "<id последнего прочитанного или null (= до последнего в чате)>" }
```

`system_event` в unread не входят.

---

## 8. Для агентов

| Действие | Метод |
|----------|--------|
| Взять | `POST .../claim` |
| Передать | `POST .../assign` `{ "assigneeId" }` |
| Отпустить | `POST .../unassign` |
| Приоритет / теги / waiting_user | `PATCH .../conversations/{id}` |
| Заметки | `GET/POST .../notes` |
| Теги справочник | `GET/POST /tags`, `DELETE /tags/{id}` |
| Сотрудники | `GET/PUT /staff`, `DELETE /staff/{userId}` |

Назначенный чат: писать может только assignee (иначе 403).

---

## 9. Пример

```typescript
const API = "https://api.example.com/api/v1";

async function openChat(token: string) {
  const res = await fetch(`${API}/support/conversations`, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ source: "browser_extension" }),
  });
  return (await res.json()).data;
}

function wsFrame(event: string, payload: Record<string, unknown> = {}) {
  return JSON.stringify({
    v: 1,
    event,
    ts: new Date().toISOString(),
    requestId: null,
    correlationId: null,
    payload,
  });
}
```

---

## Ошибки

```json
{ "error": { "code": "PERMISSION_DENIED", "message": "..." } }
```

`UNAUTHORIZED`, `PERMISSION_DENIED`, `NOT_FOUND`, `CONFLICT`, `VALIDATION_ERROR`, `RATE_LIMITED`.

Лимиты (ориентир): open 30/мин, messages 60/мин, uploads 20/мин.

---

## Telegram

Клиенту бот не нужен. Настройки на сервере: токен, webhook secret, API base URL.

См. [архитектура](../architecture/support-chat.md).
