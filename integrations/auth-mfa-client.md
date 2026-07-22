# Клиентская интеграция: Auth + MFA

Контракт для **extension-chrome**, Manager Portal и Admin Panel. Backend уже реализован; клиенты должны обработать MFA challenge без упрощений.

Связанные документы: [Аутентификация](../architecture/authentication.md), [API design](../architecture/api-design.md).

---

## Общие правила

- Все REST JSON — **camelCase**.
- Ошибки — объект `error: { code, message, details }` (см. стандартный error envelope).
- Access token передаётся как `Authorization: Bearer <accessToken>`.
- `deviceId` рекомендуется передавать на login / mfa complete / refresh (привязка refresh family).

---

## Пользовательский вход (`/api/v1/auth/*`)

Используют: **extension**, **manager-portal**.

### 1. Login без MFA

`POST /api/v1/auth/login`

```json
{
  "email": "user@example.com",
  "password": "…",
  "deviceId": "optional-device-id"
}
```

**200:**

```json
{
  "accessToken": "…",
  "refreshToken": "…",
  "tokenType": "bearer"
}
```

### 2. Login при включённом MFA

Если у пользователя `mfaEnabled === true` и код не передан:

**403** с кодом `MFA_REQUIRED` (токены **не** выдаются):

```json
{
  "error": {
    "code": "MFA_REQUIRED",
    "message": "Multi-factor authentication required",
    "details": {
      "mfaToken": "<короткоживущий JWT challenge, ~5 мин>"
    }
  }
}
```

Клиент обязан:

1. Показать UI ввода 6-значного TOTP.
2. Сохранить `details.mfaToken` в памяти (не в long-lived storage).
3. Вызвать complete (ниже). Не трактовать 403 MFA_REQUIRED как «неверный пароль» и не разлогинивать.

Опционально можно сразу передать код в login:

```json
{
  "email": "user@example.com",
  "password": "…",
  "deviceId": "…",
  "mfaCode": "123456"
}
```

При неверном коде — **401** `UNAUTHORIZED` (`Invalid MFA code`).

### 3. Завершение MFA

`POST /api/v1/auth/mfa/complete`  
Rate limit: **5/min**. Без Bearer.

```json
{
  "mfaToken": "<из details.mfaToken>",
  "code": "123456",
  "deviceId": "optional-device-id"
}
```

**200** — та же `TokenResponse`, что у login.

| Ошибка | Когда |
|--------|--------|
| 401 | Неверный/просроченный `mfaToken` или неверный код |
| 400 | MFA не включена / валидация полей |

### 4. Настройка MFA (уже залогинен)

| Метод | Путь | Auth | Назначение |
|-------|------|------|------------|
| POST | `/api/v1/auth/mfa/setup` | Bearer | Получить `secret` + `provisioningUri` (otpauth) |
| POST | `/api/v1/auth/mfa/verify` | Bearer | Подтвердить код и **включить** MFA |

`verify` после setup **не** выдаёт токены — только включает MFA для аккаунта. Вход с MFA идёт через login → complete.

### 5. Logout (усиление)

`POST /api/v1/auth/logout`

```json
{
  "refreshToken": "…",
  "accessToken": "…"
}
```

`accessToken` опционален, но **рекомендуется**: сервер отзовёт `jti` access JWT до `exp`. Без него access живёт до TTL (~15 мин).

`logout_all` (если используете) инвалидирует все access-токены пользователя через Redis `revoked_before`.

---

## Admin вход (`/api/v1/admin/auth/*`)

Использует: **admin-panel**.

Поток идентичен user auth:

1. `POST /api/v1/admin/auth/login` — body: `email`, `password`, `deviceId?`, `mfaCode?`
2. При MFA → **403** `MFA_REQUIRED` + `details.mfaToken`
3. `POST /api/v1/admin/auth/mfa/complete` — `mfaToken`, `code`, `deviceId?`
4. Успех → `accessToken` + `refreshToken` (у суперадмина в JWT будет `is_superadmin`)

Support staff может войти через admin login, но **не** получит `is_superadmin` и не пройдёт `require_superuser` на `/admin/*` data API — только `/support/*`.

---

## UX-чеклист для клиентов

```
[ ] Обработать error.code === "MFA_REQUIRED" отдельно от прочих 403
[ ] Не сохранять mfaToken в localStorage / chrome.storage.local
[ ] Поле ввода кода: inputMode=numeric, autocomplete=one-time-code, 6 цифр
[ ] После успешного complete — обычное сохранение access/refresh
[ ] При истечении mfaToken (~5 мин) — вернуть пользователя на шаг пароля
[ ] Неверный MFA-код: показать ошибку, оставить на шаге кода (тот же mfaToken, пока жив)
[ ] Logout передаёт и refreshToken, и accessToken
```

### Рекомендуемый state machine

```
Idle → CredentialsSubmit
         ├─ 200 tokens → Authenticated
         └─ 403 MFA_REQUIRED → MfaPrompt
                                 ├─ complete 200 → Authenticated
                                 ├─ 401 bad code → MfaPrompt (retry)
                                 └─ 401 expired token → CredentialsSubmit
```

---

## Пример (TypeScript)

```ts
type TokenResponse = {
  accessToken: string;
  refreshToken: string;
  tokenType?: string;
};

type ApiErrorBody = {
  error: {
    code: string;
    message: string;
    details?: { mfaToken?: string };
  };
};

async function loginWithMfa(params: {
  email: string;
  password: string;
  deviceId?: string;
  askMfaCode: () => Promise<string>;
}): Promise<TokenResponse> {
  const loginRes = await fetch(`${API}/auth/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      email: params.email,
      password: params.password,
      deviceId: params.deviceId,
    }),
  });

  if (loginRes.ok) {
    return loginRes.json();
  }

  const body = (await loginRes.json()) as ApiErrorBody;
  if (loginRes.status !== 403 || body.error?.code !== "MFA_REQUIRED") {
    throw new Error(body.error?.message ?? "Login failed");
  }

  const mfaToken = body.error.details?.mfaToken;
  if (!mfaToken) throw new Error("MFA challenge missing");

  const code = await params.askMfaCode();
  const completeRes = await fetch(`${API}/auth/mfa/complete`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      mfaToken,
      code,
      deviceId: params.deviceId,
    }),
  });

  if (!completeRes.ok) {
    const err = (await completeRes.json()) as ApiErrorBody;
    throw new Error(err.error?.message ?? "MFA failed");
  }
  return completeRes.json();
}
```

Для admin-panel замените пути на `/admin/auth/login` и `/admin/auth/mfa/complete`.

---

## Extension: дополнительные рекомендации (без обязательного кода в этом PR)

Backend не требует изменений extension в том же релизе, что MFA API, но для безопасной реализации:

1. **MFA** — как в чеклисте выше на экране login/register.
2. **Background fetch** — принимать только URL с allowlist хостов (`*.markethacker.ru`, WB/Ozon/Lamoda) и `sender.id === chrome.runtime.id`.
3. Токены предпочтительно в `chrome.storage.session`; при logout отзывать access через body `accessToken`.

Реализацию делает фронтенд по этому документу.

---

## Rate limits

| Endpoint | Лимит |
|----------|--------|
| `POST /auth/login` | 5/min |
| `POST /auth/mfa/complete` | 5/min |
| `POST /auth/mfa/setup` | 3/min |
| `POST /auth/mfa/verify` | 5/min |
| `POST /admin/auth/login` | 5/min |
| `POST /admin/auth/mfa/complete` | 5/min |
