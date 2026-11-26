# Measurement Disputes Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Инструмент оспаривания обмеров WB в TEAM: локальный разбор XLSX, первое обращение без ИИ, async jobs разбора ответа и фотоанализа с Мурликами, админ-настройки и веса; код расширения не меняем — только интеграционная дока.

**Architecture:** Клиентский парсер и шаблоны в `manager-portal`; backend-модуль `measurement_dispute` по паттерну `card_audit` (ORM job + settings + ARQ + OpenRouter + reserve/commit Мурликов); admin UI для модели/`rule_context` и существующая страница весов Мурликов.

**Tech Stack:** FastAPI / SQLAlchemy / Alembic / ARQ / OpenRouter / Pydantic; manager-portal Next.js + `xlsx`; admin-panel Next.js.

**Spec:** `docs/superpowers/specs/2026-11-26-measurement-disputes-design.md`  
**TZ:** `tz_osparivanie.md`

## Global Constraints

- Страница только в manager-portal: `/measurement-disputes` (без `wb-` в path).
- `extension-chrome` не трогать; интеграция — `docs/integrations/measurement-disputes-extension.md`.
- XLSX не управляется на сервер и не в LLM; на API — структурированный JSON / URL фото.
- LLM только через OpenRouter (`OPENROUTER_API_KEY` на backend); ключ не в клиенте.
- Feature key: `measurement_dispute` (`user_or_org_seat`).
- Weights defaults: `measurement_dispute_analyze_response/base = 5`; `measurement_dispute_photo_analysis/per_measurement = 3` (админ может переопределить; `MIN_WEIGHT` не понижаем).
- UI без внутренних имён API/моделей/SQL в пользовательских текстах.
- Нет `any` / `Record<string, any>`; явные `interface`/`type`.
- Коммитить только по явной просьбе пользователя.
- Backend tests: `cd backend && uv run pytest <path> -v`.
- Backend lint: `cd backend && uv run ruff check <paths>`.
- Portal: следовать паттернам `card-audits` (PageHeader, Alert paywall, `useApi`/`useMutation`/`withAuth`).
- Перед production: сверить `rule_context` с актуальной офертой WB (§21 ТЗ).

## Resolved open points (from spec)

1. Default weights: **5** (text) / **3** (per photo measurement).
2. Text-анализ платный (`base ≥ 1`); бесплатный text не делаем в v1.
3. `GET /active` реализовать; подключение к FAB активных AI-jobs — Task 12 (если хук сопоставим с card-audit; иначе оставить endpoint + list/detail polling).

## File map

### Create — backend

- `backend/src/markethacker/modules/measurement_dispute/__init__.py`
- `backend/src/markethacker/modules/measurement_dispute/domain/__init__.py`
- `backend/src/markethacker/modules/measurement_dispute/domain/models.py`
- `backend/src/markethacker/modules/measurement_dispute/domain/settings.py`
- `backend/src/markethacker/modules/measurement_dispute/domain/result_schemas.py`
- `backend/src/markethacker/modules/measurement_dispute/domain/pii.py`
- `backend/src/markethacker/modules/measurement_dispute/application/__init__.py`
- `backend/src/markethacker/modules/measurement_dispute/application/quota_service.py`
- `backend/src/markethacker/modules/measurement_dispute/application/settings_service.py`
- `backend/src/markethacker/modules/measurement_dispute/application/service.py`
- `backend/src/markethacker/modules/measurement_dispute/infrastructure/__init__.py`
- `backend/src/markethacker/modules/measurement_dispute/infrastructure/repository.py`
- `backend/src/markethacker/modules/measurement_dispute/infrastructure/llm/__init__.py`
- `backend/src/markethacker/modules/measurement_dispute/infrastructure/llm/prompts.py`
- `backend/src/markethacker/modules/measurement_dispute/infrastructure/llm/parser.py`
- `backend/src/markethacker/modules/measurement_dispute/infrastructure/llm/pipeline.py`
- `backend/src/markethacker/modules/measurement_dispute/infrastructure/jobs/__init__.py`
- `backend/src/markethacker/modules/measurement_dispute/infrastructure/jobs/run_dispute.py`
- `backend/src/markethacker/modules/measurement_dispute/api/__init__.py`
- `backend/src/markethacker/modules/measurement_dispute/api/schemas.py`
- `backend/src/markethacker/modules/measurement_dispute/api/router.py`
- `backend/src/markethacker/modules/measurement_dispute/api/admin_router.py`
- `backend/alembic/versions/20261126_0065_measurement_disputes.py`
- `backend/tests/unit/measurement_dispute/test_result_schemas.py`
- `backend/tests/unit/measurement_dispute/test_pii.py`
- `backend/tests/unit/measurement_dispute/test_compute_cost_branches.py` (или расширить billing test)
- `backend/tests/integration/test_measurement_disputes_api.py`

