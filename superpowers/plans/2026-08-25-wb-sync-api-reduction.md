# WB Sync API Reduction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Снизить число WB Content/stocks запросов на sync: stocks только по matched `mp_sku`, Content — редкий полный refresh с персистом и TTL (platform default + org override в Admin).

**Architecture:** Application sync собирает chrt из `product_matching` (org + marketplace; у listing нет `account_id`). `WbSellerAdapter.fetch_fbs_stocks` больше не тянет Content; отдельный `fetch_content_catalog` пишется в `stock_control_wb_catalog_entries` + `catalog_synced_at`. Эффективный TTL = org override ?? platform setting (24h).

**Tech Stack:** Python 3.11+ / FastAPI / SQLAlchemy 2 / Alembic / pytest; Admin Panel Next.js; platform_settings.

**Spec:** `docs/superpowers/specs/2026-08-25-wb-sync-api-reduction-design.md`

## Global Constraints

- Не менять cadence 15 мин, advice/alerts, Ozon-оптимизацию, UI TTL у продавца.
- Matching — org+marketplace (не per-account); «matched для WB-кабинета» = все WB listings организации.
- На маркетплейс остатки не писать; PUT stocks запрещён.
- Токены и сырые JWT не логировать.
- Тексты для продавца без жаргона API; admin может видеть технические имена настроек.
- Нет `any` / `Record<string, any>` / `unknown` без нужды.
- Коммитить только по явной просьбе пользователя (шаги Commit — что класть в индекс; не коммитить самовольно).
- Тесты из `backend/`: `uv run pytest <path> -v`.

## File map

Создать:

- `backend/src/markethacker/modules/stock_control/domain/wb_catalog.py` — parse chrt, `should_refresh_wb_content_catalog`, effective TTL
- `backend/alembic/versions/20260825_0062_wb_content_catalog_cache.py`
- `backend/tests/unit/stock_control/test_wb_catalog.py`
- `admin-panel` UI: поле platform TTL + org override на странице org (см. Task 7–8)

Изменить:

- `backend/src/markethacker/modules/stock_control/domain/models.py` — `catalog_synced_at`, org TTL, ORM entries
- `backend/src/markethacker/modules/stock_control/domain/ports.py` — optional `chrt_ids` на stocks
- `backend/src/markethacker/modules/stock_control/infrastructure/marketplace/wb.py` — split Content / stocks
- `backend/src/markethacker/modules/stock_control/infrastructure/marketplace/ozon.py` — принять `chrt_ids=None` и игнорировать
- `backend/src/markethacker/modules/stock_control/infrastructure/api_token_vault.py` — `catalog_synced_at`
- `backend/src/markethacker/modules/stock_control/infrastructure/repository.py` — catalog CRUD
- `backend/src/markethacker/modules/stock_control/application/sync.py` — matched stocks + rare Content
- `backend/src/markethacker/modules/platform_settings/application/defaults.py` — ключ TTL
- `backend/src/markethacker/modules/admin/...` — platform settings schema + org patch
- `admin-panel/src/lib/api.ts`, settings page, organizations/[id]
- `docs/architecture/stock-control.md` — секция sync
- `backend/tests/unit/stock_control/test_wb_parser.py`, `test_sync.py`

Не трогать: manager-portal PATCH settings продавца (reserve/low/stale), Telegram copy.

---

### Task 1: Чистые функции каталога WB

**Files:**
- Create: `backend/src/markethacker/modules/stock_control/domain/wb_catalog.py`
- Test: `backend/tests/unit/stock_control/test_wb_catalog.py`

**Interfaces:**
- Produces:
  - `parse_wb_chrt_id(mp_sku: str) -> int | None`
  - `collect_matched_chrt_ids(mp_skus: Sequence[str]) -> list[int]` (уникальные, порядок стабильный)
  - `effective_wb_content_ttl_hours(*, org_override: int | None, platform_default: int) -> int`
  - `should_refresh_wb_content_catalog(*, catalog_synced_at: datetime | None, now: datetime, ttl_hours: int, catalog_entry_count: int, matched_chrt_ids: Sequence[int], catalog_ids_by_chrt: Mapping[int, str | None], listing_catalog_id_by_chrt: Mapping[int, str | None]) -> bool`

- [ ] **Step 1: Write failing tests**

