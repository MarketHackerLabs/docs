# Stock Control Add Products + Admin Menu Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Модалка «Добавить товары» (ручной ввод + файл → черновик → Confirm) с optional остатком; в Admin — отдельный пункт «Контроль остатков» → «Параметры».

**Architecture:** Черновик на клиенте; файл разбирается через `POST …/product-matching/parse` (без записи, переиспользует Python CSV/XLSX). Запись только через `POST …/commit` (all-or-nothing, ≤2000 строк): upsert matching + optional `ledger/set`. Admin: новая страница UI на существующем `GET|PATCH /admin/settings`.

**Tech Stack:** FastAPI / SQLAlchemy / pytest; manager-portal Next.js + существующий `Modal`; admin-panel sidebar pattern как card-audit.

**Spec:** `docs/superpowers/specs/2026-08-26-stock-control-add-products-design.md`

## Global Constraints

- Тексты продавца без жаргона API / внутренних имён.
- Owner-only для parse/commit/template (как текущий import).
- Лимит черновика и commit: **2000** строк.
- Confirm all-or-nothing: при любой ошибке валидации пакета — ноль записей в БД.
- Пустой `warehouseQty` не меняет существующий `physical_qty`.
- Нет `any` / `Record<string, any>` / `unknown` без нужды.
- Коммитить только по явной просьбе пользователя (шаги Commit = stage; не коммитить самовольно).
- Backend тесты: `uv run pytest <path> -v` из `backend/`.
- Не трогать org override TTL на странице организации; seller TTL UI не добавлять.

## Locked decisions (из open questions спеки)

| Вопрос | Решение |
|--------|---------|
| Парсер файла | `POST …/parse` на бэкенде (без новых npm xlsx) |
| JSON поля | `internalSku`, `wbSku`, `ozonSku`, `warehouseQty` (camelCase via CamelModel) |
| `POST …/import` | Оставить endpoint (3 колонки, без qty) для совместимости; portal больше не вызывает |
| Admin API | Только новая страница; тот же `/admin/settings` |

## File map

Создать:

- `backend/tests/unit/product_matching/test_matching_commit.py`
- `manager-portal/src/lib/stock-control-draft.ts` — клиентская валидация черновика + лимит
- `manager-portal/src/components/stock-control-add-products-modal.tsx`
- `admin-panel/src/components/stock-control-nav.tsx`
- `admin-panel/src/app/(admin)/stock-control/settings/page.tsx`

Изменить:

- `backend/src/markethacker/modules/product_matching/domain/matching_import.py` — qty column, draft parse, validate package
- `backend/src/markethacker/modules/product_matching/application/service.py` — parse, commit, template header
- `backend/src/markethacker/modules/product_matching/api/schemas.py` — request/response models
- `backend/src/markethacker/modules/product_matching/api/router.py` — `/parse`, `/commit`
- `backend/tests/unit/product_matching/test_matching_import.py` — qty + draft
- `manager-portal/src/lib/api.ts` — parse/commit helpers
- `manager-portal/src/components/modal.tsx` — size `full` при необходимости
- `manager-portal/src/app/(manager)/stock-control/page.tsx` — кнопка + модалка, убрать старый import card flow
- `admin-panel/src/components/sidebar.tsx` — зона «Контроль остатков»
- `admin-panel/src/app/(admin)/settings/page.tsx` — убрать карточку
- `docs/architecture/stock-control.md` — поток добавления товаров + admin menu

---

### Task 1: Domain — draft parse, qty column, package validation

**Files:**
- Modify: `backend/src/markethacker/modules/product_matching/domain/matching_import.py`
- Test: `backend/tests/unit/product_matching/test_matching_import.py`

**Interfaces:**
- Produces:
  - `MAX_MATCHING_COMMIT_ROWS = 2000`
  - `MatchingImportRow` добавляет `warehouse_qty: int | None = None`
  - `DraftMatchingRow(row_number: int, internal_sku: str | None, wb_sku: str | None, ozon_sku: str | None, warehouse_qty: int | None, issues: tuple[MatchingImportIssue, ...])`
  - `DraftMatchingResult(rows: tuple[DraftMatchingRow, ...], truncated: bool)`
  - `parse_matching_draft_table(table: list[list[str]]) -> DraftMatchingResult` — все непустые/не-`#` строки попадают в `rows` (с issues на строке); при >2000 строк после фильтра — `truncated=True`, в `rows` только первые 2000
  - `parse_matching_draft_file(content: bytes, *, filename: str) -> DraftMatchingResult`
  - `validate_matching_commit_rows(rows: Sequence[MatchingImportRow]) -> tuple[MatchingImportIssue, ...]` — пакетная валидация для commit (дубликаты WB/Ozon внутри пакета, SKU, ≥1 marketplace, qty ≥ 0 или None); пустой список rows → issue
  - Заголовки qty: `на складе`, `warehouse qty`, `warehouse_qty`, `qty`, `остаток`
  - Существующий `parse_matching_table` / `import` путь: если колонка qty есть — заполнять `warehouse_qty`, иначе `None` (обратная совместимость)