### Create — manager-portal

- `manager-portal/src/lib/measurement-dispute/types.ts`
- `manager-portal/src/lib/measurement-dispute/parse-money.ts`
- `manager-portal/src/lib/measurement-dispute/parse-report.ts`
- `manager-portal/src/lib/measurement-dispute/group-measurements.ts`
- `manager-portal/src/lib/measurement-dispute/deadline.ts`
- `manager-portal/src/lib/measurement-dispute/first-appeal.ts`
- `manager-portal/src/lib/measurement-dispute/manual-retry-template.ts`
- `manager-portal/src/lib/measurement-dispute/demo-fixture.ts`
- `manager-portal/src/lib/measurement-dispute/analytics.ts`
- `manager-portal/src/app/(manager)/measurement-disputes/page.tsx`
- `manager-portal/src/app/(manager)/measurement-disputes/new/page.tsx`
- `manager-portal/src/app/(manager)/measurement-disputes/[id]/page.tsx`
- Unit tests рядом с lib (vitest/jest — использовать тот же runner, что уже в portal; если тестов нет — `*.test.ts` под существующий `package.json` scripts)

### Create — admin-panel

- `admin-panel/src/app/(admin)/measurement-dispute/settings/page.tsx`
- `admin-panel/src/components/measurement-dispute-nav.tsx` (опционально, если нужен nav как у card-audit)

### Create — docs

- `docs/architecture/measurement-disputes.md`
- `docs/integrations/measurement-disputes-extension.md`

### Modify

- `backend/src/markethacker/modules/billing/domain/features.py`
- `backend/src/markethacker/modules/billing/domain/mh_credit_weights.py`
- `backend/src/markethacker/modules/billing/application/mh_credit_weights_service.py` (если `compute_action_cost` пробрасывает kwargs)
- `backend/tests/unit/billing/test_mh_credit_weights.py`
- `backend/src/markethacker/infrastructure/database/all_models.py`
- `backend/src/markethacker/infrastructure/jobs/worker.py`
- `backend/src/markethacker/main.py`
- `manager-portal/package.json` (добавить `xlsx` если ещё нет)
- `manager-portal/src/lib/plan-features.ts`
- `manager-portal/src/components/sidebar.tsx`
- `manager-portal/src/lib/api.ts` (типы MeasurementDispute*)
- `manager-portal/src/components/reviews-analysis-active-widget.tsx` (Task 12, optional FAB)
- `admin-panel/src/lib/api.ts`
- `admin-panel/src/components/sidebar.tsx`
- `docs/architecture/billing.md` (строка про новые веса)
- `docs/architecture/README.md` (ссылка на новый doc)
- `docs/operations/murliki-unit-economics.md` (строка про новые действия)
- `docs/superpowers/specs/2026-11-26-measurement-disputes-design.md` (статус уже approved)

---

### Task 1: Feature flag + веса Мурликов

**Files:**
- Modify: `backend/src/markethacker/modules/billing/domain/features.py`
- Modify: `backend/src/markethacker/modules/billing/domain/mh_credit_weights.py`
- Modify: `backend/src/markethacker/modules/billing/application/mh_credit_weights_service.py` (только если сигнатура `compute_action_cost` / `compute_cost` нуждается в `measurement_count`)
- Modify: `backend/tests/unit/billing/test_mh_credit_weights.py`

