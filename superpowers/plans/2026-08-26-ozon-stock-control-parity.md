# Ozon Stock Control Hot-Path Parity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Сделать hot path Ozon зеркалом WB matched-only: stocks только по числовым matched SKU, binding без `offer_id`, доменная проверка Client-Id/Api-Key и понятные подсказки в Manager Portal.

**Architecture:** Sync для `ozon` собирает matched sku из `product_matching` (org + marketplace), передаёт их в `fetch_fbs_stocks(..., chrt_ids=...)` (тот же параметр порта, что и для WB chrtId). Адаптер Ozon больше не обходит `product/list` / `product/info/list`. Парсеры пишут `mp_sku` только из `sku`. Перед vault — `validate_ozon_stock_token`.

**Tech Stack:** Python 3.11+ / FastAPI / httpx / pytest; Manager Portal Next.js / React.

**Spec:** `docs/superpowers/specs/2026-08-26-ozon-stock-control-parity-design.md`

## Global Constraints

- Канон `mp_sku` для Ozon — строка положительного целочисленного SKU Ozon (не `offer_id`).
- Не добавлять каталог / картинки / TTL / Admin-настройки Ozon.
- Не менять cadence 15 мин, advice/alerts, FBO/FBW, Guided Connect.
- Не вызывать `POST /v1/roles`.
- На маркетплейс остатки не писать; PUT запрещён.
- Client-Id / Api-Key и sealed JSON не логировать.
- Тексты продавцу на русском, без путей API и внутренних кодов.
- Нет `any` / `Record<string, any>` / `unknown` без нужды.
- Коммитить только по явной просьбе пользователя (шаги Commit — что класть в индекс; не коммитить самовольно).
- Тесты из `backend/`: `uv run pytest <path> -v`.
- Перед правками кода — согласование с пользователем, если сессия не в режиме «делай по плану».

## File map

Создать:

- `backend/src/markethacker/modules/stock_control/domain/ozon_sku.py` — parse / collect matched sku
- `backend/src/markethacker/modules/stock_control/domain/ozon_token.py` — validate sealed JSON
- `backend/tests/unit/stock_control/test_ozon_sku.py`
- `backend/tests/unit/stock_control/test_ozon_token.py`

Изменить:

- `backend/src/markethacker/modules/stock_control/infrastructure/marketplace/ozon.py` — matched-only stocks; sku-only parsers; удалить hot-path `_list_skus`
- `backend/src/markethacker/modules/stock_control/application/sync.py` — ветка ozon как WB matched + zeroing
- `backend/src/markethacker/modules/stock_control/application/service.py` — validate перед vault для ozon
- `backend/tests/unit/stock_control/test_ozon_parser.py` — sku-only; mock fetch matched-only
- `backend/tests/unit/stock_control/test_sync.py` — ozon matched-only + zeroing
- `manager-portal/src/app/(manager)/accounts/[id]/page.tsx` — заголовок, description, инструкция Ozon
- `manager-portal/src/components/stock-control-add-products-modal.tsx` — подсказка про числовой SKU
- `docs/architecture/stock-control.md` — § Ozon

Не трогать: WB Content/TTL, Admin Panel, Alembic, `ports.py` сигнатуру (параметр остаётся `chrt_ids`).

---

### Task 1: Domain helpers — Ozon SKU

**Files:**
- Create: `backend/src/markethacker/modules/stock_control/domain/ozon_sku.py`
- Test: `backend/tests/unit/stock_control/test_ozon_sku.py`

**Interfaces:**
- Produces:
  - `parse_ozon_sku(mp_sku: str) -> int | None`
  - `collect_matched_ozon_skus(mp_skus: Sequence[str]) -> list[int]` — уникальные, стабильный порядок первого появления
- Consumes: ничего

- [ ] **Step 1: Write failing tests**

