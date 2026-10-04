# WB nm→sizes Board Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** При добавлении nmID разворачивать все размеры WB в отдельные SKU|chrt|Ozon-строки; на доске группировать размеры по nm; алерты оставить на каждый размер.

**Architecture:** Content-кэш хранит `chrt + nm + tech_size`. `ProductMatchingService` (уже имеет `StockControlRepository`) перед commit/parse-expand резолвит nm→sizes; при отсутствии nm в кэше — force Content refresh кабинета с active-токеном. Board отдаёт `wbNmId`/`techSize`; Portal группирует страницу. Sync hot path без смены matched-only.

**Tech Stack:** Python 3.11+ / SQLAlchemy / Alembic / pytest; Manager Portal Next.js / React.

**Spec:** `docs/superpowers/specs/2026-08-26-wb-sizes-board-design.md`

## Global Constraints

- Схема matching для продавца: SKU | артикул WB | артикул Ozon; в БД WB listing `mp_sku = chrt`, `catalog_id = nm`.
- Ввод WB в модалке/файле = **nmID** (не chrt, не баркод).
- Ozon при expand: копировать только если у nm ровно 1 размер; при N>1 — пусто.
- Physical qty из колонки — одинаково на каждый размер; пусто = не трогать.
- Лимит 2000 — после развёртки.
- Алерты/Telegram — на size-SKU, без сводки на nm.
- Нет авто-матча Ozon↔WB, баркодов в UI, выбора кабинета в модалке.
- Нет `any` / `Record<string, any>`; тексты продавцу без путей API.
- Коммитить только по явной просьбе пользователя.
- Тесты: `cd backend && uv run pytest <path> -v`.

## File map

Создать:

- `backend/alembic/versions/YYYYMMDD_XXXX_wb_catalog_tech_size.py`
- `backend/src/markethacker/modules/product_matching/domain/wb_nm_expand.py`
- `backend/tests/unit/product_matching/test_wb_nm_expand.py`
- (при необходимости) `backend/tests/unit/stock_control/test_wb_catalog_sizes_parse.py`

Изменить:

- `stock_control/domain/models.py` — `tech_size`, `wb_size` на `StockControlWbCatalogEntry`
- `stock_control/infrastructure/marketplace/wb.py` — парсинг sizes → богаче структура
- `stock_control/infrastructure/repository.py` — replace/list sizes; `list_sizes_for_nm`
- `stock_control/application/sync.py` + Fake ports/tests — совместимость с новым replace
- `product_matching/application/service.py` — expand перед validate/commit; force refresh
- `product_matching` schemas / portal draft copy — подсказка nm
- `stock_control/api/schemas.py` + `application/service.py` board — `wbNmId`, `techSize`
- `manager-portal` board page + `api.ts` — группировка
- `docs/architecture/stock-control.md`, `product-matching.md`

---

### Task 1: Domain type + catalog ORM/migration

**Files:**
- Modify: `backend/src/markethacker/modules/stock_control/domain/models.py` (`StockControlWbCatalogEntry`)
- Create: `backend/alembic/versions/*_wb_catalog_tech_size.py` (следующий revision id по стилю репо)
- Create/Modify: `backend/src/markethacker/modules/stock_control/domain/wb_catalog.py` — dataclass размера

**Interfaces:**
- Produces:
  ```python
  @dataclass(frozen=True, slots=True)
  class WbCatalogSize:
      chrt_id: int
      nm_id: str
      tech_size: str  # "" если нет
      wb_size: str    # "" если нет
  ```
- ORM: `tech_size: Mapped[str] = mapped_column(String(64), nullable=False, default="")`, `wb_size` аналогично.

- [ ] **Step 1: Add dataclass to `wb_catalog.py`**

```python
@dataclass(frozen=True, slots=True)
class WbCatalogSize:
    chrt_id: int
    nm_id: str
    tech_size: str = ""
    wb_size: str = ""
```

- [ ] **Step 2: Extend ORM + Alembic**

Добавить колонки `tech_size`, `wb_size` `VARCHAR(64) NOT NULL DEFAULT ''`. Индекс `ix_stock_control_wb_catalog_account_nm` на `(marketplace_account_id, nm_id)`.

- [ ] **Step 3: Commit (по просьбе)**

```bash
cd backend
git add src/markethacker/modules/stock_control/domain/models.py \
  src/markethacker/modules/stock_control/domain/wb_catalog.py \
  alembic/versions/*_wb_catalog_tech_size.py
git commit -m "feat(stock_control): store WB tech_size in content catalog cache"
```

