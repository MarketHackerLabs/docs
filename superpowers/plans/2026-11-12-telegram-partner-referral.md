# Telegram Partner Referral Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Партнёрская кампания `referral_link` отдаёт Telegram deep link `t.me/{bot}?start=ref_{CODE}`; новый пользователь по ссылке регистрируется в боте с first-touch attribution и клиентским бонусом кампании.

**Architecture:** Префикс `ref_` в `/start` отделяет партнёрку от Redis link-кода. Click/attribution/promo — существующие partners/billing сервисы. Для `discount` код кладётся в Redis login-token и после `POST /auth/telegram/exchange` попадает в `setPendingPromoCode` на Manager Portal. `telegramReferralUrl` строится рядом с веб-`referralUrl`.

**Tech Stack:** FastAPI / SQLAlchemy / Redis; admin-panel и manager-portal Next.js / TypeScript.

**Spec:** `docs/superpowers/specs/2026-11-12-telegram-partner-referral-design.md`

## Global Constraints

- Формат deep link: `https://t.me/{PRODUCT_TELEGRAM_BOT_USERNAME}?start=ref_{PUBLIC_CODE}`.
- Payload `ref_` = партнёрка; без префикса = текущий account-link; пустой `/start` = текущая регистрация/вход.
- First-touch; существующий TG-аккаунт — только login, без re-attribute и без повторного бонуса.
- `free_period` → `PromoService.redeem_promo`; `discount` → `pending_promo_code` в login token (не `redeem_promo`).
- Сбой бонуса не откатывает регистрацию/attribution.
- Нет новых таблиц; веб-`?ref=` не менять.
- Нет `any` / `Record<string, any>`; UI без внутренних имён сервисов/SQL.
- Репозитории раздельные: `docs/`, `backend/`, `admin-panel/`, `manager-portal/`.
- Коммитить только по явной просьбе пользователя.
- Тесты backend: `cd backend && uv run pytest <path> -v`.
- Линтер backend: `cd backend && uv run ruff check <paths>`.
- Перед правками кода — согласование с пользователем по правилу репо (если сессия новая).

## File map

Создать:

- `backend/tests/unit/test_telegram_partner_referral.py`
- `backend/tests/unit/test_telegram_login_tokens.py` (если удобнее отдельно от referral)

Изменить:

- `backend/src/markethacker/modules/partners/application/campaign_service.py` — `telegram_referral_url`
- `backend/src/markethacker/modules/partners/api/schemas.py` — `telegram_referral_url`
- `backend/src/markethacker/modules/partners/api/admin_router.py` — заполнение TG URL
- `backend/src/markethacker/modules/partners/api/router.py` — `/me/campaigns` + `_campaign_response`
- `backend/src/markethacker/modules/telegram/infrastructure/tokens.py` — JSON login payload + `pending_promo_code`
- `backend/src/markethacker/modules/telegram/application/service.py` — разбор `ref_`, attribution, bonus, login token
- `backend/src/markethacker/modules/telegram/api/schemas.py` — `TelegramExchangeResponse.pending_promo_code`
- `backend/src/markethacker/modules/auth/api/router.py` — прокинуть `pending_promo_code` (если response из telegram schemas через auth)
- `manager-portal/src/lib/api.ts` — тип exchange + поле
- `manager-portal/src/app/login/telegram/page.tsx` — `setPendingPromoCode`
- `admin-panel/src/lib/api.ts` — `telegramReferralUrl` на кампании
- `admin-panel/src/app/(admin)/partners/[id]/page.tsx` — копирование TG-ссылки
- `admin-panel/src/app/(admin)/partners/campaigns/[id]/page.tsx` — копирование TG-ссылки
- `docs/architecture/partners.md`
- `docs/architecture/telegram-bot.md`

---

### Task 1: `telegramReferralUrl` (CampaignService + API)

**Files:**
- Modify: `backend/src/markethacker/modules/partners/application/campaign_service.py`
- Modify: `backend/src/markethacker/modules/partners/api/schemas.py`
- Modify: `backend/src/markethacker/modules/partners/api/admin_router.py`
- Modify: `backend/src/markethacker/modules/partners/api/router.py`
- Test: `backend/tests/unit/test_telegram_partner_referral.py`