```python
from markethacker.modules.stock_control.domain import ozon_sku as oz


def test_parse_ozon_sku() -> None:
    assert oz.parse_ozon_sku("1001") == 1001
    assert oz.parse_ozon_sku(" 99 ") == 99
    assert oz.parse_ozon_sku("0") is None
    assert oz.parse_ozon_sku("-1") is None
    assert oz.parse_ozon_sku("abc") is None
    assert oz.parse_ozon_sku("12.3") is None
    assert oz.parse_ozon_sku("art-1") is None


def test_collect_matched_ozon_skus_dedupes() -> None:
    assert oz.collect_matched_ozon_skus(["10", "bad", "10", "20", "0"]) == [10, 20]
```

- [ ] **Step 2: Run tests — expect FAIL (module missing)**

Run: `cd backend && uv run pytest tests/unit/stock_control/test_ozon_sku.py -v`  
Expected: FAIL import / module not found

- [ ] **Step 3: Implement**

```python
"""Парсинг matched Ozon SKU для контроля остатков."""

from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from collections.abc import Sequence


def parse_ozon_sku(mp_sku: str) -> int | None:
    raw = mp_sku.strip()
    if not raw.isdigit():
        return None
    value = int(raw)
    if value <= 0:
        return None
    return value


def collect_matched_ozon_skus(mp_skus: Sequence[str]) -> list[int]:
    seen: set[int] = set()
    ordered: list[int] = []
    for mp_sku in mp_skus:
        sku = parse_ozon_sku(mp_sku)
        if sku is None or sku in seen:
            continue
        seen.add(sku)
        ordered.append(sku)
    return ordered
```

- [ ] **Step 4: Run tests — expect PASS**

Run: `cd backend && uv run pytest tests/unit/stock_control/test_ozon_sku.py -v`  
Expected: PASS

- [ ] **Step 5: Commit (только по просьбе пользователя)**

```bash
cd backend
git add src/markethacker/modules/stock_control/domain/ozon_sku.py \
  tests/unit/stock_control/test_ozon_sku.py
git commit -m "feat(stock_control): parse matched Ozon SKUs"
```

---

### Task 2: Domain — validate Ozon token + wire `put_api_token`

**Files:**
- Create: `backend/src/markethacker/modules/stock_control/domain/ozon_token.py`
- Modify: `backend/src/markethacker/modules/stock_control/application/service.py` (ветка `put_api_token` рядом с WB validate)
- Test: `backend/tests/unit/stock_control/test_ozon_token.py`

**Interfaces:**
- Produces: `validate_ozon_stock_token(sealed: str) -> str | None` — текст ошибки или `None`
- Consumes: sealed JSON `{"clientId","apiKey"}`

- [ ] **Step 1: Write failing tests**

```python
from markethacker.modules.stock_control.domain.ozon_token import validate_ozon_stock_token


def test_validate_ozon_stock_token_accepts_valid() -> None:
    assert validate_ozon_stock_token('{"clientId":"c1","apiKey":"k1"}') is None
    assert validate_ozon_stock_token('{"clientId":" c1 ","apiKey":" k1 "}') is None


def test_validate_ozon_stock_token_rejects_malformed() -> None:
    assert validate_ozon_stock_token("plain") is not None
    assert validate_ozon_stock_token("{") is not None
    assert validate_ozon_stock_token("[]") is not None
    assert validate_ozon_stock_token('{"clientId":"","apiKey":"k"}') is not None
    assert validate_ozon_stock_token('{"clientId":"c","apiKey":""}') is not None
    assert validate_ozon_stock_token('{"clientId":1,"apiKey":"k"}') is not None
    assert validate_ozon_stock_token('{"apiKey":"k"}') is not None
```

Сообщения — непустые русские строки (точный текст в реализации; assert `is not None` / `in` достаточно).

- [ ] **Step 2: Run — expect FAIL**

Run: `cd backend && uv run pytest tests/unit/stock_control/test_ozon_token.py -v`  
Expected: FAIL import

- [ ] **Step 3: Implement `ozon_token.py`**

