# Welcome Onboarding Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** После регистрации (сайт и Telegram) показывать `/welcome` в Manager Portal: дозаполнение профиля для TG, затем шаги установки расширения; завершение → `/dashboard` с флагом на бэкенде и зеркалом в `localStorage`.

**Architecture:** Колонка `users.onboarding_completed_at` + `POST /auth/onboarding/complete` + поля в `/users/me`. Портал: страница `/welcome` (auth-layout), гейты в shell/login/register/telegram, `/complete-profile` → redirect. UI в стиле портала, структура как у референса «Быстрый старт».

**Tech Stack:** FastAPI / SQLAlchemy / Alembic / Pydantic; Manager Portal Next.js / React / TypeScript / Tailwind.

**Spec:** `docs/superpowers/specs/2026-10-18-welcome-onboarding-design.md`

## Global Constraints

- Выход после welcome всегда `/dashboard` (сайт и Telegram).
- TG: шаги 01/02 недоступны, пока не сохранён `complete-profile`.
- Сайт: форма профиля скрыта, сразу шаги.
- Backfill существующих пользователей в миграции: `onboarding_completed_at = created_at`.
- Новые пользователи: `onboarding_completed_at = NULL`.
- Chrome Web Store URL = значение `TELEGRAM_WELCOME_PLUGIN_URL` из backend.
- Шаг 02 CTA: только `https://seller.wildberries.ru`.
- Нет демо-ссылки «как это работает».
- UI без внутренних кодов/stack; `any` / `Record<string, any>` запрещены.
- Коммитить только по явной просьбе пользователя.
- Backend тесты: `cd backend && uv run pytest <path> -v`.
- Backend lint: `cd backend && uv run ruff check <paths>`.
- Portal lint: `cd manager-portal && npm run lint`.

## Review Focus

- Повторный `POST /auth/onboarding/complete` не меняет timestamp и не падает — тест в Task 2.
- Пользователь с заполненным профилем, но `onboarding_completed_at IS NULL`, при login/shell попадает на `/welcome`, а не в портал — проверка в Task 5.
- Устаревшее `localStorage` при `needsOnboarding=true` сбрасывается; при `false` синхронизируется — проверка в Task 3.
- Прямой заход на `/welcome` при завершённом onboarding → `/dashboard` — проверка в Task 4.
- Закладка `/complete-profile` всегда уводит на `/welcome` — проверка в Task 4.

## File map

Создать:

- `backend/alembic/versions/20261018_0061_user_onboarding_completed_at.py`
- `backend/tests/unit/test_user_onboarding.py`
- `manager-portal/src/lib/onboarding.ts`
- `manager-portal/src/lib/extension-links.ts`
- `manager-portal/src/app/welcome/page.tsx`

Изменить:

- `backend/src/markethacker/modules/users/domain/models.py`
- `backend/src/markethacker/modules/users/api/schemas.py`
- `backend/src/markethacker/modules/users/application/service.py`
- `backend/src/markethacker/modules/users/infrastructure/repository.py`
- `backend/src/markethacker/modules/auth/api/schemas.py` (или schemas рядом с telegram — ответ complete)
- `backend/src/markethacker/modules/auth/api/router.py`
- `manager-portal/src/lib/api.ts`
- `manager-portal/src/app/complete-profile/page.tsx`
- `manager-portal/src/app/register/page.tsx`
- `manager-portal/src/app/login/page.tsx`
- `manager-portal/src/app/login/telegram/page.tsx`
- `manager-portal/src/components/manager-shell.tsx`
- `manager-portal/src/lib/nav-persistence.ts` (добавить `/welcome` в blocked paths при необходимости)
- `docs/integrations/telegram-auth-client.md`
- `docs/architecture/telegram-bot.md` (краткая пометка)
- `docs/superpowers/specs/2026-10-18-welcome-onboarding-design.md` (статус уже approved)

---

### Task 1: Schema — `onboarding_completed_at` + миграция с backfill

**Files:**
- Modify: `backend/src/markethacker/modules/users/domain/models.py`
- Create: `backend/alembic/versions/20261018_0061_user_onboarding_completed_at.py`

**Interfaces:**
- Produces: `User.onboarding_completed_at: Mapped[datetime | None]`
- Consumes: none

- [ ] **Step 1: Add column on `User`**

```python
onboarding_completed_at: Mapped[datetime | None] = mapped_column(
    nullable=True,
)
```

(тот же стиль nullable datetime, что `last_login_at` / `last_active_at`)

- [ ] **Step 2: Alembic migration**

