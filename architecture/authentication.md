# Аутентификация и авторизация

## Особенности browser extension

Chromium-расширение (MV3) не может использовать httpOnly-cookie так же, как веб-приложение. Поэтому:

- **Token-based auth** — access + refresh JWT.
- Токены хранятся в `chrome.storage.session` (очищается при закрытии браузера) или `chrome.storage.local`.
- CORS настроен на `chrome-extension://<extension_id>`.

## Поток аутентификации

```mermaid
sequenceDiagram
    participant EXT as Extension
    participant API as Backend API
    participant AUTH as Auth Service
    participant PG as PostgreSQL
    participant REDIS as Redis

    EXT->>API: POST /auth/login (email, password)
    API->>AUTH: verify credentials
    AUTH->>PG: check user + MFA
    AUTH->>REDIS: store refresh token family
    AUTH-->>EXT: access_token (15m) + refresh_token (30d)

    Note over EXT: Токены в chrome.storage.session

    EXT->>API: GET /analytics (Authorization: Bearer ...)
    API->>AUTH: validate JWT + permissions
    API-->>EXT: data

    Note over EXT: access_token истёк

    EXT->>API: POST /auth/refresh (refresh_token)
    API->>AUTH: validate + rotate
    AUTH->>REDIS: invalidate old, store new
    AUTH-->>EXT: new access_token + new refresh_token
```

## Токены

| Тип | TTL | Хранение (extension) |
|-----|-----|------------------------|
| Access JWT | 15 мин | `chrome.storage.session` |
| Refresh token | 30 дней | `chrome.storage.session` + привязка к `device_id` |
| Device fingerprint | — | Extension install ID |

### JWT payload (access token)

```json
{
  "sub": "user_uuid",
  "org_id": "current_org_uuid",
  "permissions": ["analytics:read", "ads:write"],
  "iat": 1710000000,
  "exp": 1710000900,
  "jti": "unique_token_id"
}
```

Минимальный payload — permissions резолвятся при выдаче токена и кэшируются до истечения TTL.

### Refresh token

- Хранится в БД как **hash** (не plaintext).
- Привязан к `family_id` — все токены одной сессии/устройства.
- Привязан к `device_id` — ID устройства из extension.

## Refresh token rotation

При каждом `POST /auth/refresh`:

1. Старый refresh token **инвалидируется**.
2. Выдаётся новая пара access + refresh.
3. `family_id` сохраняется — все токены семейства связаны.

### Token family detection

Если использован уже отозванный refresh token (reuse):

1. Инвалидируется **всё семейство** токенов.
2. Пользователь разлогинивается на всех устройствах этой сессии.
3. Событие записывается в audit log.
4. Опционально — уведомление пользователю (подозрение на компрометацию).

## Эндпоинты auth

| Метод | Путь | Описание |
|-------|------|----------|
| POST | `/api/v1/auth/register` | Регистрация + создание org |
| POST | `/api/v1/auth/login` | Вход, выдача токенов |
| POST | `/api/v1/auth/refresh` | Обновление access token |
| POST | `/api/v1/auth/logout` | Отзыв refresh token |
| POST | `/api/v1/auth/switch-org` | Смена активной организации |
| POST | `/api/v1/auth/mfa/setup` | Настройка TOTP |
| POST | `/api/v1/auth/mfa/verify` | Подтверждение MFA при login |

## MFA (TOTP)

- Обязательна для ролей `owner` и `admin` (настраивается политикой).
- Опциональна для остальных.
- Реализация: стандартный TOTP (Google Authenticator, Authy).
- Backup codes — 10 одноразовых кодов при настройке MFA.

## Регистрация

При `POST /auth/register`:

1. Создаётся `User`.
2. Создаётся `Organization` (имя = email или заданное).
3. Создаётся `Membership` с ролью `owner`.
4. Выдаётся пара токенов.

## OAuth2 PKCE (будущее)

Для web-dashboard и SSO:

```mermaid
sequenceDiagram
    participant EXT as Extension / Web
    participant API as Backend
    participant BROWSER as Browser Popup

    EXT->>BROWSER: Open /oauth/authorize (code_challenge)
    BROWSER->>API: User logs in
    API-->>BROWSER: redirect with authorization_code
    BROWSER-->>EXT: callback with code
    EXT->>API: POST /oauth/token (code + code_verifier)
    API-->>EXT: access_token + refresh_token
```

На MVP достаточно email/password + refresh. PKCE добавляется при появлении web-клиента.

## CORS

```python
ALLOWED_ORIGINS = [
    "chrome-extension://<extension_id>",
    # будущий web dashboard:
    # "https://app.markethacker.ru",
]
```

## Rate limiting

| Эндпоинт | Лимит |
|----------|-------|
| `POST /auth/login` | 5 req/min per IP + per email |
| `POST /auth/register` | 3 req/min per IP |
| `POST /auth/refresh` | 10 req/min per token |
| `POST /auth/mfa/verify` | 5 req/min per user |

Реализация: Redis sliding window.