```python
"""Проверка sealed-токена Ozon для контроля остатков.

Формат vault: JSON {"clientId","apiKey"} (camelCase).
"""

from __future__ import annotations

import json

_MSG_MALFORMED = (
    "Ключ выглядит неверно. Укажите Client-Id и Api-Key из кабинета продавца Ozon."
)
_MSG_EMPTY_CLIENT = "Укажите Client-Id из кабинета продавца Ozon."
_MSG_EMPTY_KEY = "Укажите Api-Key из кабинета продавца Ozon."


def validate_ozon_stock_token(sealed: str) -> str | None:
    """Проверяет форму sealed JSON до ping.

    Returns:
        Текст ошибки для пользователя или None, если форма подходит.
    """
    try:
        payload = json.loads(sealed)
    except json.JSONDecodeError:
        return _MSG_MALFORMED
    if not isinstance(payload, dict):
        return _MSG_MALFORMED
    client_id = payload.get("clientId")
    api_key = payload.get("apiKey")
    if not isinstance(client_id, str):
        return _MSG_MALFORMED if "clientId" not in payload else _MSG_EMPTY_CLIENT
    if not isinstance(api_key, str):
        return _MSG_MALFORMED if "apiKey" not in payload else _MSG_EMPTY_KEY
    if not client_id.strip():
        return _MSG_EMPTY_CLIENT
    if not api_key.strip():
        return _MSG_EMPTY_KEY
    return None
```

Упростить ветки типов при желании: любой не-string / missing → `_MSG_MALFORMED` или отдельные empty — главное: тесты PASS и тексты русские без API-путей.

- [ ] **Step 4: Wire `service.py`**

В `put_api_token` после WB-блока добавить:

```python
from markethacker.modules.stock_control.domain.ozon_token import validate_ozon_stock_token

# ...
if account.marketplace == "wildberries":
    reject = validate_personal_stock_token(sealed)
    if reject is not None:
        raise ValidationError(reject)
elif account.marketplace == "ozon":
    reject = validate_ozon_stock_token(sealed)
    if reject is not None:
        raise ValidationError(reject)
```

- [ ] **Step 5: Run token tests — expect PASS**

Run: `cd backend && uv run pytest tests/unit/stock_control/test_ozon_token.py -v`  
Expected: PASS

- [ ] **Step 6: Commit (только по просьбе)**

```bash
cd backend
git add src/markethacker/modules/stock_control/domain/ozon_token.py \
  src/markethacker/modules/stock_control/application/service.py \
  tests/unit/stock_control/test_ozon_token.py
git commit -m "feat(stock_control): validate Ozon Client-Id and Api-Key shape"
```

---

### Task 3: Adapter — matched-only stocks + sku-only parsers

**Files:**
- Modify: `backend/src/markethacker/modules/stock_control/infrastructure/marketplace/ozon.py`
- Modify: `backend/tests/unit/stock_control/test_ozon_parser.py`
- Optional fixture tweak only if нужны кейсы offer_id-only

**Interfaces:**
- Consumes: `chrt_ids: Sequence[int] | None` — для Ozon это список matched Ozon SKU
- Produces: `fetch_fbs_stocks` без `_list_skus`; `parse_ozon_stocks` / `parse_ozon_orders` с `mp_sku` только из `sku`

- [ ] **Step 1: Write / extend failing tests**

В `test_ozon_parser.py` добавить:

```python
import respx  # только если уже принят в проекте для WB; иначе httpx MockTransport / ASGI — смотри test_wb_parser.py

# Минимально без HTTP: парсер

def test_parse_ozon_stocks_ignores_offer_id_without_sku() -> None:
    rows = parse_ozon_stocks(
        {
            "result": [
                {
                    "offer_id": "art-only",
                    "present": 5,
                    "warehouse_id": 501,
                    "warehouse_name": "FBS",
                    "warehouse_type": "fbs",
                }
            ]
        }
    )
    assert rows == []


def test_parse_ozon_orders_ignores_line_without_sku() -> None:
    lines = parse_ozon_orders(
        {
            "result": {
                "postings": [
                    {
                        "posting_number": "1-1",
                        "status": "awaiting_deliver",
                        "products": [{"offer_id": "art-1", "quantity": 2}],
                        "analytics_data": {"warehouse_id": 501},
                    }
                ]
            }
        }
    )
    assert lines == []
```

