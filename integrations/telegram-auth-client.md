# Интеграция Telegram-auth в клиентах

Референс-реализация: Manager Portal (`/login/telegram`, `/complete-profile`, настройки → Telegram).

Расширение и другие клиенты **подключают те же API**; UI в `extension-chrome` намеренно не реализован.

Базовый URL: `/api/v1`.

## 1. Вход через бота

1. Пользователь вызывает `/login` (или `/start`) у product-бота.
2. Бот отвечает inline-кнопкой со ссылкой:

```
{MANAGER_PORTAL_URL}/login/telegram?token={oneTimeToken}
```

Для другого клиента замените host/path, но **тот же** `token` передайте в exchange.

3. Клиент вызывает:

### `POST /auth/telegram/exchange`

**Тело:**

| Поле | Тип | Обязательный | Описание |
|---|---|---|---|
| `token` | string | да | One-time токен из ссылки бота |
| `deviceId` | string | нет | Идентификатор устройства (обязателен по смыслу для extension) |
| `mfaCode` | string | нет | TOTP / backup, если MFA уже известен |

**Пример запроса:**

```json
{
  "token": "AbCdEf…",
  "deviceId": "ext-device-uuid"
}
```

**Успех `200`:**

```json
{
  "accessToken": "…",
  "refreshToken": "…",
  "tokenType": "bearer",
  "needsProfileCompletion": true
}
```

**MFA `403`, code `MFA_REQUIRED`:**

```json
{
  "error": {
    "code": "MFA_REQUIRED",
    "details": { "mfaToken": "…" }
  }
}
```

Затем стандартный `POST /auth/mfa/complete` с `mfaToken` + `code` + `deviceId` (см. [auth-mfa-client.md](./auth-mfa-client.md)).

**Ошибки:** ссылка устарела / уже использована → `401`.

## 2. Дозаполнение профиля

Если `needsProfileCompletion === true` (или `GET /auth/telegram/status` вернул флаг):

### `POST /auth/telegram/complete-profile`

**Auth:** Bearer access token.

| Поле | Тип | Обязательный |
|---|---|---|
| `email` | string (email) | да |
| `password` | string (min 8) | да |

**Пример ответа:**

```json
{
  "needsProfileCompletion": false,
  "email": "user@example.com"
}
```

Пока флаг true — не пускайте в основной UI.

## 3. Статус / привязка

### `GET /auth/telegram/status`

```json
{
  "linked": true,
  "telegramUserId": "123456",
  "username": "seller",
  "needsProfileCompletion": false
}
```

### `POST /auth/telegram/link`

Создаёт код и deep-link `https://t.me/{bot}?start={code}`. Пользователь должен открыть бота.

### `DELETE /auth/telegram/link`

Отвязывает Telegram от текущего пользователя.

## 4. Рекомендации для extension

1. После deep-link / custom URL handler получить `token`.
2. `exchange` с `deviceId` из `chrome.storage` / своего store.
3. Сохранить JWT как при email-login.
4. Если `needsProfileCompletion` — экран email+password → `complete-profile`.
5. Привязку существующего аккаунта делайте из экрана настроек через `link` (тот же flow, что в Manager Portal).

Email/password login остаётся независимым каналом.