**Interfaces:**
- Produces:
  - `MEASUREMENT_DISPUTE = "measurement_dispute"` в `FEATURE_CATALOG`
  - `ACTION_MEASUREMENT_DISPUTE_ANALYZE_RESPONSE = "measurement_dispute_analyze_response"`
  - `ACTION_MEASUREMENT_DISPUTE_PHOTO_ANALYSIS = "measurement_dispute_photo_analysis"`
  - `PARAM_PER_MEASUREMENT = "per_measurement"`
  - `compute_cost(...)` ветки для обоих action_key

- [ ] **Step 1: Write failing tests for new costs**

В `test_mh_credit_weights.py` добавить:

```python
from markethacker.modules.billing.domain.mh_credit_weights import (
    ACTION_MEASUREMENT_DISPUTE_ANALYZE_RESPONSE,
    ACTION_MEASUREMENT_DISPUTE_PHOTO_ANALYSIS,
)

def test_measurement_dispute_analyze_response_cost() -> None:
    assert compute_cost(ACTION_MEASUREMENT_DISPUTE_ANALYZE_RESPONSE) == 5

def test_measurement_dispute_photo_analysis_cost() -> None:
    assert compute_cost(ACTION_MEASUREMENT_DISPUTE_PHOTO_ANALYSIS, measurement_count=1) == 3
    assert compute_cost(ACTION_MEASUREMENT_DISPUTE_PHOTO_ANALYSIS, measurement_count=10) == 30
    assert compute_cost(ACTION_MEASUREMENT_DISPUTE_PHOTO_ANALYSIS, measurement_count=0) == 3  # min 1
```

- [ ] **Step 2: Run tests — expect fail**

```bash
cd backend && uv run pytest tests/unit/billing/test_mh_credit_weights.py -v
```

Expected: FAIL (unknown action / import error).

- [ ] **Step 3: Add feature to catalog**

В `features.py` после `CARD_GENERATE`:

```python
MEASUREMENT_DISPUTE = "measurement_dispute"
```

и `FeatureInfo` в `FEATURE_CATALOG`:

```python
FeatureInfo(
    key=MEASUREMENT_DISPUTE,
    label="Оспаривание обмеров WB",
    description=(
        "Разбор отчёта по удержаниям, черновики обращений и ИИ-разбор "
        "ответов/фото обмеров Wildberries. "
        "Участники организации наследуют фичу с тарифа владельца."
    ),
    grant_scope=GRANT_SCOPE_USER_OR_ORG_SEAT,
),
```

- [ ] **Step 4: Extend weights catalog + compute_cost**

В `mh_credit_weights.py`:

```python
ACTION_MEASUREMENT_DISPUTE_ANALYZE_RESPONSE = "measurement_dispute_analyze_response"
ACTION_MEASUREMENT_DISPUTE_PHOTO_ANALYSIS = "measurement_dispute_photo_analysis"
PARAM_PER_MEASUREMENT = "per_measurement"
```

Добавить в `WEIGHT_CATALOG`:

```python
WeightDefinition(
    action_key=ACTION_MEASUREMENT_DISPUTE_ANALYZE_RESPONSE,
    param_key=PARAM_BASE,
    default_weight=5,
    label="Оспаривание обмеров — разбор ответа WB",
    description="Списание за один job разбора ответа поддержки.",
),
WeightDefinition(
    action_key=ACTION_MEASUREMENT_DISPUTE_PHOTO_ANALYSIS,
    param_key=PARAM_PER_MEASUREMENT,
    default_weight=3,
    label="Оспаривание обмеров — фотоанализ за обмер",
    description="Списание за один уникальный обмер с фото в job фотоанализа.",
),
```

В `compute_cost` добавить параметр `measurement_count: int = 1` и ветки:

```python
if action_key == ACTION_MEASUREMENT_DISPUTE_ANALYZE_RESPONSE:
    return resolve_weight(
        ACTION_MEASUREMENT_DISPUTE_ANALYZE_RESPONSE, PARAM_BASE, overrides=overrides
    )

if action_key == ACTION_MEASUREMENT_DISPUTE_PHOTO_ANALYSIS:
    count = max(1, int(measurement_count))
    return (
        resolve_weight(
            ACTION_MEASUREMENT_DISPUTE_PHOTO_ANALYSIS,
            PARAM_PER_MEASUREMENT,
            overrides=overrides,
        )
        * count
    )
```

