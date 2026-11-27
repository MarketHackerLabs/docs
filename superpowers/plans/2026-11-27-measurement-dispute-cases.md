# Measurement Dispute Cases Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Список и виджет показывают одно оспаривание (кейс по отчёту); анализы фото и ответа WB — дочерние ИИ-jobs внутри кейса; открытие из списка ведёт в workspace этого кейса.

**Architecture:** Новая таблица `measurement_dispute_cases`; существующие `measurement_disputes` получают обязательный `case_id`. API переносится на `/cases` и `/jobs`. Portal: list → `{caseId}` workspace; `/new` создаёт кейс и redirect. Старые jobs удаляются с release висящих резервов Мурликов.

**Tech Stack:** FastAPI / SQLAlchemy / Alembic / ARQ; manager-portal Next.js; существующий модуль `measurement_dispute`.

**Spec:** `docs/superpowers/specs/2026-11-27-measurement-dispute-cases-design.md`

## Global Constraints

- XLSX не на сервер; на кейсе только шапка + `reportSummary` (`FileTotals`).
- Веса Мурликов и LLM pipeline не менять.
- Старые list/create/`/{id}` job-как-список эндпоинты удалить (breaking); обновить extension-доку.
- Старые строки `measurement_disputes` удалить полностью; перед удалением — `release` при `quota_reserved = true`.
- DELETE кейса вне v1.
- UI без внутренних имён API/SQL в пользовательских текстах.
- Нет `any` / `Record<string, any>`; явные `interface`/`type`.
- Коммитить только по явной просьбе пользователя (отдельные репо: `backend`, `manager-portal`, `docs`).
- Backend tests: `cd backend && uv run pytest <path> -v`.
- Backend lint: `cd backend && uv run ruff check <paths>`.
- Portal check: `cd manager-portal && npx tsc --noEmit`.

## Review Focus

- Job с `quota_reserved=true` при миграции: после upgrade резерв снят, jobs пусты.
- Чужой `caseId` / job другого пользователя → `404`, не `403`.
- Два kind под одним case → в `GET /cases` одна строка со `status` по правилу spec.
- Открытие `/measurement-disputes/{caseId}` без localStorage → шапка+jobs, без редиректа на `/new`.
- Повторный анализ создаёт новый job, не затирает историю в `jobs[]`.

---

## File map

### Backend — create / modify

- Create: `backend/alembic/versions/20261127_0066_measurement_dispute_cases.py`
- Modify: `backend/src/markethacker/modules/measurement_dispute/domain/models.py` — `MeasurementDisputeCase`; `case_id` на `MeasurementDispute`
- Create: `backend/src/markethacker/modules/measurement_dispute/domain/case_status.py` — `compute_case_status(jobs) -> Literal[...]`
- Modify: `backend/src/markethacker/modules/measurement_dispute/infrastructure/repository.py` — case CRUD + list active cases + jobs by case
- Modify: `backend/src/markethacker/modules/measurement_dispute/application/dto.py` — `CreateMeasurementDisputeCaseRequest`, `ReportSummaryDto`
- Modify: `backend/src/markethacker/modules/measurement_dispute/application/service.py` — case + job-under-case
- Modify: `backend/src/markethacker/modules/measurement_dispute/api/schemas.py` — case response schemas
- Modify: `backend/src/markethacker/modules/measurement_dispute/api/router.py` — новые маршруты, удалить старые
- Modify: `backend/tests/integration/test_measurement_disputes_api.py` — под cases API
- Create: `backend/tests/unit/measurement_dispute/test_case_status.py`

### Manager-portal

- Modify: `manager-portal/src/lib/api.ts` — типы case / пути
- Modify: `manager-portal/src/lib/measurement-dispute/draft-storage.ts` — ключ по `caseId`
- Modify: `manager-portal/src/lib/measurement-dispute/poll-dispute.ts` — poll `GET /jobs/{id}`
- Modify: `manager-portal/src/app/(manager)/measurement-disputes/page.tsx` — список кейсов
- Modify: `manager-portal/src/app/(manager)/measurement-disputes/new/page.tsx` — `POST /cases` → replace `{caseId}`
- Replace role of: `manager-portal/src/app/(manager)/measurement-disputes/[id]/page.tsx` → workspace кейса (или переименовать в явный case page; Next `[id]` = `caseId`)
- Modify: `manager-portal/src/components/measurement-dispute/report-workspace.tsx` — `caseId`, create job under case
- Modify: `manager-portal/src/components/reviews-analysis-active-widget.tsx` — `/cases/active`
- Modify: `manager-portal/src/components/sidebar.tsx` — href `/measurement-disputes`

### Docs

- Modify: `docs/architecture/measurement-disputes.md`
- Modify: `docs/integrations/measurement-disputes-extension.md`

---