**Interfaces:**
- Produces:
  - `CampaignService.telegram_referral_url(campaign: PartnerCampaign, bot_username: str) -> str | None`
  - `CampaignResponse.telegram_referral_url: str | None = None` (JSON camelCase `telegramReferralUrl`)

- [ ] **Step 1: Write failing unit tests for URL builder**

```python
"""Telegram partner referral helpers."""

from __future__ import annotations

from types import SimpleNamespace

from markethacker.modules.partners.application.campaign_service import CampaignService


def test_telegram_referral_url_builds_ref_prefix() -> None:
    campaign = SimpleNamespace(public_code="ABC123")
    url = CampaignService.telegram_referral_url(campaign, "mhcatdev_bot")
    assert url == "https://t.me/mhcatdev_bot?start=ref_ABC123"


def test_telegram_referral_url_strips_at() -> None:
    campaign = SimpleNamespace(public_code="X1Y")
    url = CampaignService.telegram_referral_url(campaign, "@MyBot")
    assert url == "https://t.me/MyBot?start=ref_X1Y"


def test_telegram_referral_url_none_without_username() -> None:
    campaign = SimpleNamespace(public_code="ABC123")
    assert CampaignService.telegram_referral_url(campaign, "") is None
    assert CampaignService.telegram_referral_url(campaign, "   ") is None


def test_telegram_referral_url_none_when_payload_too_long() -> None:
    # Telegram start payload max 64 chars: "ref_" + code
    code = "A" * 61
    campaign = SimpleNamespace(public_code=code)
    assert CampaignService.telegram_referral_url(campaign, "bot") is None
```

- [ ] **Step 2: Run tests — expect fail (method missing)**

Run: `cd backend && uv run pytest tests/unit/test_telegram_partner_referral.py::test_telegram_referral_url_builds_ref_prefix -v`  
Expected: FAIL (AttributeError / import error on `telegram_referral_url`)

- [ ] **Step 3: Implement `telegram_referral_url`**

В `CampaignService` (рядом с `referral_url`):

```python
@staticmethod
def telegram_referral_url(campaign: PartnerCampaign, bot_username: str) -> str | None:
    username = (bot_username or "").lstrip("@").strip()
    if not username:
        return None
    payload = f"ref_{campaign.public_code}"
    if len(payload) > 64:
        return None
    return f"https://t.me/{username}?start={payload}"
```

`referral_url` не менять. Метод может быть `@staticmethod` (как в тестах) — если нужен instance method без `self` логики, оставить static.

- [ ] **Step 4: Schema + routers**

В `CampaignResponse` добавить:

```python
telegram_referral_url: str | None = None
```

`admin_router._campaign`:

```python
async def _campaign(c: PartnerCampaign, session: AsyncSession) -> CampaignResponse:
    settings = get_settings()
    svc = CampaignService(session)
    data = CampaignResponse.model_validate(c, from_attributes=True)
    data.referral_url = await svc.referral_url(c, settings.manager_portal_url)
    data.telegram_referral_url = svc.telegram_referral_url(
        c, settings.product_telegram_bot_username
    )
    return data
```

`router._campaign_response` расширить:

```python
def _campaign_response(
    c: PartnerCampaign,
    referral_url: str | None = None,
    telegram_referral_url: str | None = None,
) -> CampaignResponse:
    data = CampaignResponse.model_validate(c, from_attributes=True)
    if referral_url:
        data.referral_url = referral_url
    data.telegram_referral_url = telegram_referral_url
    return data
```

В `my_campaigns` передавать оба URL (как admin).

- [ ] **Step 5: Run unit tests + ruff**

Run: `cd backend && uv run pytest tests/unit/test_telegram_partner_referral.py -v && uv run ruff check src/markethacker/modules/partners tests/unit/test_telegram_partner_referral.py`  
Expected: PASS

- [ ] **Step 6: Commit (только если пользователь попросил)**

```bash
cd backend && git add src/markethacker/modules/partners tests/unit/test_telegram_partner_referral.py && git commit -m "feat(partners): expose telegramReferralUrl for referral campaigns"
```