Прокинуть `measurement_count` в `MhCreditWeightsService.compute_action_cost`, если он тонкой обёрткой над `compute_cost`.

- [ ] **Step 5: Re-run tests — expect pass**

```bash
cd backend && uv run pytest tests/unit/billing/test_mh_credit_weights.py -v
```

- [ ] **Step 6: Commit** (только если пользователь попросил)

```bash
git add backend/src/markethacker/modules/billing/domain/features.py \
  backend/src/markethacker/modules/billing/domain/mh_credit_weights.py \
  backend/src/markethacker/modules/billing/application/mh_credit_weights_service.py \
  backend/tests/unit/billing/test_mh_credit_weights.py
git commit -m "$(cat <<'EOF'
feat(billing): add measurement dispute feature and MH credit weights

EOF
)"
```

---

### Task 2: ORM jobs + settings + migration

**Files:**
- Create: `backend/src/markethacker/modules/measurement_dispute/domain/models.py`
- Create: `backend/src/markethacker/modules/measurement_dispute/domain/settings.py`
- Create: `backend/alembic/versions/20261126_0065_measurement_disputes.py`
- Modify: `backend/src/markethacker/infrastructure/database/all_models.py`

**Interfaces:**
- Produces:
  - `MeasurementDispute` ORM → table `measurement_disputes`
  - `MeasurementDisputeSettings` singleton → `measurement_dispute_settings`
  - Constants: `KIND_ANALYZE_RESPONSE`, `KIND_PHOTO_ANALYSIS`, status literals
  - Defaults: `DEFAULT_ANALYZE_LLM_MODEL = "openai/gpt-5.4"`, `DEFAULT_PHOTO_LLM_MODEL` (vision-capable OpenRouter slug, например `openai/gpt-5.4` или актуальный vision; зафиксировать при реализации по OpenRouter docs), `DEFAULT_PROMPT_VERSION = "v1"`, `wb_response_max_chars=8000`, `max_bulk_photos=50`

- [ ] **Step 1: Create settings ORM** (mirror `CardAuditSettings`, расширить полями из спеки: `llm_model`, `photo_llm_model`, `prompt_version`, `photo_prompt_version`, `wb_response_max_chars`, `max_bulk_photos`, `rule_context`, `rule_source_url`, `analyze_response_enabled`, `photo_analysis_enabled`, `updated_at`, `updated_by`).

- [ ] **Step 2: Create job ORM**

Поля из спеки: `id`, `user_id`, `organization_id` (nullable), `kind`, `status`, `request_payload` (JSONB), `result_payload` (JSONB nullable), `prompt_version`, `model`, `usage_input_tokens`, `usage_output_tokens`, `quota_reserved`, `quota_units`, `error_code`, `error_message`, `created_at`, `started_at`, `finished_at`. Индексы: `(user_id, created_at desc)`, `(status)` для active.

- [ ] **Step 3: Alembic migration**

`revision = "20261126_0065"`, `down_revision = "20261123_0064"`. Создать обе таблицы; seed settings row `id=1` с дефолтами.

- [ ] **Step 4: Register in `all_models.py`**

```python
from markethacker.modules.measurement_dispute.domain.models import MeasurementDispute  # noqa: F401
from markethacker.modules.measurement_dispute.domain.settings import MeasurementDisputeSettings  # noqa: F401
```

- [ ] **Step 5: Apply migration locally**

```bash
cd backend && uv run alembic upgrade head
```

Expected: OK, tables exist.

- [ ] **Step 6: Commit** (по просьбе)

---

### Task 3: Result schemas + PII scrub + unit tests

**Files:**
- Create: `backend/src/markethacker/modules/measurement_dispute/domain/result_schemas.py`
- Create: `backend/src/markethacker/modules/measurement_dispute/domain/pii.py`
- Create: `backend/tests/unit/measurement_dispute/test_result_schemas.py`
- Create: `backend/tests/unit/measurement_dispute/test_pii.py`