`revision = "20261018_0061"`, `down_revision = "20261013_0060"`:

```python
def upgrade() -> None:
    op.add_column(
        "users",
        sa.Column("onboarding_completed_at", sa.DateTime(timezone=True), nullable=True),
    )
    op.execute(
        sa.text(
            "UPDATE users SET onboarding_completed_at = created_at "
            "WHERE onboarding_completed_at IS NULL"
        )
    )


def downgrade() -> None:
    op.drop_column("users", "onboarding_completed_at")
```

Важно: backfill в той же `upgrade` **до** появления нового кода создания пользователей с `NULL`. После деплоя кода новые register/TG create оставляют `NULL` (default колонки).

- [ ] **Step 3: Verify migration revises head**

Run: `cd backend && uv run alembic heads`
Expected: включает `20261018_0061`

- [ ] **Step 4: Commit** (только если пользователь явно попросил)

---

### Task 2: API — me fields + `POST /auth/onboarding/complete`

**Files:**
- Modify: `backend/src/markethacker/modules/users/api/schemas.py`
- Modify: `backend/src/markethacker/modules/users/application/service.py`
- Modify: `backend/src/markethacker/modules/users/infrastructure/repository.py`
- Modify: `backend/src/markethacker/modules/auth/api/schemas.py`
- Modify: `backend/src/markethacker/modules/auth/api/router.py`
- Create: `backend/tests/unit/test_user_onboarding.py`

**Interfaces:**
- Produces:
  - `UserResponse.onboarding_completed_at: datetime | None`
  - `UserResponse.needs_onboarding: bool` (computed / выставляется как `onboarding_completed_at is None`)
  - `UserRepository.mark_onboarding_completed(user_id: uuid.UUID, at: datetime) -> User | None`
  - `UserService.complete_onboarding(user_id: uuid.UUID) -> dict` с ключами camel через schema:  
    `{ "onboardingCompletedAt": datetime, "needsOnboarding": false }`
  - `OnboardingCompleteResponse` (CamelModel): `onboarding_completed_at: datetime`, `needs_onboarding: bool = False`
  - Router: `POST /auth/onboarding/complete`, JWT, limiter разумный (например `30/minute`)

- [ ] **Step 1: Write failing unit tests**

В `test_user_onboarding.py` (стиль как `test_telegram_welcome.py` — mock repo/service):

1. `test_complete_onboarding_sets_timestamp_when_null` — user с `onboarding_completed_at=None` → после complete поле не None, `needsOnboarding` false.
2. `test_complete_onboarding_idempotent` — повторный вызов не меняет уже установленный timestamp.
3. `test_needs_onboarding_true_when_null` / `test_needs_onboarding_false_when_set` — сериализация/`UserMeResponse` или helper: `needs_onboarding == (onboarding_completed_at is None)`.

- [ ] **Step 2: Run tests — expect FAIL**

Run: `cd backend && uv run pytest tests/unit/test_user_onboarding.py -v`
Expected: FAIL (нет метода / поля)

- [ ] **Step 3: Implement repository + service + schemas + router**

- `UserResponse`: поля `onboarding_completed_at`, `needs_onboarding` (через `computed_field` или явную передачу в `get_me`).
- `get_me`: прокинуть `onboarding_completed_at` и `needs_onboarding`.
- `complete_onboarding`: если уже set — вернуть как есть; иначе `datetime.now(UTC)`, `mark_onboarding_completed`, flush.
- Router endpoint рядом с telegram auth routes; импорт `UserService` / repo как в `/users/me`.

- [ ] **Step 4: Run tests — expect PASS**

Run: `cd backend && uv run pytest tests/unit/test_user_onboarding.py -v`
Expected: PASS

- [ ] **Step 5: Ruff**

Run: `cd backend && uv run ruff check src/markethacker/modules/users src/markethacker/modules/auth/api tests/unit/test_user_onboarding.py`

- [ ] **Step 6: Commit** (только по просьбе)

---

### Task 3: Portal — API client + onboarding storage + extension URL

**Files:**
- Create: `manager-portal/src/lib/extension-links.ts`
- Create: `manager-portal/src/lib/onboarding.ts`
- Modify: `manager-portal/src/lib/api.ts`
- Modify: `manager-portal/src/lib/nav-persistence.ts`