### Task 1: Case status helper + ORM

**Files:**
- Create: `backend/src/markethacker/modules/measurement_dispute/domain/case_status.py`
- Modify: `backend/src/markethacker/modules/measurement_dispute/domain/models.py`
- Test: `backend/tests/unit/measurement_dispute/test_case_status.py`

**Interfaces:**
- Produces: `compute_case_status(job_statuses: Sequence[str]) -> Literal["draft","active","succeeded","failed","cancelled"]` по правилам spec §Домен
- Produces: class `MeasurementDisputeCase` (`__tablename__ = "measurement_dispute_cases"`, поля spec)
- Produces: `MeasurementDispute.case_id: Mapped[uuid.UUID]` FK CASCADE

- [ ] **Step 1: Write failing unit tests for `compute_case_status`**

Cases: `[]→draft`; `[queued]→active`; `[running,succeeded]→active`; `[succeeded,failed]→failed`; `[succeeded]→succeeded`; `[cancelled,cancelled]→cancelled`; `[succeeded,cancelled]→succeeded`.

- [ ] **Step 2: Run test to verify it fails**

Run: `cd backend && uv run pytest tests/unit/measurement_dispute/test_case_status.py -v`  
Expected: FAIL import/name

- [ ] **Step 3: Implement `compute_case_status` and ORM models**

- [ ] **Step 4: Run tests — PASS**

- [ ] **Step 5: Commit** (если пользователь просит) в репо `backend`

---

### Task 2: Alembic migration (wipe + cases + case_id)

**Files:**
- Create: `backend/alembic/versions/20261127_0066_measurement_dispute_cases.py`
- Test: integration/migration path covered in Task 5; здесь — ручной upgrade на тестовой БД или pytest alembic если принят в проекте

**Interfaces:**
- Consumes: `MeasurementDisputeQuotaService.resolution_from_dispute` + `release`
- Upgrade: (1) для каждой строки `measurement_disputes` с `quota_reserved=true` — release через сервис квоты в sync/async bridge как принято в backend Alembic data migrations; (2) `DELETE FROM measurement_disputes`; (3) create `measurement_dispute_cases`; (4) add `case_id` NOT NULL FK + indexes
- Downgrade: drop `case_id`, drop cases table (jobs after wipe stay empty — acceptable)

- [ ] **Step 1: Write migration revision `0066` after `0065`**

- [ ] **Step 2: Run upgrade on local/test DB; assert tables**

Run: `cd backend && uv run alembic upgrade head` (или принятый в проекте способ)  
Expected: `measurement_dispute_cases` exists; `measurement_disputes.case_id` NOT NULL; no leftover reserved rows

- [ ] **Step 3: Commit** (по просьбе)

---

### Task 3: Repository + service (cases + jobs under case)

**Files:**
- Modify: `…/infrastructure/repository.py`
- Modify: `…/application/dto.py`
- Modify: `…/application/service.py`
- Test: extend/replace `backend/tests/integration/test_measurement_disputes_api.py` (реализация API в Task 4; здесь service-level или сразу API — предпочтительно писать failing API tests в Task 4; для Task 3 — unit/repo tests если есть паттерн, иначе перейти к Task 4 с red tests)

**Interfaces:**
- Produces: `MeasurementDisputeService.create_case(user_id, body: CreateMeasurementDisputeCaseRequest) -> MeasurementDisputeCase`
- Produces: `list_cases_for_user`, `list_active_cases_for_user`, `get_case_for_user`
- Produces: `create_job_for_case(user_id, case_id, body: CreateMeasurementDisputeRequest) -> MeasurementDispute` — логика бывшего `create` + проверка case принадлежит user + `case_id=` + bump `case.updated_at`
- Produces: `get_job_for_user` / `get_result` / `cancel` — бывшие методы по job id (переименовать для ясности допустимо)
- Removes: list/create job без case

Рекомендация: сразу Task 4 с failing integration tests, затем service+router вместе.

---

### Task 4: API router + schemas + integration tests

**Files:**
- Modify: `…/api/schemas.py`
- Modify: `…/api/router.py`
- Modify: `backend/tests/integration/test_measurement_disputes_api.py`

**Interfaces:**
- Case schemas: `MeasurementDisputeCaseSummary` с полями spec JSON
- Job schemas: существующий summary/result; пути `/jobs/{jobId}`
- Router paths exactly as spec §API
- Delete old endpoints listed in spec

- [ ] **Step 1: Rewrite integration tests to new paths**

Must assert:
- `POST /cases` → 201; then `POST /cases/{id}/jobs` photo + analyze → `GET /cases` length 1; detail has 2 jobs
- `GET /cases/active` while queued; empty after succeed/fail
- job on unknown case → 404
- cancel queued → release; case not active
- old `POST /measurement-disputes` → 404/405