**Interfaces:**
- Produces:
  - `AnalyzeResponseResult` Pydantic (поля §11 ТЗ; `category` Literal)
  - `PhotoVerdict` Literal: `appeal|borderline|no_reason|insufficient`
  - `PhotoAnalysisItemResult`, `PhotoAnalysisResult` (items + aggregates)
  - `scrub_pii(text: str) -> str`
  - `ANALYZE_RESPONSE_CATEGORIES`

- [ ] **Step 1: Failing tests for schema validation**

```python
def test_analyze_response_rejects_unknown_category() -> None:
    with pytest.raises(ValidationError):
        AnalyzeResponseResult.model_validate({..., "category": "NOPE"})

def test_analyze_response_accepts_no_stock() -> None:
    result = AnalyzeResponseResult.model_validate({...valid NO_STOCK payload...})
    assert result.category == "NO_STOCK"
```

- [ ] **Step 2: Failing tests for PII**

```python
def test_scrub_pii_removes_email_phone() -> None:
    text = scrub_pii("Пишите на a@b.ru или +7 (999) 123-45-67")
    assert "@" not in text
    assert "999" not in text or "[УДАЛЕНО]" in text
```

- [ ] **Step 3: Implement schemas + scrub (regex email/phone/token-like)**

- [ ] **Step 4: Run unit tests**

```bash
cd backend && uv run pytest tests/unit/measurement_dispute/ -v
```

---

### Task 4: Quota service + application service (create/estimate/cancel)

**Files:**
- Create: `.../application/quota_service.py`
- Create: `.../application/settings_service.py`
- Create: `.../application/service.py`
- Create: `.../infrastructure/repository.py`
- Create: `.../api/schemas.py` (request/response DTOs; ещё без router wire, либо вместе с Task 6)

**Interfaces:**
- Consumes: feature `MEASUREMENT_DISPUTE`, weights actions, ORM
- Produces:
  - `MeasurementDisputeQuotaService.compute_cost(kind, measurement_count=1) -> int`
  - `resolve/reserve/commit/release` как `CardAuditQuotaService`
  - `MeasurementDisputeService.estimate_cost(body) -> {units}`
  - `MeasurementDisputeService.create(user_id, body) -> MeasurementDispute` (validate kind enabled, bulk ≤ max, scrub text, reserve, enqueue)
  - `list/get/get_result/cancel/list_active`

Validation rules on create:

- `kind=analyze_response`: non-empty `wb_response_text` after strip; truncate to `wb_response_max_chars`; attach `rule_context` from settings if missing.
- `kind=photo_analysis`: each item has photo URL + positive net + discrepancy; if `len(items) > max_bulk_photos` → `ValidationError` with user-safe message; single-item always allowed even when file had >50 (caller sends one).

- [ ] **Step 1: Implement repository** (CRUD by id/user, list pagination, active statuses).

- [ ] **Step 2: Implement settings_service** (`get_view` / `update` singleton).

- [ ] **Step 3: Implement quota_service**.

- [ ] **Step 4: Implement service.create** with reserve; enqueue stub that raises until Task 5 — или enqueue real job name `run_measurement_dispute` already, worker added next task.

- [ ] **Step 5: Unit-test cost selection via quota_service with mocked weights if feasible; otherwise cover in integration Task 7.

---

### Task 5: LLM prompts/pipeline + ARQ worker

**Files:**
- Create: `.../infrastructure/llm/prompts.py`
- Create: `.../infrastructure/llm/parser.py`
- Create: `.../infrastructure/llm/pipeline.py`
- Create: `.../infrastructure/jobs/run_dispute.py`
- Modify: `backend/src/markethacker/infrastructure/jobs/worker.py`

**Interfaces:**
- Consumes: `OpenRouterLlmClient` из `modules/reviews_analysis/infrastructure/llm/client.py`
- Produces:
  - `build_analyze_system_prompt() -> str` (текст §9 ТЗ)
  - `build_analyze_user_prompt(input_json, rule_context) -> str` (§10)
  - `build_photo_*` prompts (§22: overlay digits, Top/Side, no perspective measuring, verdicts)
  - `MeasurementDisputeLlmPipeline.run_analyze` / `run_photo`
  - Parser: extract JSON → Pydantic; on failure raise; pipeline does **one** schema-repair retry with error hint
  - `async def run_measurement_dispute(ctx, dispute_id: str) -> None` — load job, set running, call pipeline, save result + usage, commit/release

