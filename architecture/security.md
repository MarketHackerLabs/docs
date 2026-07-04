# Безопасность

## Уровни защиты

```
┌─────────────────────────────────────────┐
│ Transport: TLS 1.3, HSTS                │
├─────────────────────────────────────────┤
│ API: Rate limit, input validation,      │
│      CORS, security headers             │
├─────────────────────────────────────────┤
│ Auth: JWT, MFA, session management      │
├─────────────────────────────────────────┤
│ Data: Encryption at rest (credentials), │
│       row-level org isolation           │
├─────────────────────────────────────────┤
│ Ops: Audit logs, secrets in env/vault,  │
│      dependency scanning                │
└─────────────────────────────────────────┘
```

## Transport

- **TLS 1.3** обязателен в production.
- **HSTS** с `max-age=31536000; includeSubDomains`.
- Сертификаты через Let's Encrypt или cloud provider.

## API Security

### Валидация входных данных

- Pydantic v2 на всех эндпоинтах.
- Строгие типы, `extra = "forbid"` в request schemas.
- Санитизация строковых полей.

### Security headers

```
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Content-Security-Policy: default-src 'none'
Strict-Transport-Security: max-age=31536000; includeSubDomains
```

### Rate limiting

Redis-based sliding window на чувствительных эндпоинтах (см. [Аутентификация](./authentication.md)).

Глобальный лимит: 100 req/min per user для API-эндпоинтов.

## Шифрование credentials маркетплейсов

Credentials для API Wildberries/Ozon — наиболее чувствительные данные системы.

### Алгоритм

- **AES-256-GCM** с уникальным nonce на каждую запись.
- Ключ шифрования (`ENCRYPTION_KEY`) — из env, 32 байта.
- Поддержка **ротации ключей** через `key_version`.

### Хранение

```python
class MarketplaceCredential:
    encrypted_secret: bytes   # ciphertext
    nonce: bytes              # 12 bytes для GCM
    key_version: int          # версия ключа
```

В БД **никогда** не хранится plaintext. Расшифровка — только в application layer, результат **никогда** не отдаётся клиенту.

### WB Portal Proxy — изоляция cookies

Для portal-сессии Wildberries применяется дополнительное разделение:

| Ключ | Браузер менеджера | Сервер прокси |
|------|:-----------------:|:-------------:|
| `authorizev3` / `WBTokenV3` | ✅ bootstrap | ✅ vault |
| `wbx-validation-key` | ❌ | ✅ vault |
| `x-supplier-id` | ❌ | ✅ vault |
| `locale` | ✅ relay | ✅ merge |

Сервер добавляет `SERVER_ONLY_COOKIE_KEYS` к каждому исходящему запросу к WB. Браузер менеджера не может прочитать или подменить эти значения. Подробнее: [WB Portal Proxy](./wb-portal-proxy.md#безопасность-credentials-в-прокси).

### Ротация ключей

1. Новый ключ → `key_version = 2`.
2. Фоновая задача перешифровывает записи с `key_version = 1`.
3. Старый ключ сохраняется до завершения миграции.

## Пароли

- **Argon2id** (предпочтительно) или bcrypt.
- Минимум 8 символов, проверка на common passwords (haveibeenpwned API).
- Password hash никогда не логируется и не возвращается в API.

## Multi-tenancy isolation

### Application layer

- `org_id` берётся из пути запроса (`/organizations/{org_id}/...`), а не из JWT — токен не содержит org-контекста (см. [Контроль доступа](./access-control.md)).
- `require_org_path_context` проверяет членство пользователя в `org_id` из пути и только затем устанавливает `current_org_id` для RLS — подмена org через query/body невозможна.
- Repository-методы принимают `org_id` как обязательный параметр.

### Database layer (RLS)

```sql
ALTER TABLE marketplace_accounts ENABLE ROW LEVEL SECURITY;

CREATE POLICY org_isolation ON marketplace_accounts
    USING (org_id = current_setting('app.current_org_id')::uuid);
```

RLS — дополнительный слой. Даже при ошибке в application layer PostgreSQL не отдаст чужие данные.

## Audit log

Все чувствительные действия записываются:

| Действие | Примеры |
|----------|---------|
| Auth | login, logout, failed login, MFA setup, token reuse |
| Access | member-access grant/revoke, org ownership change |
| Data | credential access, marketplace account link/unlink |
| Org | member invite/remove, org settings change |

```json
{
  "userId": "uuid",
  "orgId": "uuid",
  "action": "marketplace_account.linked",
  "resourceType": "marketplace_account",
  "resourceId": "uuid",
  "metadata": { "marketplace": "wildberries" },
  "ip": "1.2.3.4",
  "userAgent": "...",
  "createdAt": "2026-06-05T10:00:00Z"
}
```

> Формат соответствует JSON-ответам admin API. В PostgreSQL колонки остаются в snake_case (`user_id`, `created_at`, …).

Audit log — append-only, без UPDATE/DELETE.

## Secrets management

| Среда | Подход |
|-------|--------|
| Development | `.env` файл (в `.gitignore`) |
| CI | GitHub Secrets |
| Production | Env vars → Vault / cloud secrets manager |

Секреты, которые не должны попасть в код:

```
DATABASE_URL
REDIS_URL
JWT_SECRET
ENCRYPTION_KEY
```

## Dependency security

- `pip-audit` / `safety` в CI pipeline.
- Dependabot для автоматических PR на обновления.
- Pinning версий в `pyproject.toml` / `uv.lock`.

## OWASP Top 10 — митигации

| Риск | Митигация |
|------|-----------|
| Injection | Pydantic validation, parameterized queries (SQLAlchemy) |
| Broken Auth | JWT rotation, MFA, rate limiting |
| Sensitive Data Exposure | Encryption at rest, TLS, no credentials in logs |
| Broken Access Control | Owner-based ownership + explicit member-access гранты + RLS |
| Security Misconfiguration | Security headers, minimal permissions, no debug in prod |
| SSRF | Whitelist для исходящих запросов к MP API |