```python
from datetime import UTC, datetime, timedelta

from markethacker.modules.stock_control.domain import wb_catalog as cat


def test_parse_wb_chrt_id() -> None:
    assert cat.parse_wb_chrt_id("12345") == 12345
    assert cat.parse_wb_chrt_id(" 99 ") == 99
    assert cat.parse_wb_chrt_id("abc") is None
    assert cat.parse_wb_chrt_id("12.3") is None


def test_collect_matched_chrt_ids_dedupes() -> None:
    assert cat.collect_matched_chrt_ids(["10", "bad", "10", "20"]) == [10, 20]


def test_effective_ttl_prefers_org_override() -> None:
    assert cat.effective_wb_content_ttl_hours(org_override=6, platform_default=24) == 6
    assert cat.effective_wb_content_ttl_hours(org_override=None, platform_default=24) == 24


def test_should_refresh_when_never_synced() -> None:
    now = datetime(2026, 8, 25, tzinfo=UTC)
    assert (
        cat.should_refresh_wb_content_catalog(
            catalog_synced_at=None,
            now=now,
            ttl_hours=24,
            catalog_entry_count=0,
            matched_chrt_ids=[1],
            catalog_ids_by_chrt={},
            listing_catalog_id_by_chrt={1: None},
        )
        is True
    )


def test_should_not_refresh_when_fresh_and_complete() -> None:
    now = datetime(2026, 8, 25, tzinfo=UTC)
    synced = now - timedelta(hours=1)
    assert (
        cat.should_refresh_wb_content_catalog(
            catalog_synced_at=synced,
            now=now,
            ttl_hours=24,
            catalog_entry_count=2,
            matched_chrt_ids=[1, 2],
            catalog_ids_by_chrt={1: "100", 2: "200"},
            listing_catalog_id_by_chrt={1: "100", 2: "200"},
        )
        is False
    )


def test_should_refresh_when_ttl_expired() -> None:
    now = datetime(2026, 8, 25, tzinfo=UTC)
    synced = now - timedelta(hours=25)
    assert (
        cat.should_refresh_wb_content_catalog(
            catalog_synced_at=synced,
            now=now,
            ttl_hours=24,
            catalog_entry_count=1,
            matched_chrt_ids=[1],
            catalog_ids_by_chrt={1: "100"},
            listing_catalog_id_by_chrt={1: "100"},
        )
        is True
    )


def test_should_refresh_when_listing_missing_catalog_and_map_misses() -> None:
    now = datetime(2026, 8, 25, tzinfo=UTC)
    synced = now - timedelta(hours=1)
    assert (
        cat.should_refresh_wb_content_catalog(
            catalog_synced_at=synced,
            now=now,
            ttl_hours=24,
            catalog_entry_count=0,
            matched_chrt_ids=[1],
            catalog_ids_by_chrt={},
            listing_catalog_id_by_chrt={1: None},
        )
        is True
    )
```

- [ ] **Step 2: Run tests — expect FAIL**

Run: `uv run pytest tests/unit/stock_control/test_wb_catalog.py -v`  
Expected: import/collection errors until module exists.

- [ ] **Step 3: Implement `wb_catalog.py`**

```python
"""Правила matched chrt и редкого refresh Content-каталога WB."""

from __future__ import annotations

from collections.abc import Mapping, Sequence
from datetime import datetime, timedelta


def parse_wb_chrt_id(mp_sku: str) -> int | None:
    raw = mp_sku.strip()
    if not raw.isdigit():
        return None
    value = int(raw)
    return value if value > 0 else None


def collect_matched_chrt_ids(mp_skus: Sequence[str]) -> list[int]:
    seen: set[int] = set()
    ordered: list[int] = []
    for mp_sku in mp_skus:
        chrt = parse_wb_chrt_id(mp_sku)
        if chrt is None or chrt in seen:
            continue
        seen.add(chrt)
        ordered.append(chrt)
    return ordered


def effective_wb_content_ttl_hours(*, org_override: int | None, platform_default: int) -> int:
    if org_override is not None:
        return org_override
    return platform_default


def should_refresh_wb_content_catalog(
    *,
    catalog_synced_at: datetime | None,
    now: datetime,
    ttl_hours: int,
    catalog_entry_count: int,
    matched_chrt_ids: Sequence[int],
    catalog_ids_by_chrt: Mapping[int, str | None],
    listing_catalog_id_by_chrt: Mapping[int, str | None],
) -> bool:
    if catalog_synced_at is None or catalog_entry_count <= 0:
        return True
    if now - catalog_synced_at >= timedelta(hours=ttl_hours):
        return True
    for chrt in matched_chrt_ids:
        listing_catalog = listing_catalog_id_by_chrt.get(chrt)
        if listing_catalog:
            continue
        mapped = catalog_ids_by_chrt.get(chrt)
        if not mapped:
            return True
    return False
```