Worker registration (mirror card_audit):

```python
from markethacker.modules.measurement_dispute.infrastructure.jobs.run_dispute import (
    run_measurement_dispute,
)
_run_measurement_dispute = func(run_measurement_dispute, name="run_measurement_dispute", timeout=1800, max_tries=2)
# add to functions list
```

Logging: `request_id` (= job id), `category` (if analyze), `latency_ms`, `status`, `prompt_version` — **не** логировать полный WB text и размеры.

- [ ] **Step 1: Implement prompts + parser + pipeline**

- [ ] **Step 2: Implement job runner with commit/release**

- [ ] **Step 3: Register in worker.py**

- [ ] **Step 4: Unit test parser with fixture JSON strings (valid + garbage → retry path mock)**

---

### Task 6: Public + admin API routers + main.py

**Files:**
- Create: `.../api/router.py` (`prefix="/measurement-disputes"`)
- Create: `.../api/admin_router.py` (`prefix="/admin/measurement-dispute"`)
- Create/finish: `.../api/schemas.py`
- Modify: `backend/src/markethacker/main.py`

**Endpoints:** как в спеке. Auth: `get_current_user_id`. Errors → существующие exception handlers.

Admin: `GET/PATCH /settings` + `require_superuser`.

- [ ] **Step 1: Implement schemas (camelCase API как в других модулях — проверить card_audit response alias style и повторить)**

- [ ] **Step 2: Implement routers**

- [ ] **Step 3: Wire in main.py** рядом с card_audit

- [ ] **Step 4: Smoke with OpenAPI / local curl against running API** (если окружение доступно)

---

### Task 7: Backend integration + failure tests

**Files:**
- Create: `backend/tests/integration/test_measurement_disputes_api.py`

**Cases:**

1. User without feature → 403 on create.
2. estimate-cost photo `measurement_count=10` → units `30` (defaults).
3. create analyze_response → 202, status queued, quota reserved.
4. Mock LLM success → worker → succeeded + result schema + credits committed.
5. Mock invalid JSON twice → failed + credits released.
6. photo bulk > max_bulk_photos → 422.
7. cancel queued → released.

Follow existing integration test fixtures (auth user, DB, ARQ mock) from card_audit/reviews if present; иначе минимальный pattern из `tests/integration/`.

```bash
cd backend && uv run pytest tests/integration/test_measurement_disputes_api.py -v
```

---

### Task 8: Admin-panel settings UI

**Files:**
- Modify: `admin-panel/src/lib/api.ts` — types `MeasurementDisputeSettings`, `UpdateMeasurementDisputeSettings`
- Modify: `admin-panel/src/components/sidebar.tsx` — пункт «Оспаривание обмеров» / settings
- Create: `admin-panel/src/app/(admin)/measurement-dispute/settings/page.tsx`

**UI fields:** llm_model, photo_llm_model, prompt versions, wb_response_max_chars, max_bulk_photos, rule_context (textarea), rule_source_url, toggles kinds. Copy: без секретов; «Ключ OpenRouter задаётся на сервере».

Веса — уже на `/billing/mh-credit-weights` (появятся из каталога автоматически после Task 1). Проверить вручную, что новые строки видны.

- [ ] **Step 1: Types + page + sidebar**

- [ ] **Step 2: Manual check load/save**

- [ ] **Step 3: Update `docs/operations/murliki-unit-economics.md` + `docs/architecture/billing.md` одной строкой про новые действия**

---

### Task 9: Portal lib — parse money, report, group, fixtures

**Files:**
- Create: `manager-portal/src/lib/measurement-dispute/types.ts`
- Create: `manager-portal/src/lib/measurement-dispute/parse-money.ts`
- Create: `manager-portal/src/lib/measurement-dispute/parse-report.ts`
- Create: `manager-portal/src/lib/measurement-dispute/group-measurements.ts`
- Create: `manager-portal/src/lib/measurement-dispute/demo-fixture.ts`
- Create: `manager-portal/src/lib/measurement-dispute/*.test.ts` (если в portal настроен vitest; иначе backend-like чистые функции тестировать через `node:test` / добавить vitest только если уже есть — **не** вводить новый test runner без необходимости; проверить `manager-portal/package.json` scripts)
- Modify: `manager-portal/package.json` — dependency `xlsx` (как в market-navigators `^0.18.5`)

