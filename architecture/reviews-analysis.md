# Анализ отзывов и вопросов по товарам маркетплейсов

## Назначение

Модуль `reviews_analysis` запускает AI-анализ отзывов и вопросов по одной или нескольким номенклатурам.

Сырые отзывы **не хранятся**: backend загружает их с маркетплейса на время job, считает статистику, делает выборку для LLM и сохраняет только агрегированный результат.

## Поток

```mermaid
flowchart LR
  Client -->|nomenclatures| API
  API -->|reserve quota + enqueue| ARQ
  ARQ --> Fetch[WB/Lamoda/Ozon adapter]
  Fetch --> Pipeline[stats + sample + LLM]
  Pipeline --> PG[(result meta)]
  Fetch -.->|discard| X[нет хранения отзывов]
```

1. `POST /api/v1/reviews-analyses` — reserve квоты, `202`, ARQ job.
2. Worker: fetch → gender → aggregate LLM → optional per-item → finalize → commit квоты.
3. `GET /api/v1/reviews-analyses/marketplaces` — включённые маркетплейсы для формы создания.
4. `GET /api/v1/reviews-analyses/active` — виджет активных анализов.
5. `GET /api/v1/reviews-analyses/{id}` — статус / прогресс (без результата).
6. `GET /api/v1/reviews-analyses/{id}/result` — полный отчёт (`completed`).
6. `POST /api/v1/reviews-analyses/{id}/cancel` — отмена.

## Интеграция в Manager Portal

UI: Manager Portal → Анализ отзывов (`/reviews-analysis`).

- Список, создание, карточка со статусом и отчётом.
- Виджет активного анализа в оболочке портала (пока идёт обработка).
- Доступ по фиче тарифа `reviews_analysis` (`session.planFeatures`).
- Только клиентский API из таблицы выше; admin journal — отдельно в Admin Panel.

## Интеграция в браузерное расширение

Auth — как у остальных API расширения: `Authorization: Bearer <accessToken>`.  
Обновление токена — `POST /api/v1/auth/refresh`. Подробнее: [authentication.md](./authentication.md).

JSON — camelCase. Обёртка: `{ "data": ... }` или paginated `{ "items", "total", "page", "pageSize" }`.

### Рекомендуемый UX-сценарий

```mermaid
sequenceDiagram
  participant EXT as Extension
  participant API as Backend

  EXT->>API: POST /reviews-analyses
  API-->>EXT: 202 summary (pending)
  loop пока status pending|processing
    EXT->>API: GET /reviews-analyses/active
    API-->>EXT: [{id, status, progressPercent, ...}]
    Note over EXT: виджет-лоадер
  end
  EXT->>API: GET /reviews-analyses/{id}/result
  API-->>EXT: result для экрана отчёта
```

| Шаг | Запрос | Что делать на клиенте |
|-----|--------|------------------------|
| Запуск | `POST .../reviews-analyses` | Сохранить `id` из `data`, открыть виджет |
| Квота | `GET .../quota` | Показать остаток (для extension — в «товарах», см. фасад ниже) |
| Маркетплейсы | `GET .../marketplaces` | Список включённых площадок для формы |
| Виджет | `GET .../active` каждые 2–5 с | Если массив не пуст — лоадер; пуст — скрыть |
| Poll одного | `GET .../{id}` | Альтернатива active, если открыт конкретный анализ |
| Отчёт | `GET .../{id}/result` | Только после `status === "completed"` |
| Список | `GET .../reviews-analyses` | История: title, status, nm, даты |
| Отмена | `POST .../{id}/cancel` | По кнопке в виджете |

Не показывайте пользователю: имена моделей, токены, stage-коды пайплайна, quota/org internals, сырые ошибки инфраструктуры.

### Эндпоинты клиента

Базовый prefix: `/api/v1/reviews-analyses`.

#### `POST /` → `202`

Тело:

| Поле | Тип | Обязательный | Описание |
|------|-----|--------------|----------|
| `marketplace` | string | да | `wb` / `lamoda`. `ozon` — создание временно отключено (антибот на DC IP) |
| `nomenclatures` | string[] | да | 1…50 артикулов. WB/Ozon — числовые строки; Lamoda — SKU (`RTLAAN490701`). Числа в JSON принимаются и нормализуются в строки |
| `organizationId` | uuid \| null | нет | Нужен при неоднозначном seat (иначе 409) |
| `sampleSize` | number \| null | нет | Размер выборки; иначе default из настроек |
| `splitByItems` | boolean | нет | default `true` |
| `includeRecommendations` | boolean | нет | default `true` |
| `mode` | `"full"` \| `"budget"` | нет | default `full` |
| `title` | string \| null | нет | Заголовок в списке |
| `dateFrom` / `dateTo` | string \| null | нет | Фильтр дат (если поддерживается источником) |

Ответ: `ReviewsAnalysisSummary` (см. ниже).

Лимит тарифа — пул **Мурликов** за период (`max_mh_credits_per_period`). Анализ отзывов списывает вес `reviews_analysis/base` (дефолт **10** Мурликов) за каждый артикул в запросе, в том числе при повторном анализе того же товара. Если стоимость запуска больше остатка, create отклоняется целиком (`402 PLAN_LIMIT_EXCEEDED`); частичного запуска нет.

`GET /quota` для обратной совместимости с Chrome extension отдаёт **фасад в единицах «товаров»**: `limit/used/remaining = floor(credits / weight)`, где `weight` — эффективный вес `reviews_analysis/base`. Manager-portal показывает пользователю язык Мурликов и пояснение «1 товар = N Мурликов»; extension может продолжать считать ответ в товарах.

#### `GET /quota`

Query: `organizationId` (опционально).

Ответ `data`:

| Поле | Тип | Описание |
|------|-----|----------|
| `limit` | number \| null | Лимит в единицах фасада (товары для extension); `null` — безлимит |
| `used` | number | Зарезервировано + списано за период (в тех же единицах) |
| `remaining` | number \| null | Остаток; `null` при безлимите |
| `periodStart` / `periodEnd` | datetime | Границы расчётного периода |
| `organizationId` | uuid \| null | Org seat, если применимо |
| `quotaMode` | string | `per_member_seat` \| `shared_pool` |

#### `GET /marketplaces`

Список маркетплейсов, включённых в admin settings. Форма создания показывает только их.

Ответ `data`:

| Поле | Тип | Описание |
|------|-----|----------|
| `items` | array | Включённые площадки |
| `items[].id` | string | Канонический id: `wb` \| `lamoda` \| `ozon` \| `yandex_market` \| `avito` |
| `items[].label` | string | Подпись для UI |

#### `GET /`

Query: `page` (≥1), `pageSize` (1…100, alias `pageSize`), `status` (опционально).

Ответ: paginated список `ReviewsAnalysisSummary`.

#### `GET /active`

Ответ `data`: массив `ReviewsAnalysisActiveItem` — только `pending` и `processing` текущего пользователя.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid | Идентификатор анализа |
| `status` | `"pending"` \| `"processing"` | Состояние |
| `marketplace` | string | Маркетплейс |
| `nomenclatures` | string[] | Артикулы |
| `title` | string \| null | Заголовок |
| `progressPercent` | number | 0…100 для лоадера |

#### `GET /{analysisId}`

Ответ: `ReviewsAnalysisSummary`.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid | Идентификатор |
| `status` | `"pending"` \| `"processing"` \| `"completed"` \| `"error"` \| `"cancelled"` | Статус |
| `marketplace` | string | Маркетплейс |
| `nomenclatures` | string[] | Артикулы |
| `title` | string \| null | Заголовок |
| `progressPercent` | number | Прогресс 0…100 |
| `reviewsCount` | number \| null | После обработки |
| `questionsCount` | number \| null | После обработки |
| `errorMessage` | string \| null | Пользовательский текст при `error` |
| `createdAt` | datetime | Создан |
| `completedAt` | datetime \| null | Завершён / ошибка / отмена |

В summary **нет** `result`, модели LLM, cost, quota ids, внутренних stage.

#### `GET /{analysisId}/result`

Только для `status === "completed"` и непустого результата.