---

### Task 2: Adapter + repository write/read sizes

**Files:**
- Modify: `wb.py` `_list_card_ids` / `fetch_content_catalog`
- Modify: `repository.py` `replace_wb_catalog_entries`, добавить `list_wb_catalog_sizes`, `list_sizes_for_nm`
- Modify: sync + unit tests / FakeSellerPort `fetch_content_catalog`

**Interfaces:**
- `fetch_content_catalog(token) -> list[WbCatalogSize]` (или `dict` chrt→WbCatalogSize; предпочтительно **list**, sync строит map)
- `replace_wb_catalog_entries(account_id, org_id, entries: Sequence[WbCatalogSize])`
- `list_wb_catalog_map` оставить `dict[int, str | None]` для sync (из nm_id)
- `list_sizes_for_nm(account_id, nm_id: str) -> list[WbCatalogSize]`
- `list_accounts_with_nm(org_id, nm_id) -> list[uuid.UUID]` (через join credentials active + entries)

- [ ] **Step 1: Failing parser test**

В тесте на парсинг карточки (новый или `test_wb_parser.py`):

```python
def test_content_catalog_keeps_tech_size() -> None:
    # минимальный payload cards[].sizes[] с chrtID, techSize, wbSize
    # assert sizes[0].tech_size == "XL"
```

Реализовать извлечением в `_list_card_ids` → возвращать `list[WbCatalogSize]`.

- [ ] **Step 2: Update `replace_wb_catalog_entries` + callers**

Sync:

```python
refreshed = await content_port.fetch_content_catalog(token)
await repo.replace_wb_catalog_entries(account_id, cabinet.org_id, refreshed)
wb_catalog_ids_by_chrt = {row.chrt_id: row.nm_id for row in refreshed}
```

Fake `CountingWbPort.fetch_content_catalog` → `list[WbCatalogSize]`.

- [ ] **Step 3: `list_sizes_for_nm` + test**

```python
async def list_sizes_for_nm(self, account_id: uuid.UUID, nm_id: str) -> list[WbCatalogSize]:
    ...
```

- [ ] **Step 4: Run tests**

`uv run pytest tests/unit/stock_control/test_wb_parser.py tests/unit/stock_control/test_repository.py tests/unit/stock_control/test_sync.py -v`  
Expected: PASS

- [ ] **Step 5: Commit (по просьбе)**

---

### Task 3: Pure expand helper

**Files:**
- Create: `backend/src/markethacker/modules/product_matching/domain/wb_nm_expand.py`
- Test: `backend/tests/unit/product_matching/test_wb_nm_expand.py`

**Interfaces:**
- Consumes: `WbCatalogSize`, `MatchingImportRow` / простой input dataclass
- Produces:

```python
@dataclass(frozen=True, slots=True)
class NmExpandInput:
    base_sku: str
    nm_id: str
    ozon_sku: str | None
    physical_qty: int | None  # None = не трогать; иначе int для всех sizes

def expand_nm_to_matching_rows(
    row: NmExpandInput,
    sizes: Sequence[WbCatalogSize],
) -> list[MatchingImportRow]:
    ...
```

Правила:
- `sizes` пуст → caller не вызывает (ошибка выше)
- `internal_sku = f"{base}/{tech}"` если `tech_size.strip()` else `f"{base}/{chrt}"`
- `wb_sku = str(chrt_id)`, catalog позже = nm
- `ozon_sku` только если `len(sizes)==1`
- qty копируется на каждую строку

- [ ] **Step 1–4: TDD** как в skill (fail → implement → pass)

- [ ] **Step 5: Commit (по просьбе)**

---

### Task 4: Wire expand into parse/commit

**Files:**
- Modify: `product_matching/application/service.py`
- Modify: tests `test_matching_commit.py` / новый integration-style unit с Fake catalog
- Modify: portal copy + template hint (колонка = nm)