---

### Task 2: Login token с `pending_promo_code`

**Files:**
- Modify: `backend/src/markethacker/modules/telegram/infrastructure/tokens.py`
- Modify: `backend/src/markethacker/modules/telegram/application/service.py` (`exchange_login_token`, все вызовы `issue_login_token` / `consume_login_token`)
- Modify: `backend/src/markethacker/modules/telegram/api/schemas.py`
- Modify: `backend/src/markethacker/modules/auth/api/router.py` (`telegram_exchange` response)
- Test: `backend/tests/unit/test_telegram_login_tokens.py`

**Interfaces:**
- Produces:
  - `issue_login_token(user_id: uuid.UUID, *, pending_promo_code: str | None = None) -> str`
  - `consume_login_token(token: str) -> tuple[uuid.UUID, str | None] | None`
  - `TelegramExchangeResponse.pending_promo_code: str | None = None`
  - `TelegramService.exchange_login_token(...) -> tuple[TokenPair, bool, str | None]`  
    (pair, needs_completion, pending_promo_code)

- [ ] **Step 1: Failing tests for token roundtrip**

```python
"""Redis login tokens with optional pending promo."""

from __future__ import annotations

import json
import uuid
from unittest.mock import AsyncMock, patch

import pytest

from markethacker.modules.telegram.infrastructure import tokens as token_mod


@pytest.mark.asyncio
async def test_issue_and_consume_login_token_with_promo() -> None:
    user_id = uuid.uuid4()
    store: dict[str, str] = {}

    redis = AsyncMock()

    async def _set(key: str, value: str, ex: int | None = None) -> None:
        store[key] = value

    async def _get(key: str) -> str | None:
        return store.get(key)

    async def _delete(key: str) -> None:
        store.pop(key, None)

    redis.set = _set
    redis.get = _get
    redis.delete = _delete

    with patch.object(token_mod, "get_redis", return_value=redis):
        token = await token_mod.issue_login_token(user_id, pending_promo_code="PROMO1")
        result = await token_mod.consume_login_token(token)

    assert result is not None
    got_user, promo = result
    assert got_user == user_id
    assert promo == "PROMO1"


@pytest.mark.asyncio
async def test_consume_legacy_plain_user_id() -> None:
    user_id = uuid.uuid4()
    store = {f"telegram:login:tok": str(user_id)}
    redis = AsyncMock()
    redis.get = AsyncMock(return_value=store["telegram:login:tok"])
    redis.delete = AsyncMock()

    with patch.object(token_mod, "get_redis", return_value=redis):
        result = await token_mod.consume_login_token("tok")

    assert result == (user_id, None)
```

Подстроить ключи/`patch` под фактический `_LOGIN_PREFIX`, если тесты падают на деталях мока — сохранить контракт assert’ов.

- [ ] **Step 2: Run — expect fail**

Run: `cd backend && uv run pytest tests/unit/test_telegram_login_tokens.py -v`  
Expected: FAIL (сигнатура / нет promo)

- [ ] **Step 3: Implement tokens**

```python
import json

async def issue_login_token(
    user_id: uuid.UUID,
    *,
    pending_promo_code: str | None = None,
) -> str:
    token = _new_code(40)
    redis = get_redis()
    payload = json.dumps(
        {
            "user_id": str(user_id),
            "pending_promo_code": pending_promo_code,
        }
    )
    await redis.set(f"{_LOGIN_PREFIX}{token}", payload, ex=_LOGIN_TTL_SECONDS)
    return token


async def consume_login_token(token: str) -> tuple[uuid.UUID, str | None] | None:
    redis = get_redis()
    key = f"{_LOGIN_PREFIX}{token.strip()}"
    raw = await redis.get(key)
    if raw is None:
        return None
    await redis.delete(key)
    text = raw if isinstance(raw, str) else raw.decode()
    try:
        data = json.loads(text)
    except json.JSONDecodeError:
        try:
            return uuid.UUID(text), None
        except ValueError:
            return None
    if not isinstance(data, dict):
        return None
    try:
        user_id = uuid.UUID(str(data["user_id"]))
    except (KeyError, ValueError, TypeError):
        return None
    promo = data.get("pending_promo_code")
    promo_s = promo.strip().upper() if isinstance(promo, str) and promo.strip() else None
    return user_id, promo_s
```

