# Подключение клиента к чату поддержки

Базовый URL: `{API}/api/v1`  
OpenAPI: `{API}/docs` (tag `support`)

Авторизация: `Authorization: Bearer <accessToken>` (обычный login).  
Тела — camelCase.

Агенты: `POST /admin/auth/login`, права — `GET /support/staff/me`.

---

## Модель для клиента

Один постоянный чат на пользователя. Extension и Manager Portal пишут в одну переписку.  
`closed` / «Решено» только убирает из очереди агентов; следующее сообщение клиента снова открывает тот же чат.  
`source` — откуда открыли/написали, не отдельная ветка.

Системные события (claim, теги и т.п.) клиенту в ленте не отдаются.

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
URL: `GET /support/attachments/{id}/url`.

---

## 5. WebSocket

```
WS {API}/api/v1/support/ws?access_token=<token>
```

```json
{ "type": "subscribe", "conversationId": "..." }
{ "type": "ping" }
```

События: `message.created`, `message.updated`, `conversation.updated`, `unread.updated`.  
Отправка только через REST. Ping ~25 с. После reconnect — subscribe + догрузка истории.

Клиенту `message.created` приходит без subscribe, если он владелец чата (`customerUserId`).  
`unread.updated` — персонально по `userId`.

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
{ "messageId": "<id последнего прочитанного или null>" }
```

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