- [ ] **Step 1: Failing tests**

```python
from markethacker.modules.product_matching.domain.matching_import import (
    MAX_MATCHING_COMMIT_ROWS,
    MatchingImportRow,
    parse_matching_draft_table,
    validate_matching_commit_rows,
)


def test_draft_keeps_invalid_rows_with_issues() -> None:
    result = parse_matching_draft_table(
        [
            ["SKU", "артикул WB", "артикул Ozon", "На складе"],
            ["", "WB-1", "", ""],
            ["SKU-1", "", "", ""],
            ["SKU-2", "WB-2", "", "5"],
        ]
    )
    assert len(result.rows) == 3
    assert result.rows[0].issues  # empty sku
    assert result.rows[1].issues  # no marketplace
    assert not result.rows[2].issues
    assert result.rows[2].warehouse_qty == 5


def test_draft_truncates_at_limit() -> None:
    header = ["SKU", "артикул WB", "артикул Ozon"]
    body = [[f"S{i}", f"W{i}", ""] for i in range(MAX_MATCHING_COMMIT_ROWS + 3)]
    result = parse_matching_draft_table([header, *body])
    assert result.truncated is True
    assert len(result.rows) == MAX_MATCHING_COMMIT_ROWS


def test_validate_commit_rejects_bad_qty_and_duplicates() -> None:
    issues = validate_matching_commit_rows(
        [
            MatchingImportRow(1, "A", "WB-1", None, -1),
            MatchingImportRow(2, "B", "WB-1", None, None),
        ]
    )
    assert issues
```

- [ ] **Step 2: Run tests — expect FAIL**

Run: `uv run pytest tests/unit/product_matching/test_matching_import.py -v`

- [ ] **Step 3: Implement domain changes**

Сохранить семантику старого `parse_matching_table` для import (invalid rows по-прежнему не в `rows`), добавить draft/validate как выше. Qty: пустая ячейка → `None`; нечисло / float / `<0` → issue на строке (draft) или в validate (commit).

- [ ] **Step 4: Run tests — expect PASS**

Run: `uv run pytest tests/unit/product_matching/test_matching_import.py -v`

- [ ] **Step 5: Stage**

```bash
git add src/markethacker/modules/product_matching/domain/matching_import.py \
  tests/unit/product_matching/test_matching_import.py
```

---

### Task 2: API schemas + parse + commit service (matching only first)

**Files:**
- Modify: `backend/src/markethacker/modules/product_matching/api/schemas.py`
- Modify: `backend/src/markethacker/modules/product_matching/application/service.py`
- Modify: `backend/src/markethacker/modules/product_matching/api/router.py`
- Test: `backend/tests/unit/product_matching/test_matching_commit.py` (service-level с fake repo / или unit validate+service helpers)

**Interfaces:**
- Produces schemas:
  - `CommitMatchingRowIn`: `internal_sku: str`, `wb_sku: str | None = None`, `ozon_sku: str | None = None`, `warehouse_qty: int | None = None`
  - `CommitMatchingRequest`: `rows: list[CommitMatchingRowIn]`
  - `CommitMatchingResult`: `committed: int`, `issue_count: int`, `issues: list[ImportIssue]`
  - `ParseMatchingRowOut`: `row_number`, `internal_sku`, `wb_sku`, `ozon_sku`, `warehouse_qty`, `issues: list[ImportIssue]`
  - `ParseMatchingResult`: `rows: list[ParseMatchingRowOut]`, `truncated: bool`
- Produces service:
  - `parse_matching_file_draft(...) -> ParseMatchingResult`
  - `commit_matching(*, org_id, requester_id, rows: Sequence[CommitMatchingRowIn]) -> CommitMatchingResult`
- Template: `MATCHING_CSV_HEADER = "SKU,артикул WB,артикул Ozon,На складе\n"`
- Router: `POST /parse` (multipart file), `POST /commit` (JSON body)

