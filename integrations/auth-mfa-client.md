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

1. Показать UI ввода кода из приложения **или одноразового резервного кода**
   (`XXXX-XXXX`).
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

`mfaCode` / `code` на complete — TOTP (6–8 цифр) **или** резервный код (8 hex, с дефисами или без).

При неверном коде — **401** `UNAUTHORIZED`.

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

В `code` допустим и резервный код, например `"AB12-CD34"`.

**200** — та же `TokenResponse`, что у login.

| Ошибка | Когда |
|--------|--------|
| 401 | Неверный/просроченный `mfaToken` или неверный код |
| 400 | MFA не включена / валидация полей |

### 4. Настройка MFA (уже залогинен)

| Метод | Путь | Auth | Назначение |
|-------|------|------|------------|
| POST | `/api/v1/auth/mfa/setup` | Bearer | Получить `secret` + `provisioningUri` (otpauth) |
| POST | `/api/v1/auth/mfa/verify` | Bearer | Подтвердить TOTP, **включить** MFA, выдать резервные коды |
| GET | `/api/v1/auth/mfa/backup-codes` | Bearer | Сколько неиспользованных резервных кодов осталось |
| POST | `/api/v1/auth/mfa/backup-codes/regenerate` | Bearer | Перевыпуск (`password` + TOTP/резервный код) |
| POST | `/api/v1/auth/mfa/disable` | Bearer | Отключить MFA (`password` + TOTP/резервный код) |

**Ответ `verify`:**

```json
{
  "data": {
    "enabled": true,
    "backupCodes": ["AB12-CD34", "…"]
  }
}
```

`backupCodes` — **только при первом включении** (10 одноразовых кодов). Показать пользователю один раз; в БД хранятся только хеши. Повторный `verify` при уже включённой MFA вернёт `backupCodes: null`.

`regenerate` возвращает `{ "backupCodes": […] }` и инвалидирует предыдущие неиспользованные.

Manager Portal: Настройки → Безопасность — включение MFA, показ/перевыпуск резервных кодов, отключение.

### Extension

В расширении нужна полная поддержка того же контракта: вход с MFA
(`MFA_REQUIRED` → `/auth/mfa/complete`), настройка/отключение MFA, смена и
восстановление пароля. Реализацию делает разработчик расширения по этому
документу и [password-recovery.md](./password-recovery.md).

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
[ ] Поле кода: TOTP или резервный `XXXX-XXXX`, autocomplete=one-time-code
[ ] После setup/verify показать backupCodes один раз (копирование / скачивание)
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

## Extension

Расширение должно реализовать тот же auth-контракт, что и Manager Portal:
MFA на входе, настройка/отключение MFA, смена и восстановление пароля.
Делает разработчик расширения; backend уже готов.

Дополнительно:

1. **MFA** — по чеклисту выше на экране login (и в настройках аккаунта).
2. **Background fetch** — только URL с allowlist хостов (`*.markethacker.ru`, WB/Ozon/Lamoda) и `sender.id === chrome.runtime.id`.
3. Токены предпочтительно в `chrome.storage.session`; при logout отзывать access через body `accessToken`.

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
