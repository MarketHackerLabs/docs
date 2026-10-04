# Design: паритет Ozon в контроле остатков (hot path)

Дата: 2026-08-26  
Статус: approved for planning  
Модуль: `stock_control` (Ozon) + UI ключа / matching в Manager Portal

## Проблема

Адаптер Ozon уже читает FBS-остатки и заказы, но на каждом sync:

1. Полный обход каталога (`POST /v3/product/list` → `POST /v3/product/info/list`) ради списка `sku`.
2. `POST /v2/product/info/stocks-by-warehouse/fbs` по **всем** sku кабинета, а не по matching.
3. В снимках и заказах `mp_sku` предпочитает `sku`, но при отсутствии `sku` падает на `offer_id` — риск рассинхрона с matching.
4. Нет доменной валидации sealed-токена перед vault (в отличие от WB JWT).
5. Инструкции в UI не фиксируют, что «артикул Ozon» = числовой SKU площадки.

Лимиты Seller API растут с размером каталога кабинета, а tracked SKU — с `product_matching`. Картинки / Content-каталог Ozon в этом изменении не решаем.

## Цель

Сделать hot path Ozon зеркалом WB matched-only: запросы stocks пропорциональны matching, корректный binding по числовому `sku`, понятное сохранение ключа и подсказки в UI.

## Вне scope

- Каталог / картинки Ozon, TTL, Admin-настройки для Ozon
- Guided Connect / portal session для Ozon
- Смена cadence sync (15 минут)
- FBO / FBW
- Проверка ролей ключа через `POST /v1/roles`
- Миграция существующих matching с `offer_id` на `sku` (продавец правит вручную)
- Смена бизнес-правил advice / alerts

## Решение (выбранный подход)

**Hot-path parity:** matched-only stocks по числовому Ozon `sku` + доменная проверка Client-Id/Api-Key + инструкции в UI. Без каталога и TTL.

### Канонический идентификатор

`mp_sku` для marketplace `ozon` — строка **положительного целочисленного SKU Ozon** (аналог WB `chrtId`).

- Matching / модалка «Добавить товары»: колонка «артикул Ozon» = числовой SKU, **не** `offer_id` продавца.
- Невалидные значения пропускаются; sync не валится.
- Существующие строки matching с `offer_id` не матчятся, пока продавец не заменит на SKU.

### Hot path (каждые 15 минут, кабинет ozon с активным токеном)

1. `ping` — `POST /v1/warehouse/list` (без изменений семантики).
2. Собрать matched sku из `product_matching` (`org_id` + marketplace `ozon`), парсинг как положительное int.
3. `POST /v2/product/info/stocks-by-warehouse/fbs` чанками **только по этому списку** (`sku: [...]`).  
   Полный обход `product/list` + `product/info/list` с hot path **удалить**.
4. Если matched пуст — stocks не вызываем; заказы и алерты обрабатываются как обычно.
5. Парсер снимков: `mp_sku` только из поля `sku`. FBO/FBW отбрасываются. Строки без `sku` не попадают в снимки.
6. Zeroing `listed_qty = 0` только для запрошенных пар склад/SKU, отсутствующих в ответе (зеркало WB).
7. Заказы: курсор без изменений; для binding в линиях использовать только `sku` (без fallback на `offer_id`).

Документация Seller API: запрос stocks принимает `sku` и/или `offer_id`; при обоих приоритет у `sku`. Мы передаём только `sku`.

### Токен

Sealed формат без изменений: JSON `{"clientId","apiKey"}` в `stock_control_api_credentials`.

Новый модуль `ozon_token.py` (зеркало `wb_token.py`):

- `validate_ozon_stock_token(sealed) -> str | None` (или аналог reject-сообщения).
- Отклонять: битый JSON, не-object, пустые / не-строковые `clientId`/`apiKey`.
- Сообщения на русском, без путей API и внутренних кодов.
- `POST /v1/roles` в v1 не вызываем.

`put_api_token` для Ozon: validate → vault → `ping`. 401/403 → invalid + понятный текст продавцу.

### Manager Portal

Карточка ключа на `/accounts/{id}` для Ozon:

- Заголовок в духе «Ключ Seller API Ozon».
- Краткая инструкция: где взять Client-Id и Api-Key; ключ не заменяет вход в кабинет; нужны права на склады, остатки FBS и заказы FBS (без названий endpoint’ов).
- Поля ввода без смены потока.

Модалка / подсказки matching: явно указать, что «артикул Ozon» — числовой SKU из кабинета Ozon.

## Данные

Схема БД не меняется. Новых таблиц и TTL-полей нет.

## Ошибки и краевые случаи

| Случай | Поведение |
|--------|-----------|
| Stocks 401/403 | Invalid token, как сейчас |
| Stocks 4xx (не auth) | `SellerStocksUnsupportedError` → fallback оценки по заказам |
| 429/5xx | retry / `SellerRetryError` по текущим правилам адаптера |
| Ответ stocks без `sku` | строка пропускается |
| Заказ без `sku` | линия не создаёт движение по matching |
| Matching с `offer_id` | не матчится (ожидаемо) |

## Тесты

- Unit: `validate_ozon_stock_token` (валидный JSON, пустые поля, битый JSON).
- Unit: парсинг matched sku; парсеры stocks/orders — `mp_sku` только из `sku`.
- Sync: пустой matching → stocks не вызываются; zeroing только по запрошенным sku; невалидный `mp_sku` пропускается.
- Минимальный HTTP-мок адаптера: stocks с переданным списком sku, без `product/list`.

## Документация

Обновить § Ozon в `docs/architecture/stock-control.md`: matched-only по числовому sku, без Content-каталога и без полного обхода каталога на hot path.

## Затронутые области (для плана)

| Область | Файлы (ориентир) |
|---------|------------------|
| Adapter | `stock_control/infrastructure/marketplace/ozon.py` |
| Sync | `stock_control/application/sync.py` (+ helpers парсинга sku, зеркало `wb_catalog.parse_wb_chrt_id`) |
| Token | новый `stock_control/domain/ozon_token.py`; `application/service.py` |
| Portal | `accounts/[id]/page.tsx`; подсказки в add-products / draft |
| Docs | `architecture/stock-control.md` |
| Tests | `test_ozon_parser.py`, новый `test_ozon_token.py`, sync-тесты Ozon matched-only |

## Критерии готовности

1. Sync Ozon не вызывает `product/list` / `product/info/list` на hot path.
2. Stocks запрашиваются только по matched числовым sku; пустой matching → 0 stock-запросов.
3. Снимки и order lines bindятся по `sku`, не по `offer_id`.
4. Zeroing ограничен запрошенным множеством.
5. Сохранение ключа отклоняет невалидный sealed JSON до ping с понятным текстом.
6. UI объясняет Client-Id/Api-Key и что артикул Ozon = числовой SKU.
7. Архитектурный документ отражает новую семантику Ozon.