**Commit matching-only behavior in this task** (qty wiring → Task 3):  
Если `validate_matching_commit_rows` вернул issues → `committed=0`, issues в ответе, **без** вызовов repo.  
Иначе upsert product/listings как в `import_matching` (без ledger).

- [ ] **Step 1: Failing unit tests for validate→no write and happy path count**

```python
def test_commit_all_or_nothing_returns_issues_without_side_effects() -> None:
    # Arrange service with repo mock that fails if upsert called
    ...
```

- [ ] **Step 2: Implement schemas, service methods, router endpoints**

`/parse`: max 2 MB как import; owner; map `DraftMatchingResult` → `ParseMatchingResult`.

`/commit`: owner; len(rows)==0 или >2000 → `ValidationError` с понятным текстом; иначе validate; on success upsert loop.

- [ ] **Step 3: Tests PASS**

Run: `uv run pytest tests/unit/product_matching/test_matching_commit.py tests/unit/product_matching/test_matching_import.py -v`

- [ ] **Step 4: Stage**

```bash
git add src/markethacker/modules/product_matching/api/schemas.py \
  src/markethacker/modules/product_matching/application/service.py \
  src/markethacker/modules/product_matching/api/router.py \
  tests/unit/product_matching/test_matching_commit.py
```

---

### Task 3: Commit → optional warehouse qty (stock_control)

**Files:**
- Modify: `backend/src/markethacker/modules/product_matching/application/service.py`
- Test: `backend/tests/unit/product_matching/test_matching_commit.py`

**Interfaces:**
- Consumes: `StockControlRepository.ensure_sku`, `apply_ledger` + `apply_set` from `stock_control.domain.ledger`
- После успешного upsert product: если `warehouse_qty is not None` → `sku = await stock_repo.ensure_sku(org_id, product.id)` → `apply_set` → `apply_ledger`

Не вызывать stock_control HTTP; работать в том же `AsyncSession`.

- [ ] **Step 1: Test** — commit with qty calls ensure+set; without qty does not change physical

- [ ] **Step 2: Implement wiring in `commit_matching`**

- [ ] **Step 3: pytest PASS**

- [ ] **Step 4: Stage**

---

### Task 4: Manager portal — draft helpers + API client

**Files:**
- Create: `manager-portal/src/lib/stock-control-draft.ts`
- Modify: `manager-portal/src/lib/api.ts`

**Interfaces:**
- `MAX_STOCK_CONTROL_DRAFT_ROWS = 2000`
- `StockControlDraftRow`: `{ id: string; internalSku: string; wbSku: string; ozonSku: string; warehouseQty: string; issues: { field: string; message: string }[] }`
- `validateStockControlDraft(rows: StockControlDraftRow[]): StockControlDraftRow[]` — пересчёт issues (зеркало правил: sku, wb|ozon, duplicates, qty empty|int≥0)
- `draftHasBlockingIssues(rows): boolean`
- `canConfirmDraft(rows): boolean` — rows.length>0 && !draftHasBlockingIssues && length≤2000
- API:
  - `parseStockControlMatching(orgId, token, file) -> ParseMatchingResult`
  - `commitStockControlMatching(orgId, token, rows) -> CommitMatchingResult`
  - Keep template download; stop using `importStockControlOffers` from page (можно оставить функцию deprecated)

- [ ] **Step 1: Unit-test draft validation** (если в portal есть vitest/jest — использовать; иначе минимальная чистая функция и покрыть логику в Task 5 вручную / добавить vitest только если уже есть в проекте)

Проверить наличие тест-раннера:

```bash
cd manager-portal && cat package.json | rg '"test"'
```

Если тестов нет — пропустить automated test, проверить вручную в Task 5.

- [ ] **Step 2: Implement helpers + api.ts**

- [ ] **Step 3: Stage**

---

### Task 5: AddProducts modal UI

**Files:**
- Create: `manager-portal/src/components/stock-control-add-products-modal.tsx`
- Modify: `manager-portal/src/components/modal.tsx` — добавить `size?: ... | "full"` → `max-w-[min(96vw,72rem)]`