**Interfaces:**
- `parseMoney(raw: string | number | null | undefined): { amount: number | null; warning: boolean }`
- `parseWorkbook(file: ArrayBuffer): ParsedWorkbook`
- `selectHeaderRow(rows): HeaderMapping` (+ manual override API)
- `groupMeasurements(rows): MeasurementGroup[]` — key `nomenclature + measurementNumber`
- `summarizeSheet(groups): FileTotals` — operations, uniqueMeasurements, articles, accrued, storno, net
- Demo fixture totals: `{ discrepancies: 3, articles: 2, total: 3400 }`
- Real-profile fixture builder (inline table) totals: operations 153, measurements 12, articles 10, accrued 165804.5, storno -40401.2, net 125403.3

- [ ] **Step 1: Unit tests parseMoney** (`"1 250 ₽"` → 1250; `"-640,50"` → -640.5; empty/non-numeric → null + warning)

- [ ] **Step 2: Implement parseMoney**

- [ ] **Step 3: Unit tests unique articles + deadline later in Task 10; here group/summarize fixtures**

- [ ] **Step 4: Implement parse-report + group-measurements + demo-fixture**

- [ ] **Step 5: Install xlsx and verify import**

```bash
cd manager-portal && npm install xlsx@0.18.5
```

---

### Task 10: Portal lib — deadline, first appeal, manual template, analytics

**Files:**
- Create: `deadline.ts`, `first-appeal.ts`, `manual-retry-template.ts`, `analytics.ts`

**Interfaces:**
- `addCalendarDays(isoDate: string, days: number): string` — publication + 7; month-end cases
- `buildFirstAppeal(input: FirstAppealInput): string` — §7 + §22 multi mode
- `buildManualRetryTemplate(input): string` — безопасный шаблон при fail ИИ
- `trackMeasurementDisputeEvent(name, params?)` → `reachYandexMetrikaGoal` **без** текста обращений и значений XLSX

Events (goal names):  
`measurement_tool_opened`, `xlsx_parsed`, `first_draft_created`, `wb_response_analysis_started`, `wb_response_analysis_succeeded`, `wb_response_analysis_failed`, `photo_analysis_started|succeeded|failed`, `short_reply_copied`, `detailed_claim_copied`

- [ ] **Step 1: Tests for +7 days including end of month**

- [ ] **Step 2: Implement deadline + first appeal (placeholders `[УКАЖИТЕ …]`)**

- [ ] **Step 3: Manual template + analytics wrapper**

---

### Task 11: Portal pages list / new / [id] + sidebar

**Files:**
- Modify: `manager-portal/src/lib/plan-features.ts` — `PLAN_FEATURE_MEASUREMENT_DISPUTE = "measurement_dispute"`
- Modify: `manager-portal/src/components/sidebar.tsx` — пункт «Оспаривание обмеров»
- Modify: `manager-portal/src/lib/api.ts` — types for create/estimate/summary/result
- Create: three pages under `measurement-disputes/`

**List page:** paywall Alert → `/billing`; table of jobs; CTA «Новый разбор».

**New page (core UX):**  
1. Upload XLSX or «Демо-файл»  
2. Sheet select + totals block  
3. Measurement selection (single vs multi claim checkboxes; default active positive net)  
4. Form: seller dimensions, report publication date (required confirm), stock, supply date  
5. First appeal preview + copy + attachments checklist  
6. WB response textarea + appeal number → privacy notice → estimate → create `analyze_response` job  
7. Photo analysis: select active with photos → show count × tariff estimate → create `photo_analysis` job  
8. Redirect to `[id]`

**Detail page:** poll `GET /{id}` until terminal; show result cards; copy buttons; on fail show manual template + «Повторить» (new create with same client-held payload if still in sessionStorage key `mh_measurement_dispute_draft` — optional; минимум: кнопка назад на `/new`).