| Поле | Тип | Описание |
|------|-----|----------|
| `id` | uuid | Идентификатор |
| `status` | `"completed"` | Всегда completed |
| `marketplace` | string | Маркетплейс |
| `nomenclatures` | string[] | Артикулы |
| `title` | string \| null | Заголовок |
| `reviewsCount` | number \| null | Число отзывов |
| `questionsCount` | number \| null | Число вопросов |
| `result` | object | Агрегированный отчёт (stats, llm, warnings, …) |
| `createdAt` | datetime | Создан |
| `completedAt` | datetime \| null | Завершён |

Если анализ ещё идёт или без результата → `409 CONFLICT` («Результат анализа ещё не готов»).  
Чужой / несуществующий id → `404`.

Структура `result` ориентирована на UI отчёта (статистика, сильные/слабые стороны, рекомендации, предупреждения). Поля warnings содержат пользовательские сообщения; коды warnings — для логики клиента (скрытие секций), не для прямого показа как tech error.

#### `POST /{analysisId}/cancel`

Ответ: обновлённый `ReviewsAnalysisSummary`.

### Типовые ошибки (клиенту)

| HTTP | Смысл для UI |
|------|----------------|
| 401 | Нужно войти / обновить токен |
| 403 | Нет доступа / раздел выключен |
| 404 | Анализ не найден |
| 409 | Нужен `organizationId` или результат ещё не готов |
| 402 | Исчерпана квота Мурликов за период (или запрос дороже остатка) |
| 400 | Некорректный запрос (маркетплейс, nm, размер выборки) |

## Квоты

Фича тарифа: `reviews_analysis` (`user_or_org_seat`).  
Канонический лимит: `max_mh_credits_per_period` (+ докупка / буст `mh_credits`).  
Поле `max_reviews_products_per_period` — deprecated read-compat зеркало (`floor(credits / 10)`), не источник истины.