**Interfaces:**
- Produces:
  - `CHROME_EXTENSION_STORE_URL` — строка = backend `TELEGRAM_WELCOME_PLUGIN_URL`
  - `WB_SELLER_CABINET_URL = "https://seller.wildberries.ru"`
  - `ONBOARDING_STORAGE_KEY = "mh_onboarding_completed"`
  - `readOnboardingMirror(): string | null` — userId
  - `writeOnboardingMirror(userId: string): void`
  - `clearOnboardingMirror(): void`
  - `syncOnboardingMirror(userId: string, needsOnboarding: boolean): void` — если needs → clear; иначе write
  - `api.completeOnboarding(token: string) => Promise<{ onboardingCompletedAt: string; needsOnboarding: boolean }>`
  - `UserMe`: `onboardingCompletedAt: string | null`, `needsOnboarding: boolean`
- Consumes: Task 2 API shapes

- [ ] **Step 1: Add constants + storage helpers + api methods**

Точные URL:

```ts
// extension-links.ts
export const CHROME_EXTENSION_STORE_URL =
  "https://chromewebstore.google.com/detail/markethacker-" +
  "%D0%B8%D0%BD%D1%81%D1%82%D1%80%D1%83%D0%BC%D0%B5%D0%BD%D1%82%D1%8B/" +
  "bkklifjmnioogonadneakponmlalmnpg";

export const WB_SELLER_CABINET_URL = "https://seller.wildberries.ru";
```

`api.completeOnboarding` → `POST /auth/onboarding/complete` с token.

В `nav-persistence.ts`: добавить `/welcome` и `/complete-profile` в `BLOCKED_PATHS`, чтобы last-path не восстанавливал onboarding-экраны.

- [ ] **Step 2: Manual check sync helper**

В Node/REPL или временном assert в уме:  
`syncOnboardingMirror("u1", true)` → ключ отсутствует;  
`syncOnboardingMirror("u1", false)` → ключ `"u1"`.

- [ ] **Step 3: Lint touched files**

Run: `cd manager-portal && npm run lint -- --no-error-on-unmatched-pattern src/lib/onboarding.ts src/lib/extension-links.ts src/lib/api.ts src/lib/nav-persistence.ts`  
(или полный `npm run lint`, если флаги ESLint иные)

- [ ] **Step 4: Commit** (только по просьбе)

---

### Task 4: Portal — страница `/welcome` + redirect `/complete-profile`

**Files:**
- Create: `manager-portal/src/app/welcome/page.tsx`
- Modify: `manager-portal/src/app/complete-profile/page.tsx` → thin client redirect

**Interfaces:**
- Consumes: `api.getTelegramStatus`, `api.completeTelegramProfile`, `api.completeOnboarding`, `api` me (`/users/me`), `CHROME_EXTENSION_STORE_URL`, `WB_SELLER_CABINET_URL`, onboarding mirror helpers, `BrandLogo`, `GridBackdrop`, `Button`, `Input`, `LoadingSpinner`
- Produces: UX per spec section UI

- [ ] **Step 1: Replace `complete-profile/page.tsx` with redirect**

```tsx
"use client";
import { useEffect } from "react";
import { useRouter } from "next/navigation";

export default function CompleteProfileRedirectPage() {
  const router = useRouter();
  useEffect(() => {
    router.replace("/welcome");
  }, [router]);
  return null;
}
```

(допустим короткий spinner на `GridBackdrop`, если нужен anti-flash)

- [ ] **Step 2: Implement `/welcome`**

Логика загрузки:

1. Нет auth → `redirectToLogin()`.
2. Загрузить me + telegram status (parallel).
3. `syncOnboardingMirror(me.id, me.needsOnboarding)`.
4. Если `!needsProfileCompletion && !needsOnboarding` → `replace("/dashboard")`.
5. Имя для заголовка: первое слово `fullName` / local-part email / без имени.

UI блоки:

- Бейдж: `needsProfileCompletion` → «Почти готово»; иначе «Регистрация завершена».
- Форма email/password/confirm только при `needsProfileCompletion`; submit → `completeTelegramProfile`; локально снять флаг и показать шаги.
- Шаги disabled, пока `needsProfileCompletion`.
- Primary «Установить расширение» → `CHROME_EXTENSION_STORE_URL` (`target="_blank"` `rel="noreferrer"`).
- «Уже установил» → `completeOnboarding` + `writeOnboardingMirror` → `replace("/dashboard")`.
- Шаг 02: ссылка/кнопка на `WB_SELLER_CABINET_URL`.

Стили: как login/complete-profile (`GridBackdrop`, `rounded-2xl border border-border/80 bg-card/55`, `text-lime`).

- [ ] **Step 3: Manual verification checklist**