- [ ] **Step 4: Run tests — expect PASS**

Run: `uv run pytest tests/unit/stock_control/test_wb_catalog.py -v`

- [ ] **Step 5: Stage for commit (do not commit unless user asked)**

```bash
git add src/markethacker/modules/stock_control/domain/wb_catalog.py tests/unit/stock_control/test_wb_catalog.py
```

---

### Task 2: Миграция и ORM

**Files:**
- Create: `backend/alembic/versions/20260825_0062_wb_content_catalog_cache.py`
- Modify: `backend/src/markethacker/modules/stock_control/domain/models.py`
- Modify: `backend/src/markethacker/infrastructure/database/all_models.py` (если нужен явный импорт)

**Interfaces:**
- Produces ORM: `StockControlWbCatalogEntry`; fields `StockControlApiCredential.catalog_synced_at`, `StockControlOrgSettings.wb_content_catalog_ttl_hours`

- [ ] **Step 1: Add ORM fields and model**

На `StockControlApiCredential` добавить:

```python
catalog_synced_at: Mapped[datetime | None] = mapped_column(
    DateTime(timezone=True),
    nullable=True,
)
```

На `StockControlOrgSettings` добавить (CheckConstraint: null или > 0):

```python
wb_content_catalog_ttl_hours: Mapped[int | None] = mapped_column(Integer, nullable=True)
```

Новая модель:

```python
class StockControlWbCatalogEntry(TimestampMixin, Base):
    __tablename__ = "stock_control_wb_catalog_entries"
    __table_args__ = (
        UniqueConstraint(
            "marketplace_account_id",
            "chrt_id",
            name="uq_stock_control_wb_catalog_account_chrt",
        ),
        Index("ix_stock_control_wb_catalog_account_id", "marketplace_account_id"),
    )
    id: Mapped[uuid.UUID] = mapped_column(primary_key=True, default=uuid.uuid4)
    org_id: Mapped[uuid.UUID] = mapped_column(ForeignKey("organizations.id", ondelete="CASCADE"), nullable=False)
    marketplace_account_id: Mapped[uuid.UUID] = mapped_column(
        ForeignKey("marketplace_accounts.id", ondelete="CASCADE"), nullable=False
    )
    chrt_id: Mapped[int] = mapped_column(Integer, nullable=False)
    nm_id: Mapped[str | None] = mapped_column(String(64), nullable=True)
```

- [ ] **Step 2: Alembic revision `0062`**

`down_revision = "20260825_0061_drop_listing_image_url"` (или актуальный head на момент реализации).

Upgrade: add columns + create table + RLS policies по паттерну соседних stock_control таблиц (org_id).  
Downgrade: drop table + drop columns.

- [ ] **Step 3: Verify migration locally**

Run: `uv run alembic upgrade head` (в окружении с БД) или хотя бы `uv run alembic check` / import models.

- [ ] **Step 4: Stage**

```bash
git add alembic/versions/20260825_0062_wb_content_catalog_cache.py src/markethacker/modules/stock_control/domain/models.py
```

---

### Task 3: Repository + vault timestamps

**Files:**
- Modify: `backend/src/markethacker/modules/stock_control/infrastructure/repository.py`
- Modify: `backend/src/markethacker/modules/stock_control/infrastructure/api_token_vault.py`
- Test: `backend/tests/unit/stock_control/test_repository.py` (дописать) или новый `test_wb_catalog_repo.py` с моком session, если в проекте так принято для repo

**Interfaces:**
- Produces:
  - `StockControlRepository.replace_wb_catalog_entries(account_id, org_id, entries: Mapping[int, str | None]) -> None`
  - `StockControlRepository.list_wb_catalog_map(account_id) -> dict[int, str | None]`
  - `StockControlRepository.count_wb_catalog_entries(account_id) -> int`
  - `ApiTokenVault.mark_catalog_synced(account_id, synced_at: datetime) -> None`
  - credential row exposes `catalog_synced_at`

- [ ] **Step 1: Implement replace as delete-all-for-account + bulk insert in one flush**

- [ ] **Step 2: Implement list/count**

