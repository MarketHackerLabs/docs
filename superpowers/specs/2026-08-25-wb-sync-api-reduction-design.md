# Design: снижение объёма WB API в контроле остатков

Дата: 2026-08-25  
Статус: approved for planning  
Модуль: `stock_control` (Wildberries)

## Проблема

Сейчас каждый sync кабинета WB (каждые 15 минут) делает:

1. `GET` Marketplace `/ping` и Content `/ping`
2. `GET /api/v3/warehouses`
3. Полный обход Content `POST /content/v2/get/cards/list` (весь каталог продавца)
4. `POST /api/v3/stocks/{warehouseId}` по **всем** `chrtId` каталога × каждый FBS-склад
5. Инкрементальные FBS-заказы по `orders_cursor_at`

Лимиты токена WB — **на кабинет** (один токен ↔ один кабинет). Конкуренции между кабинетами по квоте нет. Узкое место — рост числа SKU/карточек **внутри** одного кабинета: полный Content и stocks по всему каталогу растут быстрее, чем число товаров в `product_matching`.

Карта `chrtId → nmID` каждый цикл строится в памяти и не персистится (кроме точечного `catalog_id` на сматченных листингах).

## Цель

Сократить число запросов к Content и Marketplace stocks пропорционально **tracked** SKU (matching), а не размеру всего каталога WB. Сохранить корректность снимков, рекомендаций и картинок на доске.

## Вне scope

- Изменение cadence sync (15 минут)
- Межкабинетный pacing / jitter планировщика
- Оптимизация Ozon в этом же изменении (отдельное решение при необходимости)
- Настройка TTL Content в UI продавца (manager-portal)
- Смена бизнес-правил advice/alerts

## Решение (выбранный подход)

**Matched-only stocks + редкий Content с персистом и TTL.**

### Hot path (каждые 15 минут)

1. По необходимости лёгкий ping (допускается оставить текущие 2×`/ping`; не цель оптимизации №1).
2. `GET warehouses` → только FBS.
3. Собрать список `chrtIds` из **listings `product_matching` этого кабинета** (`mp_sku`, парсимый как int).
4. `POST stocks/{warehouseId}` чанками **только по этому списку**.
5. Заказы с `orders_cursor_at` — без изменения семантики.
6. Upsert snapshots. Для складов/SKU, которые запрашивались и не вернулись в ответе → `listed_qty = 0` (zeroing только в запрошенном множестве).
7. Обогащение `listing.catalog_id` из **локальной** карты chrt→nm, если `nmID` известен.

Если matched listings пусты — stocks к WB не вызываем; sync заказов/алертов по правилам модуля не ломаем.

### Content (редко)

Полный обход `cards/list` и запись карты, если выполняется хотя бы одно:

- локальной карты нет или она пуста;
- `catalog_synced_at` старше эффективного TTL;
- у matched листингов кабинета есть пустой `catalog_id`, и в карте нет нужных chrt (догон для картинок).

При ошибке Content:

- 401/403 → как сейчас invalid token; **старую карту не затираем**;
- 429/5xx → оставляем старую карту, stocks по matching всё равно выполняем; UI stale уже покрывает устаревание снимков.

Невалидный `mp_sku` (не int) — пропускаем с учётом в логах/счётчике, sync не валим.

### Эффективный TTL

```
effective_ttl_hours = org.wb_content_catalog_ttl_hours
    if org.wb_content_catalog_ttl_hours is not null
    else platform.stock_control_wb_content_catalog_ttl_hours
```

Дефолт платформы: **24**.

## Данные

### `stock_control_api_credentials`

Добавить:

| Поле | Смысл |
|------|--------|
| `catalog_synced_at` | Момент последнего успешного полного Content refresh (nullable) |

### Новая таблица `stock_control_wb_catalog_entries`

| Поле | Смысл |
|------|--------|
| `marketplace_account_id` | Кабинет |
| `chrt_id` | int, размер WB |
| `nm_id` | строка/bigint nmID |
| unique `(marketplace_account_id, chrt_id)` | |

При успешном полном refresh: заменить набор строк кабинета атомарно (delete+insert в транзакции или upsert+delete orphans). Таблица строк предпочтительнее одного JSONB на credential: проще чистить и точечно читать.

### `stock_control_org_settings`

Добавить nullable:

| Поле | Смысл |
|------|--------|
| `wb_content_catalog_ttl_hours` | Override TTL; `NULL` = взять platform default; если задано — `> 0` |

Поле **не** входит в PATCH `/organizations/{org_id}/stock-control/settings` продавца.

### Platform settings

Ключ в editable platform settings (Admin Panel), например:

`stock_control_wb_content_catalog_ttl_hours` = 24

Org override правится в Admin Panel (карточка org / блок stock control), по аналогии с другими admin overrides, не в manager-portal.

## Изменения адаптера

`WbSellerAdapter`:

- `fetch_fbs_stocks(token, *, chrt_ids: Sequence[int])` — склады + stocks только по переданным id; **не** тянет Content внутри.
- Отдельный метод полного каталога, например `fetch_content_catalog(token) -> dict[int, str | None]` (chrt→nm), вызываемый sync только при условиях refresh.

Порт `SellerStockPort` согласовать так, чтобы Ozon не ломался (chrt_ids только для WB-пути в application sync, либо optional kwargs / отдельный WB-порт).

## Поток sync (application)

Для WB-кабинета:

1. Загрузить matched listings кабинета.
2. Решить, нужен ли Content refresh (TTL / пустая карта / дыры `catalog_id`).
3. При необходимости — `fetch_content_catalog`, записать entries + `catalog_synced_at`.
4. `fetch_fbs_stocks(token, chrt_ids=matched)`.
5. Orders + ledger + snapshots + alerts — как сейчас.
6. Проставить `catalog_id` из локальной карты.

## Admin / UX

- Admin: редактирование platform default TTL.
- Admin: опциональный override TTL на org.
- Manager Portal продавца: без новых полей TTL; инструкция по токену без изменений по смыслу.
- Пользовательские ошибки API без жаргона (как сейчас).

## Тесты

- Unit: из matching собираются только валидные chrt; Content не вызывается, если карта свежая и дыр нет; вызывается при протухшем TTL / пустой карте / дырах `catalog_id`.
- Sync fake port: stocks получают только matched ids; zeroing только среди запрошенных.
- Coerce/validation TTL platform и org (`> 0`, null override).
- Регрессия: orders cursor и advice на снимках не меняют семантику.

## Метрики успеха

На кабинете с большим каталогом WB и небольшим matching:

- число Content-страниц на sync → ~0 на большинстве циклов;
- число stocks-запросов → `W_fbs × ceil(matched_chrt / 1000)` вместо полного каталога;
- рекомендации и картинки на доске остаются корректными при TTL 24h и догоне по пустым `catalog_id`.

## Риски

| Риск | Митигация |
|------|-----------|
| Новый размер на WB без обновления matching | Не попадёт в stocks до импорта matching — ожидаемо для matched-only |
| Новый chrt у уже сматченного nm до refresh Content | Картинка/nm могут отставать до TTL или догона по пустому `catalog_id` |
| Рост таблицы catalog entries | Одна строка на chrt кабинета; полный refresh перезаписывает набор |

## Связанные документы

- `docs/architecture/stock-control.md` — обновить секцию синхронизации после реализации
- `docs/superpowers/specs/2026-08-24-fbs-stock-control-design.md` — исходный дизайн модуля