Для HTTP matched-only — скопировать паттерн из `test_wb_parser.py` (если там `respx` / `httpx.MockTransport`):

```python
@pytest.mark.asyncio
async def test_fetch_fbs_stocks_requests_only_given_skus(httpx_mock_or_respx) -> None:
    # Arrange: при chrt_ids=[1001, 1002] ожидается один или два POST
    # /v2/product/info/stocks-by-warehouse/fbs с body.sku == ["1001","1002"] (чанки)
    # и НОЛЬ вызовов /v3/product/list и /v3/product/info/list
    # при chrt_ids=[] — ноль stock-запросов, return []
```

Если HTTP-тест слишком дорог в этой сессии: обязательны парсер-тесты + sync-тесты Task 4; HTTP — желателен.

- [ ] **Step 2: Run parser new tests — expect FAIL (offer_id ещё матчится)**

Run: `cd backend && uv run pytest tests/unit/stock_control/test_ozon_parser.py -v`  
Expected: FAIL на новых assert

- [ ] **Step 3: Implement adapter + parsers**

`fetch_fbs_stocks`:

```python
async def fetch_fbs_stocks(
    self,
    token: str,
    *,
    chrt_ids: Sequence[int] | None = None,
) -> list[MpStockRow]:
    requested = list(chrt_ids or ())
    if not requested:
        return []
    rows: list[MpStockRow] = []
    for chunk in _chunks(requested, _STOCKS_CHUNK):
        body = await self._json(
            "POST",
            "/v2/product/info/stocks-by-warehouse/fbs",
            token=token,
            json_body={"sku": [str(sku) for sku in chunk]},
            stocks=True,
        )
        rows.extend(parse_ozon_stocks(body))
    return rows
```

Удалить метод `_list_skus` и неиспользуемые константы, если больше нигде не нужны (`_PRODUCT_LIMIT` — только для list).

В `_rows_from_flat`, `_rows_from_products`, `parse_ozon_orders`:

```python
mp_sku = _as_str(item.get("sku"))
# без: or _as_str(item.get("offer_id"))
```

- [ ] **Step 4: Run parser tests — expect PASS**

Run: `cd backend && uv run pytest tests/unit/stock_control/test_ozon_parser.py -v`  
Expected: PASS (включая старые: `mp_sku == "1001"`)

- [ ] **Step 5: Commit (только по просьбе)**

```bash
cd backend
git add src/markethacker/modules/stock_control/infrastructure/marketplace/ozon.py \
  tests/unit/stock_control/test_ozon_parser.py
git commit -m "feat(stock_control): Ozon matched-only FBS stocks by SKU"
```

---

### Task 4: Sync — Ozon matched-only + zeroing

**Files:**
- Modify: `backend/src/markethacker/modules/stock_control/application/sync.py`
- Modify: `backend/tests/unit/stock_control/test_sync.py`

**Interfaces:**
- Consumes: `collect_matched_ozon_skus`, `parse_ozon_sku` из Task 1; `port.fetch_fbs_stocks(..., chrt_ids=...)`
- Produces: для `marketplace == "ozon"` — передача matched skus; zeroing только по запрошенным

- [ ] **Step 1: Write failing sync tests**

По образцу `test_wb_stocks_are_requested_only_for_matched_chrt_ids` и `test_wb_zeroes_only_snapshots_for_requested_chrt_ids`:

```python
@pytest.mark.asyncio
async def test_ozon_stocks_are_requested_only_for_matched_skus() -> None:
    # org с ozon listings mp_sku "1001", "bad", "1001"
    # TrackingSellerPort записывает stock_chrt_calls
    # sync_account(..., marketplace ozon)
    # assert port.stock_chrt_calls == [[1001]]


@pytest.mark.asyncio
async def test_ozon_empty_matching_skips_stocks() -> None:
    # listings пусты или только невалидные mp_sku
    # assert stock_chrt_calls == [[]] или вызов с [] и adapter/Fake возвращает []
    # Фактически: один вызов с chrt_ids=[] ИЛИ sync не вызывает stocks —
    # выровнять с реализацией: предпочтительно вызывать fetch с [] как WB,
    # либо не вызывать — тогда Fake должен считать вызовы. Смотри WB-тест и копируй.


@pytest.mark.asyncio
async def test_ozon_zeroes_only_snapshots_for_requested_skus() -> None:
    # seed snapshot для sku A (matched) и B (не в matched / невалидный)
    # sync с stocks без строки для A → A listed_qty=0; B не трогать (или B не в zeroing)
```

Использовать существующие фабрики/`FakeSellerPort` / `Tracking`-класс из файла (`stock_chrt_calls`).

- [ ] **Step 2: Run — expect FAIL**

Run: `cd backend && uv run pytest tests/unit/stock_control/test_sync.py -k ozon_stocks -v`  
Expected: FAIL

- [ ] **Step 3: Implement sync branch**

Импорт:

```python
from markethacker.modules.stock_control.domain.ozon_sku import (
    collect_matched_ozon_skus,
    parse_ozon_sku,
)
```

После сбора `listings`, рядом с WB:

```python
matched_ozon_skus: list[int] = []
if cabinet.marketplace == "ozon":
    matched_ozon_skus = collect_matched_ozon_skus([listing.mp_sku for listing in listings])
```

Замена ветки stocks:

```python
if cabinet.marketplace == "wildberries":
    stock_rows = await port.fetch_fbs_stocks(token, chrt_ids=matched_chrt_ids)
elif cabinet.marketplace == "ozon":
    stock_rows = await port.fetch_fbs_stocks(token, chrt_ids=matched_ozon_skus)
else:
    stock_rows = await port.fetch_fbs_stocks(token)
```

(Сейчас `else` вызывает без ids — после изменения `else` может не понадобиться, если marketplace только wb|ozon.)

Zeroing:

```python
zeroing_bindings = bindings
if cabinet.marketplace == "wildberries":
    requested_chrt_ids = set(matched_chrt_ids)
    zeroing_bindings = [
        binding
        for binding in bindings
        if (chrt_id := parse_wb_chrt_id(binding.mp_sku)) is not None
        and chrt_id in requested_chrt_ids
    ]
elif cabinet.marketplace == "ozon":
    requested_skus = set(matched_ozon_skus)
    zeroing_bindings = [
        binding
        for binding in bindings
        if (sku := parse_ozon_sku(binding.mp_sku)) is not None and sku in requested_skus
    ]
```

- [ ] **Step 4: Run sync tests — expect PASS**

Run: `cd backend && uv run pytest tests/unit/stock_control/test_sync.py -k "ozon_stocks or ozon_zeroes or ozon_empty" -v`  
и полный `test_sync.py` чтобы не сломать WB/смешанные кейсы:

Run: `cd backend && uv run pytest tests/unit/stock_control/test_sync.py -v`  
Expected: PASS

- [ ] **Step 5: Commit (только по просьбе)**

```bash
cd backend
git add src/markethacker/modules/stock_control/application/sync.py \
  tests/unit/stock_control/test_sync.py
git commit -m "feat(stock_control): sync Ozon stocks only for matched SKUs"
```

---

### Task 5: Manager Portal — ключ и подсказка matching

**Files:**
- Modify: `manager-portal/src/app/(manager)/accounts/[id]/page.tsx`
- Modify: `manager-portal/src/components/stock-control-add-products-modal.tsx`

**Interfaces:**
- Consumes: существующий save JSON `{clientId, apiKey}`
- Produces: copy для продавца