- [ ] **Step 4: Update `TelegramService.exchange_login_token` и callers**

- `consume_login_token` → unpack `(user_id, pending_promo_code)`.
- Return `(pair, needs_completion, pending_promo_code)`.
- Существующие `issue_login_token(user.id)` без promo остаются валидны (default `None`).
- `TelegramExchangeResponse` + auth router:

```python
return TelegramExchangeResponse(
    access_token=pair.access_token,
    refresh_token=pair.refresh_token,
    needs_profile_completion=needs_completion,
    pending_promo_code=pending_promo_code,
)
```

Найти все вызовы `consume_login_token` / `exchange_login_token` и обновить.

- [ ] **Step 5: Tests + ruff**

Run: `cd backend && uv run pytest tests/unit/test_telegram_login_tokens.py -v && uv run ruff check src/markethacker/modules/telegram src/markethacker/modules/auth/api/router.py tests/unit/test_telegram_login_tokens.py`  
Expected: PASS

- [ ] **Step 6: Commit (если попросили)**

```bash
cd backend && git add src/markethacker/modules/telegram src/markethacker/modules/auth/api/router.py tests/unit/test_telegram_login_tokens.py && git commit -m "feat(telegram): carry pending promo code in login token"
```

---

### Task 3: Парсинг `ref_` и партнёрский `/start`

**Files:**
- Modify: `backend/src/markethacker/modules/telegram/application/service.py`
- Test: `backend/tests/unit/test_telegram_partner_referral.py` (дополнить)

**Interfaces:**
- Produces:
  - `parse_partner_ref_start_payload(raw: str | None) -> str | None`  
    (module-level в `service.py` или `partners/domain` — предпочтительно рядом с telegram service как чистая функция)
  - `_handle_start` ветка: если `parse_partner_ref_start_payload(link_code)` вернул code → `_handle_partner_ref_start(...)`; иначе текущий link-code path

- [ ] **Step 1: Failing parse tests**

```python
from markethacker.modules.telegram.application.service import parse_partner_ref_start_payload


def test_parse_partner_ref_start_payload() -> None:
    assert parse_partner_ref_start_payload("ref_ABC123") == "ABC123"
    assert parse_partner_ref_start_payload("ref_abc") == "ABC"  # normalize upper via caller OR parse returns raw upper
    assert parse_partner_ref_start_payload("REF_ABC") is None  # только нижний префикс ref_
    assert parse_partner_ref_start_payload("abc123") is None
    assert parse_partner_ref_start_payload(None) is None
    assert parse_partner_ref_start_payload("ref_") is None
```

Зафиксировать в реализации: префикс строго `ref_` (lowercase); код после префикса нормализуется через `normalize_public_code` внутри handler (parse возвращает сырой хвост или уже upper — выбрать одно и отразить в тестах). Рекомендация: parse возвращает хвост как есть; handler вызывает `normalize_public_code`.

Уточнённые asserts:

```python
assert parse_partner_ref_start_payload("ref_ABC123") == "ABC123"
assert parse_partner_ref_start_payload("ref_abc_1") == "abc_1"
assert parse_partner_ref_start_payload("REF_ABC") is None
assert parse_partner_ref_start_payload("ref_") is None
```

- [ ] **Step 2: Implement parse + `_handle_start` branch**

```python
_REF_PREFIX = "ref_"


def parse_partner_ref_start_payload(raw: str | None) -> str | None:
    if not raw or not raw.startswith(_REF_PREFIX):
        return None
    code = raw[len(_REF_PREFIX) :]
    return code or None
```

В `_handle_start` **до** `_link_with_code`:

```python
if link_code:
    partner_code = parse_partner_ref_start_payload(link_code)
    if partner_code is not None:
        await self._handle_partner_ref_start(
            telegram_user_id=telegram_user_id,
            chat_id=chat_id,
            username=username,
            first_name=first_name,
            raw_code=partner_code,
        )
        return
    await self._link_with_code(...)
    return
```