User-facing errors only («Не удалось разобрать ответ. Данные сохранены — можно повторить.»), детали в console/logs.

- [ ] **Step 1: plan-features + sidebar + api types**

- [ ] **Step 2: list page**

- [ ] **Step 3: new page wired to local libs + API**

- [ ] **Step 4: detail page polling + results UI (§13)**

- [ ] **Step 5: Manual QA checklist against §18/§22 acceptance (portal parts)**

---

### Task 12: Active jobs FAB hook (optional-in-scope)

**Files:**
- Modify: `manager-portal/src/components/reviews-analysis-active-widget.tsx` (или общий active-jobs widget)

- [ ] **Step 1: If pattern is a small addition** — add `kind: "measurement_dispute"` fetch from `/measurement-disputes/active` and link to `/measurement-disputes/{id}`.

- [ ] **Step 2: If invasive** — skip UI wiring; keep API `/active`; document in architecture doc as follow-up. Do not block release.

---

### Task 13: Documentation

**Files:**
- Create: `docs/architecture/measurement-disputes.md`
- Create: `docs/integrations/measurement-disputes-extension.md`
- Modify: `docs/architecture/README.md`

**Architecture doc:** назначение, flow, API table, billing keys, settings, privacy, links to TZ/spec.

**Extension integration doc (variant C):**  
полный контракт API; auth headers; UX states §14; modal checklist §2; local XLSX rules; billing estimate/create; analytics event names; acceptance checklist for extension E2E (выполняет другой разработчик). Явно: «код расширения в этой поставке не меняется».

- [ ] **Step 1: Write both docs from final API schemas (copy field lists from `api/schemas.py`)**

- [ ] **Step 2: Link from architecture README**

---

### Task 14: Spec coverage self-check + release notes for ops

- [ ] **Step 1: Walk TZ §1–§22 and spec In-scope — each item maps to a task above**

Checklist mapping:

| Requirement | Task |
|---|---|
| Local XLSX parse + aliases + manual column | 9 |
| File totals + real grouping/storno | 9 |
| First appeal deterministic + multi | 10–11 |
| Deadline +7 | 10 |
| Async analyze_response | 4–7 |
| Categories + schema + retry | 3, 5 |
| Photo analysis + bulk cap + Мурлики | 1, 4–7, 11 |
| Admin settings + weights UI | 1, 8 |
| Portal pages + feature gate | 11 |
| Privacy / no XLSX upload | 4, 5, 11 |
| Extension docs only | 13 |
| Tests unit/integration/failure | 1, 3, 7, 9, 10 |

- [ ] **Step 2: Ops** — enable `measurement_dispute` on target billing plans in admin; set `rule_context` from current WB sources (§21); tune weights if needed.

- [ ] **Step 3: Final verification commands**

```bash
cd backend && uv run pytest tests/unit/billing/test_mh_credit_weights.py tests/unit/measurement_dispute tests/integration/test_measurement_disputes_api.py -v
cd backend && uv run ruff check src/markethacker/modules/measurement_dispute src/markethacker/modules/billing/domain/features.py src/markethacker/modules/billing/domain/mh_credit_weights.py
cd manager-portal && npm run lint
cd admin-panel && npm run lint
```

---

## Self-review (plan vs spec)

1. **Spec coverage:** Portal + backend + admin + docs + billing covered; extension code explicitly out; FAB optional Task 12.
2. **Placeholders:** Defaults for weights/models locked; no TBD left for implementer except choosing exact vision model slug at Task 2/5 against current OpenRouter catalog.
3. **Type consistency:** `kind` values `analyze_response` | `photo_analysis`; feature `measurement_dispute`; routes `/measurement-disputes`; admin `/measurement-dispute/settings`; actions `measurement_dispute_analyze_response` / `measurement_dispute_photo_analysis`.

## Execution handoff

Plan saved to `docs/superpowers/plans/2026-11-26-measurement-disputes.md`.

**Two execution options:**

1. **Subagent-Driven (recommended)** — fresh subagent per task, review between tasks  
2. **Inline Execution** — execute in this session with executing-plans checkpoints  

Which approach?