- [ ] **Step 1: Обновить карточку Ozon в `OfficialMarketplaceKeyCard`**

- `title`: «Ключ Seller API Ozon» (вместо «Ключ площадки»).
- `description`: ключ из кабинета Ozon (Client-Id и Api-Key); не заменяет вход в кабинет; нужен для остатков и заказов FBS.
- Добавить `<ol>` с шагами (зеркало WB-списка), без названий endpoint’ов, например:
  1. В кабинете Ozon откройте раздел API / Seller API и создайте ключ.
  2. Скопируйте Client-Id и Api-Key.
  3. Ключ должен позволять читать склады, остатки FBS и заказы FBS.
  4. Вставьте значения ниже и нажмите «Сохранить».

Точные формулировки — коротко, без маркетинговой воды (см. `no-ai-style-text`).

- [ ] **Step 2: Подсказка в модалке добавления товаров**

Рядом с колонкой Ozon / в description модалки одна фраза:

«Артикул Ozon — числовой SKU из кабинета Ozon, не артикул продавца.»

Не менять API commit/parse и имена полей.

- [ ] **Step 3: Ручная проверка**

- Открыть `/accounts/{ozonId}` — видны заголовок, инструкция, два поля.
- Открыть «Добавить товары» на `/stock-control` — видна подсказка про числовой SKU.

- [ ] **Step 4: Commit (только по просьбе)**

```bash
cd manager-portal
git add src/app/\(manager\)/accounts/\[id\]/page.tsx \
  src/components/stock-control-add-products-modal.tsx
git commit -m "fix(stock-control): clarify Ozon API key and numeric SKU copy"
```

---

### Task 6: Architecture doc

**Files:**
- Modify: `docs/architecture/stock-control.md` (§ Ozon, сейчас строки про «без matched-only»)

- [ ] **Step 1: Заменить § Ozon**

Текст по смыслу:

### Ozon

`mp_sku` — числовой SKU Ozon (положительное целое в строке). Невалидные значения пропускаются.

**Hot path:**

1. `ping` через `POST /v1/warehouse/list`.
2. `POST /v2/product/info/stocks-by-warehouse/fbs` только по matched sku из `product_matching` (org + marketplace `ozon`), чанками. Если matched пуст — запросов stocks нет.
3. Инкрементальные FBS-заказы с `orders_cursor_at`.
4. Снимки; `listed_qty = 0` только для запрошенных склад/SKU, отсутствующих в ответе.
5. Binding снимков и заказов по полю `sku` (не по `offer_id`).

Content-каталог / картинки Ozon в v1 нет. Полный обход `product/list` на sync не выполняется.

- [ ] **Step 2: Commit (только по просьбе)**

```bash
cd docs
git add architecture/stock-control.md
git commit -m "docs(stock-control): document Ozon matched-only sync"
```

---

## Self-review (plan vs spec)

| Spec requirement | Task |
|------------------|------|
| Matched-only stocks by numeric sku | 1, 3, 4 |
| Remove product/list + info/list from hot path | 3 |
| mp_sku / orders from sku only | 3 |
| Zeroing requested set only | 4 |
| validate_ozon_stock_token before vault | 2 |
| Portal key instructions | 5 |
| Matching hint: numeric SKU not offer_id | 5 |
| architecture/stock-control.md | 6 |
| No catalog/TTL/roles/Guided Connect | Global + file map |
| Tests token / parser / sync | 1–4 |

Placeholder scan: нет TBD. Параметр порта остаётся `chrt_ids` (для Ozon = sku ids) — явно в File map и Task 3/4.

---

## Execution handoff

Plan complete and saved to `docs/superpowers/plans/2026-08-26-ozon-stock-control-parity.md`.

**Two execution options:**

1. **Subagent-Driven (recommended)** — fresh subagent per task, review between tasks  
2. **Inline Execution** — this session, executing-plans with checkpoints  

Which approach?