- [ ] **Step 3: Implement `_handle_partner_ref_start`**

Поведение по spec:

1. Если TG account уже есть → update profile + `_send_login_button` (без attribution/bonus). Можно коротко не упоминать партнёрку.
2. Иначе:
   - Попытка `ClickService(session).resolve(raw_code, landing_path="telegram:/start", visitor_key=telegram_user_id, record_click=True)` — при `NotFoundError`/`ValidationError` click_ok=False.
   - `_register_telegram_user` + `create_account` (как обычный `/start`).
   - Если click/resolve успешен: `AttributionService.try_attribute_by_code(user.id, normalized, channel_type="referral_link", source="referral_link")`.
   - Если attribution есть и `campaign.client_bonus_type == "free_period"`: `PromoService(session).redeem_promo(user.id, campaign.public_code)` в try/except → log, не падать.
   - Если attribution есть и `client_bonus_type == "discount"`: `pending_promo = campaign.public_code`.
   - `issue_login_token(user.id, pending_promo_code=pending_promo)`.
   - Сообщение: при неуспешном resolve/нормализации — «Аккаунт создан. Приглашение по ссылке не сработало.» + кнопка кабинета; при успехе — обычный текст создания аккаунта + кнопка.
   - `schedule_welcome_message`.

Импорты partners/billing — ленивые внутри метода (как в auth register), чтобы не плодить циклы.

Не логировать PII сверх текущих практик бота.

- [ ] **Step 4: Unit-test parse; smoke-тест handler с моками (минимум)**

Добавить тесты parse (обязательно). Для handler — по возможности mock `ClickService`/`AttributionService`/`PromoService`/`issue_login_token`/`_client.send_message`; если в проекте нет удобного harness для `TelegramService`, ограничиться parse + ручным smoke из spec.

- [ ] **Step 5: ruff + pytest**

Run: `cd backend && uv run pytest tests/unit/test_telegram_partner_referral.py -v && uv run ruff check src/markethacker/modules/telegram/application/service.py`  
Expected: PASS

- [ ] **Step 6: Commit (если попросили)**

```bash
cd backend && git add src/markethacker/modules/telegram/application/service.py tests/unit/test_telegram_partner_referral.py && git commit -m "feat(telegram): attribute partner referral on /start ref_CODE"
```

---

### Task 4: Manager Portal — принять pending promo

**Files:**
- Modify: `manager-portal/src/lib/api.ts`
- Modify: `manager-portal/src/app/login/telegram/page.tsx`

**Interfaces:**
- Consumes: `TelegramExchangeResponse.pendingPromoCode`
- Produces: вызов `setPendingPromoCode` из `@/lib/referral` после успешного exchange

- [ ] **Step 1: Расширить тип `exchangeTelegramLogin`**

```typescript
exchangeTelegramLogin: (
  token: string,
  deviceId?: string,
  mfaCode?: string,
) =>
  request<{
    accessToken: string;
    refreshToken: string;
    needsProfileCompletion: boolean;
    pendingPromoCode?: string | null;
  }>("/auth/telegram/exchange", {
    method: "POST",
    body: JSON.stringify({ token, deviceId, mfaCode }),
  }),
```

- [ ] **Step 2: На странице login/telegram**

Импорт:

```typescript
import { setPendingPromoCode } from "@/lib/referral";
```

В `exchange` после успешного `result`:

```typescript
if (result.pendingPromoCode) {
  setPendingPromoCode(result.pendingPromoCode);
}
await finish(
  result.accessToken,
  result.refreshToken,
  result.needsProfileCompletion,
);
```

- [ ] **Step 3: Ручная проверка типов**

Run: `cd manager-portal && npx tsc --noEmit` (или принятый в репо скрипт `npm run typecheck` / `pnpm`, если есть)  
Expected: без ошибок по изменённым файлам

- [ ] **Step 4: Commit (если попросили)**

```bash
cd manager-portal && git add src/lib/api.ts src/app/login/telegram/page.tsx && git commit -m "feat(auth): store partner pending promo after Telegram login exchange"
```