**Interfaces:**
- Перед `validate_matching_commit_rows`: для каждой входной строки с непустым `wb_sku` трактовать как nm (digit), резолвить кабинеты, expand.
- Force refresh: если ни в одном active WB credential нет nm — для каждого active WB account вызвать Content refresh (нужны `ApiTokenVault` + `WbSellerAdapter`); затем повторить lookup. Если всё ещё 0 — issue «Карточка не найдена в кабинете». Если >1 account — issue «Карточка найдена в нескольких кабинетах».
- Нет active WB token при непустом WB — issue «Сначала сохраните ключ Wildberries в настройках кабинета».
- После expand проверить `len(rows) <= MAX_MATCHING_COMMIT_ROWS`.
- `_upsert_matching_row`: `catalog_id=nm` (из expand), `mp_sku=chrt` — **не** ставить catalog_id=wb_sku как сейчас для digit wb.

Изменить persist:

```python
await self._repo.upsert_listing(
    ...,
    mp_sku=row.wb_sku,  # chrt after expand
    catalog_id=row.catalog_id,  # добавить поле в MatchingImportRow ИЛИ отдельный параметр nm
)
```

Практично: расширить `MatchingImportRow` полем `wb_catalog_id: str | None = None` (nm), выставлять в expand.

Parse file draft: либо expand на parse (нужен async catalog — тогда parse endpoint тоже expand), либо expand только на commit, а parse оставляет nm и UI показывает «будет развёрнуто».  
**Решение плана:** expand на **commit** и на **parse** (оба async через service), чтобы таблица черновика сразу показывала размеры.

- [ ] **Step 1: Unit tests** на service с stub repo sizes (mock list_sizes_for_nm / in-memory)

- [ ] **Step 2: Implement resolve + expand in `commit_matching` and `parse_matching_file_draft`**

- [ ] **Step 3: Portal** — description модалки: «Артикул WB — nmID карточки; размеры подставятся сами. Ozon укажите на каждый размер после развёртки.»

- [ ] **Step 4: pytest product_matching** PASS

- [ ] **Step 5: Commit (по просьбе)**

---

### Task 5: Board API fields

**Files:**
- Modify: `stock_control/api/schemas.py` `BoardSku`
- Modify: `stock_control/application/service.py` `_build_board_skus`
- Modify: `manager-portal/src/lib/api.ts` types
- Optionally widen board `q` search to match WB `catalog_id`

**Interfaces:**
- `BoardSku.wb_nm_id: str | None` — с WB listing `catalog_id`
- `BoardSku.tech_size: str | None` — из catalog entry по chrt listing `mp_sku`, иначе вывести из suffix `internal_sku` после `/`, иначе null

```python
# schema
wb_nm_id: str | None = None
tech_size: str | None = None
```

Поиск: если `q` — digits, также фильтровать products у которых listing.catalog_id == q.

- [ ] **Step 1–4: tests + implement**

- [ ] **Step 5: Commit (по просьбе)**

---

### Task 6: Portal board grouping

**Files:**
- Modify: `manager-portal/src/app/(manager)/stock-control/page.tsx`
- Modify: `api.ts` mapping

**UI:**
- Сгруппировать `skus` страницы по `wbNmId` (null → одиночные).
- Шапка группы: `MarketplaceThumb` / image от nm, текст nm, число размеров, badge если любой child attention.
- Дети: текущие строки размера + показать `techSize` рядом с internalSku.
- Не ломать expand складов / ledger.

- [ ] **Step 1: Implement grouping helper** `groupBoardSkusByWbNm(skus): BoardGroup[]` в `lib/` рядом со stock-control

- [ ] **Step 2: Render groups in page**

- [ ] **Step 3: Manual check** — nm с 2+ sizes выглядит группой

- [ ] **Step 4: Commit (по просьбе)**

---

### Task 7: Architecture docs

**Files:**
- `docs/architecture/stock-control.md`
- `docs/architecture/product-matching.md`

- [ ] **Step 1: Document** nm input, chrt in DB, board groups, catalog tech_size

- [ ] **Step 2: Commit (по просьбе)**

---

## Self-review (plan vs spec)

| Spec | Task |
|------|------|
| Catalog tech_size / wb_size | 1–2 |
| Expand nm → rows SKU/chrt/Ozon rules | 3–4 |
| Force refresh / cabinet resolve errors | 4 |
| Board wbNmId + techSize + search nm | 5 |
| Portal group UI | 6 |
| Alerts unchanged | (no task — regression via sync tests) |
| Docs | 7 |
| Ozon auto-match out of scope | Global |

---

## Execution handoff

Plan complete and saved to `docs/superpowers/plans/2026-08-26-wb-sizes-board.md`.

**Two execution options:**

1. **Subagent-Driven (recommended)** — fresh subagent per task  
2. **Inline Execution** — this session with checkpoints  

Which approach?