**Interfaces:**
- Props: `{ open: boolean; onClose: () => void; orgId: string; token: string; onCommitted: () => void }`
- UI (RU): title «Добавить товары»; table columns SKU / Wildberries / Ozon / На складе; buttons «Строка», «Из файла», «Шаблон», «К ошибке», «Подтвердить», «Отмена»
- «Из файла» → `parseStockControlMatching` → append rows (map issues); if `truncated` — toast/alert «Можно добавить не больше 2000 строк»
- Client re-validate after edits; highlight rows with issues; Confirm disabled via `canConfirmDraft`
- «К ошибке» cycles `querySelector` / row refs to next issue row, scrollIntoView
- Confirm → `commitStockControlMatching`; on `issueCount>0` map issues back onto rows; on success `onCommitted` + close

- [ ] **Step 1: Implement modal component**

- [ ] **Step 2: Manual check** — open modal, add bad row, Confirm disabled, jump works

- [ ] **Step 3: Stage**

---

### Task 6: Wire stock-control page

**Files:**
- Modify: `manager-portal/src/app/(manager)/stock-control/page.tsx`

**Interfaces:**
- Owner: button «Добавить товары» (вместо карточки Шаблон/Загрузить или вместо contents карточки — одна кнопка + модалка)
- Remove direct `importStockControlOffers` file input flow and importResult card UI
- Keep template download inside modal only
- After commit: refresh board (`reload`)

Empty state copy: обновить на «Добавьте товары и при необходимости укажите остаток на складе.»

- [ ] **Step 1: Wire modal; delete old import card UX**

- [ ] **Step 2: Smoke in browser** (owner)

- [ ] **Step 3: Stage**

---

### Task 7: Admin — menu item + settings page

**Files:**
- Create: `admin-panel/src/components/stock-control-nav.tsx`
- Create: `admin-panel/src/app/(admin)/stock-control/settings/page.tsx`
- Modify: `admin-panel/src/components/sidebar.tsx`
- Modify: `admin-panel/src/app/(admin)/settings/page.tsx`

**Interfaces:**
- Sidebar zone «Продукт», после card-generate или рядом:

```ts
{
  id: "stock-control",
  label: "Контроль остатков",
  icon: /* Package or Boxes from lucide — уже используемый в проекте */,
  match: ["/stock-control"],
  superuserOnly: true,
  children: [
    { href: "/stock-control/settings", label: "Параметры", exact: true },
  ],
},
```

- Nav tabs: только «Параметры»
- Settings page: load `GET /admin/settings`, form field `stockControl.wbContentCatalogTtlHours`, save via `PATCH /admin/settings` с `{ stockControl: { wbContentCatalogTtlHours } }` — скопировать паттерн с текущей карточки на `/settings`
- Remove Card «Контроль остатков» from `/settings/page.tsx` and related form state if unused

- [ ] **Step 1: Implement nav + page + sidebar**

- [ ] **Step 2: Remove card from general settings**

- [ ] **Step 3: Typecheck/lint admin-panel for touched files**

- [ ] **Step 4: Stage**

---

### Task 8: Docs architecture

**Files:**
- Modify: `docs/architecture/stock-control.md` (и при наличии — короткий абзац в product-matching)

**Content:**
- Добавление товаров: модалка, parse без записи, commit, лимит 2000, optional склад
- Admin: пункт меню Параметры (TTL)

- [ ] **Step 1: Update docs**

- [ ] **Step 2: Stage**

---

## Spec coverage checklist

| Spec requirement | Task |
|------------------|------|
| Модалка ручной + файл → черновик | 5, 6 |
| Файл не пишет сразу; append | 2, 5 |
| Проблемные строки + Confirm lock + «К ошибке» | 1, 4, 5 |
| Лимит 2000 | 1, 2, 4, 5 |
| Confirm matching + optional qty | 2, 3 |
| All-or-nothing | 1, 2 |
| Пустой qty не затирает physical | 3 |
| Template 4 columns | 2, 5 |
| UI без старого import card | 6 |
| Admin menu + settings only | 7 |
| Docs | 8 |
| Seller texts without jargon | 5, 6 |
| `POST /import` not used by portal | 6 (endpoint kept) |

## Self-review notes

- Парсер draft отдельно от legacy `parse_matching_table`, чтобы import не начал писать invalid rows.
- Qty в commit через тот же session, что matching — без отдельного HTTP ledger.
- Modal `full` size — чтобы таблица на 2000 строк была usable (виртуализацию не делаем в MVP; скролл тела модалки).

---

Plan complete and saved to `docs/superpowers/plans/2026-08-26-stock-control-add-products.md`. Two execution options:

**1. Subagent-Driven (recommended)** — свежий субагент на задачу, ревью между задачами  

**2. Inline Execution** — выполнение в этой сессии с чекпоинтами  

Which approach?