- `/complete-profile` → URL становится `/welcome`.
- Завершённый onboarding (мок/стейдж user) на `/welcome` → `/dashboard`.
- TG-форма: шаги не кликабельны до submit.

- [ ] **Step 4: Lint**

Run: `cd manager-portal && npm run lint`

- [ ] **Step 5: Commit** (только по просьбе)

---

### Task 5: Portal — редиректы register / login / telegram / ManagerShell

**Files:**
- Modify: `manager-portal/src/app/register/page.tsx` — после success `router.push("/welcome")` вместо `/dashboard`
- Modify: `manager-portal/src/app/login/telegram/page.tsx` — finish: всегда после tokens грузить me (или status+me); если `needsProfileCompletion || needsOnboarding` → `/welcome`, иначе `/dashboard` (не `/organizations`)
- Modify: `manager-portal/src/app/login/page.tsx` — в `finishLogin` после `setTokens`: если не invite flow, вызвать me; при `needsOnboarding` → `/welcome`, иначе `resolveReturnPath(from)` (invite по-прежнему `/organizations`)
- Modify: `manager-portal/src/components/manager-shell.tsx` — гейт: `needsProfileCompletion || needsOnboarding` → `replace("/welcome")`; читать me (или status+me). Не гейтить сам `/welcome` через shell (welcome вне `(manager)` layout).

**Interfaces:**
- Consumes: Task 2/3 `needsOnboarding`, `needsProfileCompletion`
- Produces: единый post-auth entry на welcome при незавершённом onboarding

- [ ] **Step 1: Wire register → `/welcome`**

Заменить `router.push("/dashboard")` на `router.push("/welcome")`.

- [ ] **Step 2: Wire telegram login finish**

После exchange: если `needsProfileCompletion` → `/welcome`.  
Иначе запросить me; если `needsOnboarding` → `/welcome`; иначе `/dashboard`.

- [ ] **Step 3: Wire email login finish**

После tokens (не invite): me → `needsOnboarding` ? `/welcome` : `resolveReturnPath(from)`.

- [ ] **Step 4: Wire ManagerShell gate**

Расширить существующий effect: помимо telegram status, учесть `needsOnboarding` из me (один запрос me может покрыть `needsOnboarding`; status всё ещё нужен для `needsProfileCompletion`, либо me+status parallel).  
Если любой флаг true → `replace("/welcome")` и не ставить `profileReady` для children shell.

- [ ] **Step 5: Manual verification**

| Сценарий | Ожидание |
|---|---|
| Новая регистрация сайта | `/welcome`, без формы, шаги активны |
| TG новый | `/welcome` с формой → после save шаги → «Уже установил» → `/dashboard` |
| Старый пользователь (backfill) | login → не `/welcome` |
| Incomplete onboarding, повторный login | снова `/welcome` |

- [ ] **Step 6: Lint**

Run: `cd manager-portal && npm run lint`

- [ ] **Step 7: Commit** (только по просьбе)

---

### Task 6: Документация

**Files:**
- Modify: `docs/integrations/telegram-auth-client.md`
- Modify: `docs/architecture/telegram-bot.md`

- [ ] **Step 1: Update telegram-auth-client.md**

В секции дозаполнения:

- После `needsProfileCompletion === true` Manager Portal открывает **`/welcome`** (форма на той же странице; `/complete-profile` — legacy redirect).
- Добавить краткий блок: `needsOnboarding` из `GET /users/me`; завершение `POST /auth/onboarding/complete`; после успеха — `/dashboard`.
- Extension: по-прежнему может показывать свой экран complete-profile API; путь `/welcome` — portal-specific.

- [ ] **Step 2: Note in telegram-bot.md**

После абзаца про `needsProfileCompletion`: клиент Manager Portal после входа/дозаполнения показывает web onboarding `/welcome`, пока `onboarding_completed_at` пуст. Ссылка на spec.

- [ ] **Step 3: Commit** (только по просьбе)

---

## Spec coverage checklist

| Spec requirement | Task |
|---|---|
| `onboarding_completed_at` + backfill | 1 |
| me fields + `POST /auth/onboarding/complete` | 2 |
| localStorage mirror | 3 |
| `/welcome` UI + form gate + steps | 4 |
| `/complete-profile` redirect | 4 |
| register / TG / login / shell redirects | 5 |
| exit `/dashboard` | 4, 5 |
| Chrome + WB URLs | 3, 4 |
| Docs | 6 |
| No demo link / no Ozon CTA | 4 (не добавлять) |