---

### Task 5: Admin Panel — копирование Telegram-ссылки

**Files:**
- Modify: `admin-panel/src/lib/api.ts` — поле на `AdminPartnerCampaign` (или эквивалент)
- Modify: `admin-panel/src/app/(admin)/partners/[id]/page.tsx`
- Modify: `admin-panel/src/app/(admin)/partners/campaigns/[id]/page.tsx`

**Interfaces:**
- Consumes: `telegramReferralUrl: string | null`
- UI copy рядом с веб-ссылкой; если `null` — не показывать кнопку TG (или показать «Telegram-ссылка недоступна» без технических деталей)

- [ ] **Step 1: Тип API**

В интерфейсе кампании:

```typescript
referralUrl: string | null;
telegramReferralUrl: string | null;
```

- [ ] **Step 2: Список кампаний на `partners/[id]`**

Рядом с блоком веб-`referralUrl` добавить второй ряд для `telegramReferralUrl` (отдельный `copiedKey`, например `` `${c.id}-tg` ``). После create: если есть `telegramReferralUrl`, показать его тоже (или оба) в `createdLink` UI — расширить state до двух ссылок, если сейчас одна строка.

Минимально приемлемо: в таблице кампаний две copy-кнопки с подписями «Веб» / «Telegram» (пользовательские слова, не имена API).

- [ ] **Step 3: `campaigns/[id]`**

Сейчас `shareValue` = web URL или promo code. Добавить отдельное копирование Telegram URL, если `campaign.telegramReferralUrl`.

- [ ] **Step 4: Typecheck**

Run: `cd admin-panel && npx tsc --noEmit` (или скрипт репо)  
Expected: OK

- [ ] **Step 5: Commit (если попросили)**

```bash
cd admin-panel && git add src/lib/api.ts "src/app/(admin)/partners" && git commit -m "feat(admin): copy Telegram partner referral link"
```

---

### Task 6: Архитектурные docs

**Files:**
- Modify: `docs/architecture/partners.md`
- Modify: `docs/architecture/telegram-bot.md`

- [ ] **Step 1: partners.md**

В секции Attribution добавить пункт:

```markdown
2b. Telegram: `/start ref_CODE` → click + регистрация в боте → first-touch attribution;
    клиентский бонус: `free_period` redeem сразу, `discount` через pending promo в login token.
```

(перенумеровать соседние пункты аккуратно, без ломания смысла)

- [ ] **Step 2: telegram-bot.md**

В таблице потоков:

| `/start ref_CODE` | Партнёрский referral: click, регистрация (если новый), attribution, бонус; существующий — только login |
| `/start CODE` (без `ref_`) | Привязка аккаунта (как сейчас) |

В токенах login: payload JSON `{ user_id, pending_promo_code }`; legacy plain UUID ещё читается.

- [ ] **Step 3: Commit (если попросили)**

```bash
cd docs && git add architecture/partners.md architecture/telegram-bot.md superpowers/plans/2026-11-12-telegram-partner-referral.md && git commit -m "docs: Telegram partner referral flow"
```

---

## Spec coverage checklist

| Spec requirement | Task |
|---|---|
| `telegramReferralUrl` admin + `/partners/me` | 1 |
| Deep link `ref_{CODE}`, payload ≤ 64 | 1, 3 |
| Разведение с link-кодом | 3 |
| Click + attribution на новой регистрации | 3 |
| Existing user → login only | 3 |
| Invalid ref → register without attribution + сообщение | 3 |
| `free_period` redeem | 3 |
| `discount` via login token → portal pending | 2, 3, 4 |
| Admin UI copy | 5 |
| Docs partners + telegram-bot | 6 |
| Tests parse/URL/token | 1, 2, 3 |

## Self-review notes

- Нет TBD/placeholder steps.
- `exchange_login_token` return arity меняется один раз в Task 2; Task 3 только передаёт `pending_promo_code` в `issue_login_token`.
- `CampaignService.telegram_referral_url` static — и unit-тесты, и routers используют одну сигнатуру.
- Веб-cookie path не трогаем.