- [ ] **Step 3: Vault `mark_catalog_synced`**

- [ ] **Step 4: Unit test replace+list roundtrip** (если есть DB fixture; иначе тестировать через sync fake в Task 5)

- [ ] **Step 5: Stage**

---

### Task 4: WbSellerAdapter — split Content / stocks

**Files:**
- Modify: `backend/src/markethacker/modules/stock_control/domain/ports.py`
- Modify: `backend/src/markethacker/modules/stock_control/infrastructure/marketplace/wb.py`
- Modify: `backend/src/markethacker/modules/stock_control/infrastructure/marketplace/ozon.py`
- Modify: `backend/tests/unit/stock_control/test_wb_parser.py`

**Interfaces:**
- Produces:
  - `SellerStockPort.fetch_fbs_stocks(self, token: str, *, chrt_ids: Sequence[int] | None = None) -> list[MpStockRow]`
  - `WbSellerAdapter.fetch_content_catalog(self, token: str) -> dict[int, str | None]`
- Ozon: `chrt_ids` игнорируется; поведение как сейчас.

- [ ] **Step 1: Update Protocol signature with optional `chrt_ids`**

- [ ] **Step 2: WB `fetch_content_catalog`** — вынести текущий `_list_card_ids` loop; возвращает `dict[chrt, nm_id]`

- [ ] **Step 3: WB `fetch_fbs_stocks`** — warehouses + stocks **только** по `chrt_ids` (если пусто — вернуть `[]` без POST stocks). `catalog_id` в строках: не заполнять из Content (sync подставит из БД) **или** опционально принимать `catalog_by_chrt` аргумент — проще: sync обогащает после.

Рекомендация: `MpStockRow.catalog_id` остаётся optional; sync выставляет `catalog_id` из map при upsert/enrich.

- [ ] **Step 4: Tests** — HTTP mock: cards list не вызывается из `fetch_fbs_stocks`; stocks body содержит только переданные chrt.

- [ ] **Step 5: Stage**

---

### Task 5: Sync orchestration

**Files:**
- Modify: `backend/src/markethacker/modules/stock_control/application/sync.py`
- Modify: `backend/tests/unit/stock_control/test_sync.py`

**Interfaces:**
- Consumes: Task 1–4
- Для WB: resolve TTL via `get_runtime_value("stock_control_wb_content_catalog_ttl_hours", 24)` + org settings override; refresh catalog; `fetch_fbs_stocks(token, chrt_ids=matched)`

- [ ] **Step 1: Failing sync tests**

Добавить фейковый port с счётчиками:

```python
class CountingWbPort:
    def __init__(self) -> None:
        self.stock_chrt_calls: list[list[int]] = []
        self.content_calls = 0

    async def ping(self, token: str) -> None:
        return None

    async def fetch_fbs_stocks(self, token: str, *, chrt_ids: Sequence[int] | None = None) -> list:
        self.stock_chrt_calls.append(list(chrt_ids or []))
        return []

    async def fetch_fbs_orders(self, token: str, updated_after: datetime) -> list:
        return []

    async def fetch_content_catalog(self, token: str) -> dict[int, str | None]:
        self.content_calls += 1
        return {111: "999"}
```

Кейсы:
1. matched `mp_sku=["111","222"]` → `stock_chrt_calls == [[111, 222]]`
2. свежая карта + заполненные catalog_id → `content_calls == 0` (нужен port с `fetch_content_catalog` и sync, который его зовёт только через helper — для Protocol можно передать WB adapter subclass / duck-typed object)
3. `catalog_synced_at is None` → content вызывается один раз

Практичный путь: sync принимает optional `content_fetcher` callable или проверяет `hasattr(port, "fetch_content_catalog")` только для `marketplace == "wildberries"`.

- [ ] **Step 2: Implement sync changes**

Порядок для WB после load token:

1. Load listings (`list_listings_for_marketplace` org WB) — как сейчас.
2. `matched = collect_matched_chrt_ids(...)`.
3. Load org settings + platform TTL → `ttl`.
4. Load catalog map/count + `credential.catalog_synced_at`.
5. Build `listing_catalog_id_by_chrt` from listings.
6. If `should_refresh...` and hasattr fetch_content_catalog: try refresh; on SellerRetryError log and continue with old map; on auth → existing invalid path.
7. On success: `replace_wb_catalog_entries` + `mark_catalog_synced`.
8. `stock_rows = await port.fetch_fbs_stocks(token, chrt_ids=matched)` (empty matched → []).
9. Enrich listing `catalog_id` from **DB map** (не только из stock_rows).
10. Rest unchanged (orders, snapshots, alerts).