Внутри биллинга списание идёт в **Мурликах** через ledger `billing_mh_credits_ledger` и веса из каталога (`mh_credit_weights`, см. [Биллинг](./billing.md#мурлики-и-веса)).  
Стоимость анализа: `compute_cost(reviews_analysis, sku_count=len(nomenclatures))` → обычно `sku_count * 10`.

### Legacy-фасад `/quota` (extension)

`ReviewsAnalysisQuotaService` конвертирует статус кредитов в «товары» для клиентов, которые ещё считают лимит по артикулам:

| Поле quota | Значение |
|------------|----------|
| `limit` / `used` / `remaining` | `floor(credits_* / weight)`, `weight = reviews_analysis/base` |
| `LIMIT_RESOURCE` в 402 | по-прежнему `reviews_products_per_period` (алиас → `mh_credits`) |

`null` в тарифе / в ответе `/quota` — безлимит; иначе число **товаров-эквивалентов** за billing-период (не сырые Мурлики).

Повторный анализ того же артикула снова расходует кредиты (каждый запуск вызывает модель).

### Хранение и расчёт

| Величина | Источник |
|----------|----------|
| `limit` (кредиты) | `get_effective_plan().max_mh_credits_per_period` (тариф + промо + add-ons) |
| `used` (кредиты) | `billing_mh_credits_ledger.reserved + consumed` за `period_start` |
| `remaining` (кредиты) | `null` при безлимите, иначе `max(0, limit - used)` |
| поля `/quota` | фасад: `floor(кредиты / weight)` |

Лимит попадает в Redis-кэш effective plan (`billing:effective_plan`, TTL 60 с / stale 5 мин) через `plan_to_dict` / `plan_from_dict`. Поле `max_mh_credits_per_period` обязательно сериализуется — иначе после первого попадания в кэш клиент видел бы ложный безлимит. Зеркало `max_reviews_products_per_period` тоже пишется для старых читателей кэша.

### Проверка и списание

1. `POST /reviews-analyses`: `resolve` (мягкая проверка) → `reserve` под `SELECT … FOR UPDATE` (жёсткая) в Мурликах.
2. Условие: `used_credits + cost <= limit_credits`. При `limit is None` ограничение не действует.
3. **All-or-nothing:** если стоимость запуска больше остатка, весь запрос отклоняется с `402 PLAN_LIMIT_EXCEEDED` (частичного списания нет).
4. Успех job → `commit` (reserved→consumed); ошибка / отмена → `release` reserved.

Режим в настройках раздела (`quotaMode`):

| Режим | Поведение |
|-------|-----------|
| `per_member_seat` (default) | У каждого участника org свой пул Мурликов = лимит тарифа owner. 2 org → 2 независимых пула. |
| `shared_pool` | Все seat-запуски едят общий пул кредитов owner. |

Анализ всегда привязан к пользователю, который запустил. При неоднозначности seat нужен `organizationId` (иначе 409).

Клиентский остаток: `GET /api/v1/reviews-analyses/quota?organizationId=…` (фасад в товарах для extension).

После смены лимита в тарифе или деплоя фикса кодека кэш `billing:effective_plan:*` можно сбросить; иначе новое значение подтянется по истечении TTL/stale.

## Настройки раздела (admin)

`GET/PATCH /api/v1/admin/reviews-analysis/settings`  
Не `platform_settings`. Секреты OpenRouter только в `.env` на сервере (не в клиентских ответах).

Поля ответа / PATCH:

| Поле | Тип | Описание |
|------|-----|----------|
| `isEnabled` | boolean | Раздел включён |
| `llmModel` | string | Модель OpenRouter |
| `defaultSampleSize` | number | Выборка по умолчанию |
| `maxNomenclatures` | number | Макс. артикулов в одном анализе |
| `maxReviewsPerAnalysis` | number | Потолок отзывов |
| `quotaMode` | string | `per_member_seat` \| `shared_pool` |
| `enabledMarketplaces` | string[] | Включённые площадки: `wb`, `lamoda`, `ozon`, `yandex_market`, `avito` (минимум одна) |

`yandex_market` и `avito` — заглушки без адаптера: их можно включить в admin, но создание анализа отклоняется до реализации источника.

UI: Admin Panel → Анализ отзывов → Параметры — чекбоксы маркетплейсов.

LLM HTTP timeout: 600 с (connect 30 с). На `ReadTimeout` — не больше 2 попыток; на 5xx upstream — до 3. Для долгих запросов через HTTP-прокси нужен высокий idle/request timeout прокси (иначе обрыв до ответа).

## Журнал анализов (admin)

UI: Admin Panel → Анализ отзывов → Анализы.

| Метод | Path | Назначение |
|-------|------|------------|
| GET | `/api/v1/admin/reviews-analysis/analyses` | Список с фильтрами |
| GET | `/api/v1/admin/reviews-analysis/analyses/{id}` | Детали + параметры запуска |
| GET | `/api/v1/admin/reviews-analysis/analyses/{id}/result` | Полный result (только completed) |
| POST | `/api/v1/admin/reviews-analysis/analyses/{id}/cancel` | Отмена (без проверки владельца) |

Query списка: `page`, `pageSize`, `status`, `marketplace`, `search` (email / имя / id анализа / title / артикул).

В ответе для админа допустимы служебные поля (`llmModel`, `progressStage`, cost, quota flags) — это панель поддержки, не клиентское UI.

## Wildberries (публичный источник)

Адаптер повторяет поток сайта, не seller API:

1. `nm` → `imt_id`: basket CDN `GET https://{host}/vol{vol}/part{part}/{nm}/info/ru/card.json`
   (`vol = nm // 100000`, `part = nm // 1000`; host из `cdn.wbbasket.ru/api/v3/upstreams` →
   `origin.mediabasket_route_map`, fallback — `shard-list.json` / статическая таблица диапазонов).
2. Хост отзывов: `GET https://feedback-bt.wildberries.ru/feedback/api/v2/host?imt={imt}` → `https://feedback-view-NN.wb.ru`.
3. Отзывы: `GET {host}/feedbacks/v2/{imt}` (параметр — **imt**). Публично ~до 1000 отзывов на карточку; фильтр по запрошенным nm.
4. Вопросы: `https://questions.wildberries.ru/api/v1/questions?imtId={imt}`.
5. Медиа: `photos[]`, `video`; legacy `photo` учитывается.

Fallback-хосты: `feedbacks1.wb.ru`, `feedbacks2.wb.ru`.

## Lamoda (публичный источник)

Seller API не подходит: там только вопросы своего кабинета, без публичных отзывов.

Адаптер ходит в storefront API витрины:

1. Сводка: `GET https://www.lamoda.ru/api/v1/product/reviews_info?sku={sku}&with_related=true`
2. Отзывы: `GET .../product/reviews?sku={sku}&sort=date&sort_direction=desc&limit={n}&offset={n}&with_related=1`
3. Вопросы: `GET .../product/questions?sku={sku}&limit={n}&offset={n}`

Маппинг в канон:

| Lamoda | Review / Question |
|--------|-------------------|
| `uuid` / `id` | `comment_id` / `question_id` |
| запрошенный `sku` | `nomenclature` (строка) |
| `rating` | `product_valuation` |
| `text` | `feedback_text` |
| `username` | `user_name` |
| `created_time` | `feedback_date` |
| `answer` (string) | `answer_text` |
| `like_count` / `dislike_count` | votes |
| `photos` / `snippet.photos` | `photos_count` |

Антибот (ServicePipe): обычный Python TLS (`httpx`) даёт `403`. Адаптер ходит через
`curl_cffi` с impersonate Chrome (как Ozon). На DC IP обычно нужны cookie и/или
residential-прокси с тем же egress, с которого снимали сессию.

Настройки:

- Sealed cookie в Postgres — admin «Сессии площадок» (приоритет над env)
- `LAMODA_STOREFRONT_COOKIE` — fallback Cookie-строка витрины (spid/spsc, желательно sid/lid)
- `LAMODA_HTTP_PROXY_URL` — опциональный HTTP-прокси; env `HTTP_PROXY`/`HTTPS_PROXY` не читаются (`trust_env=false`)
- `REVIEWS_HTTP_PROXY_PROVIDER` — см. Ozon / прогрев ProxyMarket ниже

Локальный съём: `uv run python scripts/capture_storefront_cookie.py lamoda` (см. Ozon).

Без готовой cookie адаптер снимает ServicePipe cookie запросом на главную.
При ``REVIEWS_HTTP_PROXY_PROVIDER=proxymarket`` оркестратор перед `loading_data`
выполняет этап `warming_proxy` (`infrastructure/proxy_warmup.py`): нейтральный host +
hostname витрины, пока не будет N успехов **подряд**; рабочие GET Lamoda/Ozon идут
через `get_with_transport_retry` (CONNECT/TLS). Другие провайдеры / пустое значение —
без прогрева. При массовых `lamoda_api.antibot` — обновить cookie в admin и/или прокси.

Идентификатор — Lamoda SKU (латиница+цифры, например `RTLAAN490701`), не числовой nm.

## Ozon (публичный источник)

Seller API не подходит: нужны отзывы любого товара, не только своего кабинета.

Адаптер ходит в публичный composer-api витрины:

1. Отзывы: `GET .../page/json/v2?url=/product/{id}/reviews/?sort=published_at_desc&page=1`
   Пагинация: не наращивать `page` вручную — без `page_key` виджет `webListReviews` пропадает.
   Следующая страница — из `paging.nextButton` (например `?page=2&page_key=...&sort=published_at_desc`).
2. Вопросы: `.../questions/?page={n}` (`paging.type=loadMore`, `page` без `page_key` работает).
3. Виджеты: `webListReviews-*`, `webListQuestions-*` в `widgetStates` (значение — JSON-строка или объект).

Маппинг в канон:

| Ozon | Review / Question |
|------|-------------------|
| `uuid` / `id` | `comment_id` / `question_id` |
| `itemId` | `nomenclature` |
| `content.score` | `product_valuation` |
| `content.comment` / `positive` / `negative` | текст / pros / cons |
| `publishedAt` (unix) | `feedback_date` ISO |
| ответы продавца в `comments.list` | `answer_text` |

Антибот (Variti): обычный Python TLS (`httpx`) даёт `307`/`403` даже с валидной cookie —
отсев по отпечатку клиента. Адаптер ходит через `curl_cffi` с impersonate Chrome и
следует редиректам `__rr=N`.

Настройки:

- Sealed cookie в Postgres (`reviews_storefront_sessions`) — задаётся в admin
  «Сессии площадок» без рестарта API. Приоритет над env.
- `OZON_COMPOSER_COOKIE` — fallback Cookie-строка витрины (одна строка в кавычках в `.env`)
- `OZON_HTTP_PROXY_URL` — опциональный HTTP-прокси; если пусто — берётся `LAMODA_HTTP_PROXY_URL`
- `REVIEWS_HTTP_PROXY_PROVIDER` — провайдер прокси reviews (`proxymarket` → этап `warming_proxy`; иначе без прогрева)

Локальный съём cookie (host Chrome, не prod Docker)::

    cd backend
    uv sync --group capture
    uv run patchright install chrome
    uv run python scripts/capture_storefront_cookie.py ozon

Готовую строку вставить в admin → «Сессии площадок». Cookie нужно снимать **с того же egress**,
что и прокси (иначе Variti даёт `403` / `incidentId` после успешного CONNECT). Прогрев бьёт
в composer-api и не считает `403` «успехом»; при antibot на fetch — до 8 повторов с паузой.

### Какие cookies нужны

Снимать лучше заголовок `Cookie` с **успешного** (`200`) запроса
`/api/composer-api.bx/page/json/v2` (DevTools → Network), а не «все cookies домена наугад».
Туда попадут и HttpOnly.

| Cookie | Роль | Обязательность | Срок (ориентир) |
|--------|------|----------------|-----------------|
| `abt_data` | антибот Variti | **критично** — без неё типичен `307` | короткий: часы…дни; «хрупкая», часто протухает первой |
| `rfuid` | антибот / device fingerprint | **критично** — без неё типичен `307` | короткий / средний |
| `__Secure-access-token` | сессия витрины | нужна для стабильной сессии | дни…недели; в значении есть метка выдачи |
| `__Secure-refresh-token` | обновление сессии | желательна вместе с access | как у access |
| `xcid` | client id витрины | желательна | долгая |
| `__Secure-ext_xcid` | связанный client id | желательна | долгая |
| `__Secure-ETC` | edge/client token | желательна; в Set-Cookie встречался срок ~1 год | долгая (~год) |
| `__Secure-user-id` | id пользователя (`0` = гость) | необязательна сама по себе | как сессия |
| `__Secure-ab-group` | A/B | не критична | долгая |
| `is_cookies_accepted` | согласие на cookies | не критична | долгая |
| `ADDRESSBOOKBAR_*` и прочий UI | виджеты витрины | не нужны для composer-api | разные |

Практичный минимум для cookie Ozon: `abt_data` + `rfuid` + access/refresh + `xcid`/`__Secure-ETC`.
Полная строка с успешного composer-запроса предпочтительнее ручной «сборки».

Точные `Expires` / `Max-Age` — в браузере: Application → Cookies → `ozon.ru`.
При массовых `ozon_composer.antibot` в логах — обновить cookie в admin «Сессии площадок»
(и при необходимости прокси с тем же egress, с которого снимали сессию).

Идентификатор номенклатуры — storefront `product_id` / `itemId` (целое число, совместимо с `nomenclatures[]`).

## Ограничения

- До 5000 отзывов на анализ (настраивается, потолок 5000).
- Несколько номенклатур с первого дня.
- Marketplace-agnostic канон Review/Question; адаптеры WB, Lamoda и Ozon.
- Идентификаторы номенклатур — строки (`nomenclatures: string[]`); WB/Ozon — числовые строки.
- WB публично отдаёт срез (~1000) отзывов по imt — полный `feedbackCount` с карточки больше.
- Доступность площадок управляется `enabledMarketplaces` в admin settings (по умолчанию `wb` + `lamoda`).
- Ozon: адаптер есть; на типичном DC IP Variti часто режет composer-api даже с cookie —
  включать в admin только при рабочем cookie/прокси.
- Lamoda: на DC IP обычно нужны cookie витрины (admin sealed или `LAMODA_STOREFRONT_COOKIE`)
  и/или `LAMODA_HTTP_PROXY_URL`.
- Яндекс Маркет и Авито: заглушки в каталоге/admin; create отклоняется до появления адаптера.

## Масштабирование

- Горизонтально: больше ARQ workers.
- Новый маркетплейс: адаптер `ReviewsSource` + factory.
- Смена LLM: только admin settings / OpenRouter на сервере.
