# Billing Intro Offer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Платный вводный оффер (N ₽ / N дней) с автосписанием полной monthly-цены, win-back (глобальный + точечный), UI admin/manager-portal и интеграционная дока для расширения.

**Architecture:** Конфиг в `platform_settings`; статус подписки `intro`; `payment_type=subscription_intro`; таблица `billing_intro_offer_grants`; renewals включают `intro` → после успеха `active`.

**Tech Stack:** FastAPI / SQLAlchemy / Alembic / Pydantic; admin-panel + manager-portal Next.js/TS.

**Spec:** `docs/superpowers/specs/2026-11-27-billing-intro-offer-design.md`

## Global Constraints

- Только monthly; целевой тариф из settings, не хардкод `pro`.
- «Уже покупал» = успешные `subscription_initial` | `subscription_renewal` | `subscription_intro`.
- Отдельный статус `intro` (не `trialing`).
- Extension UI не трогаем; нужна `docs/integrations/…`.
- Нет `any` / TODO / временных заглушек.
- Коммитить только по просьбе пользователя.
- Тесты: `cd backend && uv run pytest <path> -v`.

## File map

Создать:

- `backend/alembic/versions/20261127_0067_billing_intro_offer.py`
- `backend/src/markethacker/modules/billing/domain/intro_offer_grant.py`
- `backend/src/markethacker/modules/billing/domain/intro_offer.py` (eligibility pure helpers)
- `backend/src/markethacker/modules/billing/infrastructure/intro_offer_grant_repository.py`
- `backend/src/markethacker/modules/billing/application/intro_offer_service.py`
- `backend/tests/unit/billing/test_intro_offer_eligibility.py`
- `backend/tests/unit/billing/test_intro_offer_status_entitlement.py`
- `backend/tests/integration/test_billing_intro_offer.py`
- `backend/tests/integration/test_admin_intro_offer_grants.py`
- `docs/integrations/billing-intro-offer-client.md`
- `admin-panel` / `manager-portal` UI pieces (см. задачи)

Изменить (ключевое):

- `billing/domain/models.py` — `SUBSCRIPTION_STATUSES` + `intro`
- `billing/domain/subscription_entitlement.py`
- `billing/infrastructure/repository.py` — renewals/expire
- `billing/providers/yookassa/service.py` — intro checkout + activation
- `billing/application/service.py`, `api/router.py`, `api/schemas.py`
- `billing/application/checkout_intent_service.py` (если нужен тип intent)
- `platform_settings/application/defaults.py`, `service.py`
- `admin/api/schemas.py`, `router.py`
- `manager-portal` billing workspace + api
- `admin-panel` settings + user page
- `docs/architecture/billing.md`

---

### Task 1: Status `intro` + entitlement + renewals/expire filters

**Files:**
- Modify: `subscription_entitlement.py`, `models.py` (`SUBSCRIPTION_STATUSES`)
- Modify: `repository.py` (`get_subscriptions_due_for_renewal`, `expire_stale_subscriptions`)
- Modify: admin status regex / manager `subscription-status.ts` / extension entitlements if status lists hardcoded
- Test: `tests/unit/billing/test_intro_offer_status_entitlement.py`

- [ ] Add `"intro"` to statuses and period-bound entitlement
- [ ] Include `intro` in renewal due query and expire candidates
- [ ] Unit tests: entitled while period open; renewals select intro; expire after period without autopay

### Task 2: Platform settings keys

**Files:**
- Modify: `platform_settings/application/defaults.py`, `service.py` (validate/coerce/admin_view)
- Modify: `admin/api/schemas.py`, `router.py` flatten patch
- Test: extend `tests/unit/test_platform_settings.py`

- [ ] Keys: enabled, price_rub, period_days, plan_id, allow_returning_subscribers
- [ ] Validation: enabled ⇒ plan_id UUID + period 1..365 + price ≥ 1; plan must exist & monthly > 0 (async check in service or at intro purchase)

### Task 3: Grant table + repository + admin grant API

**Files:**
- Create ORM + alembic `20261127_0067` (down_revision `20261127_0066`)
- Create repo + service grant/revoke/consume
- Admin routes on user: list/grant/revoke
- Test: integration admin grants

- [ ] Migration + model registration in `all_models.py`
- [ ] Grant/revoke/consume; one active grant per user
- [ ] Admin API + audit log pattern like MH credits

### Task 4: Eligibility + public GET + intro checkout

**Files:**
- Create `intro_offer.py` helpers + `intro_offer_service.py`
- Modify billing router/schemas/service/checkout intent/yookassa
- Test: unit eligibility + integration checkout activation

- [ ] `GET /billing/intro-offer`
- [ ] `POST /billing/subscription/intro` → intent → pay → `subscription_intro` → status `intro`, N days
- [ ] Consume grant on success
- [ ] Block yearly; block if not eligible

### Task 5: Activation/renewal wiring in YooKassa

**Files:**
- Modify `yookassa/service.py` `_process_success` / create checkout for intro
- Tests for period_days from meta `intro_period_days` and status `intro`; renewal from intro → active

- [ ] Meta carries `intro_period_days` / `payment_type=subscription_intro`
- [ ] Renewal of intro charges `price_monthly`, sets `active`

### Task 6: Upgrade during intro

**Files:**
- Ensure `create_checkout_session` / upgrade allowed when status=`intro`
- Test: upgrade to full pro or enterprise ends intro via `subscription_initial` → `active`

### Task 7: Admin UI settings + user grants

**Files:**
- `admin-panel` billing settings page + user page panel
- `admin-panel/src/lib/api.ts`

### Task 8: Manager-portal UI

**Files:**
- `billing-workspace.tsx`, `api.ts`, status labels

- [ ] Show intro CTA when eligible; launch intro checkout

### Task 9: Integration docs + architecture billing.md

**Files:**
- Create `docs/integrations/billing-intro-offer-client.md`
- Update `docs/architecture/billing.md` short section
- Link from `docs/README.md` / architecture README if needed

### Task 10: Verification

- [ ] `uv run pytest` targeted suites green
- [ ] `ruff check` on touched backend paths