- [ ] **Step 2: Run — FAIL on missing routes**

Run: `cd backend && uv run pytest tests/integration/test_measurement_disputes_api.py -v`

- [ ] **Step 3: Implement schemas + router + service/repo wiring**

- [ ] **Step 4: Run — PASS; ruff on touched paths**

- [ ] **Step 5: Commit** (по просьбе)

---

### Task 5: Migration reserved-quota test

**Files:**
- Modify or create: `backend/tests/integration/test_measurement_dispute_cases_migration.py` (или добавить в существующий migration test harness, если есть)

- [ ] **Step 1: Test that seeds a `quota_reserved=true` job (pre-migration shape if harness allows), runs upgrade, asserts credits released and jobs gone**

If full alembic harness heavy: document deploy step + unit-test a `release_all_reserved_disputes(session)` helper called from migration upgrade, and test that helper.

- [ ] **Step 2: Implement helper used by migration; PASS**

---

### Task 6: Portal API types + draft keyed by caseId + poll paths

**Files:**
- Modify: `manager-portal/src/lib/api.ts`
- Modify: `manager-portal/src/lib/measurement-dispute/draft-storage.ts`
- Modify: `manager-portal/src/lib/measurement-dispute/poll-dispute.ts`
- Test: `manager-portal/src/lib/measurement-dispute/draft-storage` — если нет test runner для storage, ручная проверка + tsc

**Interfaces:**
- `MeasurementDisputeCaseSummary`, `MeasurementDisputeCaseStatus`, `CreateMeasurementDisputeCaseBody`
- `saveWorkspaceDraft(caseId, draft)`, `loadWorkspaceDraft(caseId)`, `clearWorkspaceDraft(caseId)`, `patchWorkspaceJobState(caseId, patch)`
- Storage key: `mh_measurement_dispute_workspace:${caseId}`
- Poll: `GET /measurement-disputes/jobs/${id}` and `/result`

- [ ] **Step 1: Update types and draft/poll helpers**
- [ ] **Step 2: `npx tsc --noEmit` — fix errors from call sites in same task or leave typed stubs for Task 7–8**

---

### Task 7: Portal list + new + sidebar

**Files:**
- Modify: `…/measurement-disputes/page.tsx`
- Modify: `…/measurement-disputes/new/page.tsx`
- Modify: `…/components/sidebar.tsx`

- [ ] **Step 1: List loads `GET /measurement-disputes/cases`; row click → `/measurement-disputes/{caseId}`; columns: дата, title/file, status, net из reportSummary**
- [ ] **Step 2: After successful local parse on `/new`, `POST /cases` then `router.replace(/measurement-disputes/${case.id})` with groups/totals passed via sessionStorage or re-parse — выбрать: хранить parsed sheet только в memory на new до redirect недостаточно; **решение:** `/new` после POST кладёт одноразовый payload (`groups`+`totals`+`fileMeta`) в `sessionStorage` ключ `mh_measurement_dispute_bootstrap:${caseId}`, `{caseId}` page читает и пишет в draft**
- [ ] **Step 3: Sidebar href `/measurement-disputes`**
- [ ] **Step 4: tsc PASS**

---

### Task 8: Portal case workspace + widget

**Files:**
- Modify: `…/measurement-disputes/[id]/page.tsx` (id = caseId)
- Modify: `…/components/measurement-dispute/report-workspace.tsx`
- Modify: `…/components/reviews-analysis-active-widget.tsx`

- [ ] **Step 1: `[id]` page loads `GET /cases/{id}`; if bootstrap/draft has groups — full `ReportWorkspace`; else header from case + jobs/results panels (load result via `/jobs/{id}/result` for succeeded); no redirect to `/new` for active jobs**
- [ ] **Step 2: `ReportWorkspace` takes `caseId`; create via `POST /cases/{caseId}/jobs`; poll jobs; draft keyed by caseId**
- [ ] **Step 3: Widget uses `GET /cases/active`; one FAB entry per case; href `/measurement-disputes/{caseId}`**
- [ ] **Step 4: Remove dead «job detail dump» UX; keep failed/cancel messaging user-facing**
- [ ] **Step 5: tsc PASS**

---

### Task 9: Docs

**Files:**
- Modify: `docs/architecture/measurement-disputes.md`
- Modify: `docs/integrations/measurement-disputes-extension.md`

- [ ] **Step 1: Replace job-as-list model with cases API tables and examples from spec**
- [ ] **Step 2: Commit in `docs` repo (по просьбе)

---

## Execution notes

- Репозитории раздельные: коммиты в `backend`, `manager-portal`, `docs` по отдельности.
- После Task 4 backend usable с curl; portal ломается до Task 6–8 — выкатывать согласованно.
- Extension код не трогать; только дока Task 9.