- [ ] **Step 3: Run** `uv run pytest tests/unit/stock_control/test_sync.py tests/unit/stock_control/test_wb_parser.py -v`

- [ ] **Step 4: Stage**

---

### Task 6: Platform default TTL

**Files:**
- Modify: `backend/src/markethacker/modules/platform_settings/application/defaults.py` — `EDITABLE_KEYS` + `defaults_from_env`
- Modify: admin platform settings schemas/router flatten (как другие int keys)
- Modify: `admin-panel/src/lib/api.ts` — `PlatformSettings` / update body
- Modify: `admin-panel/src/app/(admin)/settings/page.tsx` — поле «TTL каталога WB Content (часы)», default 24, min 1

**Interfaces:**
- Produces runtime key `stock_control_wb_content_catalog_ttl_hours: int` (default 24)

- [ ] **Step 1: Backend defaults**

```python
# EDITABLE_KEYS +=
"stock_control_wb_content_catalog_ttl_hours",

# defaults_from_env +=
"stock_control_wb_content_catalog_ttl_hours": 24,
```

Добавить coerce > 0 в существующий pipeline platform settings (как другие int).

- [ ] **Step 2: Expose in admin GET/PATCH platform settings** (найти схему `PlatformSettings` / nested group — положить в логичную группу, например `stockControl` или `general`; если nested — обновить `_flatten_platform_settings_patch`).

- [ ] **Step 3: Admin UI input**

- [ ] **Step 4: Smoke** — GET settings содержит ключ; PATCH 1..N сохраняется.

- [ ] **Step 5: Stage** (backend + admin-panel отдельные репы)

---

### Task 7: Org override в Admin

**Files:**
- Modify: `backend/src/markethacker/modules/admin/api/router.py` + schemas org update
- Modify: `backend/src/markethacker/modules/admin/application/service.py` + repository detail
- Modify: `backend/src/markethacker/modules/stock_control/infrastructure/repository.py` — get/update org settings TTL
- Modify: `admin-panel/src/app/(admin)/organizations/[id]/page.tsx` + `api.ts`

**Interfaces:**
- Org detail includes `wbContentCatalogTtlHours: number | null`
- PATCH org accepts optional `wbContentCatalogTtlHours: number | null` (`null` сброс на platform default)
- **Не** добавлять поле в seller `PATCH /stock-control/settings`

- [ ] **Step 1: Backend read/write through StockControlRepository ensure_settings**

- [ ] **Step 2: Admin API**

- [ ] **Step 3: Admin UI** — поле «TTL каталога WB (часы), пусто = как у платформы»

- [ ] **Step 4: Test** — unit/integration: override 6 → effective 6; null → 24

- [ ] **Step 5: Stage**

---

### Task 8: Docs architecture

**Files:**
- Modify: `docs/architecture/stock-control.md` — секция «Синхронизация»: matched stocks, редкий Content, TTL admin

- [ ] **Step 1: Update sync bullet list to match implemented behavior**

- [ ] **Step 2: Stage in docs repo**

---

## Spec coverage checklist

| Spec requirement | Task |
|------------------|------|
| Matched-only stocks | 4, 5 |
| Rare Content + persist table + catalog_synced_at | 2, 3, 5 |
| TTL platform default 24 | 6 |
| Org override admin-only | 7 |
| No seller TTL UI | 7 (explicit non-goal) |
| Auth errors keep old map; 429 keep old map | 5 |
| Invalid mp_sku skip | 1, 5 |
| Empty matched → no stocks calls | 4, 5 |
| Enrich catalog_id from local map | 5 |
| Architecture doc | 8 |
| Ozon untouched semantics | 4 |

## Self-review notes

- Matching без `account_id`: в плане явно org+marketplace WB listings (согласовано с текущим sync).
- `fetch_content_catalog` не в Protocol — duck-type / WB-only, чтобы не ломать Ozon Protocol.
- Commit steps = stage only unless user asks to commit.

---

Plan complete and saved to `docs/superpowers/plans/2026-08-25-wb-sync-api-reduction.md`. Two execution options:

**1. Subagent-Driven (recommended)** — свежий субагент на задачу, ревью между задачами  

**2. Inline Execution** — выполнение в этой сессии с чекпоинтами  

Which approach?
