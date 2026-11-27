# Клиент: отчёт «Точки входа» (enrichment)

Базовый URL: `{API}/api/v1`  
OpenAPI: `{API}/docs` (tag `entry-points`)  
Архитектура бэкенда: [entry-points.md](../architecture/entry-points.md)

Авторизация: `Authorization: Bearer <accessToken>`.  
Доступ: фича тарифа `browser_extension` (тот же gate, что у `/extension/*`).  
Тела и ответы — **camelCase**. Обёртка успеха: `{ "data": …, "meta": {} }`.

Клиент (в первую очередь Chrome extension) **сам** ходит в WB Seller API,
нормализует данные и шлёт payload на MH. Backend не ходит в WB за этим отчётом.

Excel-экспорт остаётся на клиенте.

---

## Поток для UI

```
1. Собрать данные WB (entry-points + sales/AOV + опционально activity-by-orders)
2. POST /entry-points/reports          → sync-отчёт сразу (202)
3. Пока insightsStatus = pending|running
     GET /entry-points/reports/{id}/insights
4. Готово: ready | skipped | failed (при failed — базовые тексты из rules)
```

Рекомендуемый полл: старт через 1–2 с, интервал 2 с, backoff до 5 с, таймаут ~90 с.  
Пока LLM не готов — рисуйте sync-часть (KPI, таблица, lost, day heatmap, `ruleCandidates`).

Идемпотентность: повторный POST с тем же payload в окне ~1 ч вернёт тот же `reportId`
без повторного LLM, если инсайты уже `ready` / `running` / `skipped`.

---

## 1. Создать отчёт

```http
POST /api/v1/entry-points/reports
Content-Type: application/json
Authorization: Bearer <accessToken>
```

### Request body

| Поле | Тип | Обязательный | Описание |
|------|-----|:------------:|----------|
| `meta` | object | да | Период, фильтры, seller, fallback-цена |
| `current` | object | да | Точки и метрики текущего периода |
| `previous` | object \| null | при `compareEnabled` | Тот же shape, что `current` |
| `activityByRegionHour` | object \| null | нет | Матрица регион×час для heatmap §7 |

#### `meta`

| Поле | Тип | Обязательный | Описание | Пример |
|------|-----|:------------:|----------|--------|
| `periodFrom` | string (date) | да | Начало периода `YYYY-MM-DD` | `"2026-06-16"` |
| `periodTo` | string (date) | да | Конец периода включительно | `"2026-06-22"` |
| `compareEnabled` | boolean | нет (default `false`) | Считать дельты vs прошлый период | `true` |
| `prevFrom` | string (date) \| null | если compare | Начало прошлого периода | `"2026-06-09"` |
| `prevTo` | string (date) \| null | если compare | Конец прошлого периода | `"2026-06-15"` |
| `sellerKey` | string (1…128) | да | Стабильный ключ кабинета (supplier id / cabinet id) | `"123456"` |
| `fallbackAvgPrice` | number / string decimal | да, `> 0` | AOV селлера, если у точки нет `avgPrice` | `1250.50` |
| `filters` | object | нет | См. ниже | |

Длина периода: **1…31** день. `periodTo` ≥ `periodFrom`.

##### `meta.filters`

| Поле | Тип | Default | Описание |
|------|-----|---------|----------|
| `nms` | number[] | `[]` | Артикулы WB (nmId) |
| `brands` | string[] | `[]` | Бренды |
| `subjects` | number[] | `[]` | Предметы |
| `tagIds` | string[] | `[]` | Теги кабинета |
| `buyerType` | `"all"` \| `"new"` \| `"returning"` | `"all"` | Тип покупателя |
| `countries` | string[] | `[]` | Страны |

Фильтры не пересчитываются на сервере — они часть ключа идемпотентности.
Клиент обязан уже применить их при сборе WB-данных.

#### `current` / `previous`

| Поле | Тип | Обязательный | Описание |
|------|-----|:------------:|----------|
| `points` | EntryPoint[] | да | Верхнеуровневые точки (0…80) |
| `totals` | object \| null | нет | Суммы KPI; если нет — сумма по `points` |

`totals`: `{ shows, clicks, carts, orders }` — целые ≥ 0.

#### EntryPoint

| Поле | Тип | Обязательный | Описание |
|------|-----|:------------:|----------|
| `id` | string (1…128) | да | Стабильный id точки (не менять между периодами) |
| `name` | string | да | Название для UI («Поисковая выдача») |
| `group` | string | да | Группа WB («Поиск», «Разное», …) |
| `shows` | int ≥ 0 | да | Показы |
| `clicks` | int ≥ 0 | да | Переходы |
| `carts` | int ≥ 0 | да | Корзины |
| `orders` | int ≥ 0 | да | Заказы |
| `avgPrice` | decimal \| null | да, если `orders > 0` | Средний чек по точке |
| `children` | EntryPoint[] | нет | Вложенные подточки (суммарно ≤ 400) |
| `daily` | Daily[] | нет | Посуточная разбивка (≤ 31 день) |

`Daily`: `{ date, shows, clicks, carts, orders }`.

Правила валидации на сервере:

- дубли `id` (включая детей) → ошибка;
- `avgPrice` обязателен и `> 0` при `orders > 0`.
- `clicks > shows` (в т.ч. при `shows = 0`) — **допустимо**: на WB так бывает; CTR на сервере clamp в `[0, 100]`.

#### `activityByRegionHour` (опционально)

| Поле | Тип | Описание |
|------|-----|----------|
| `averageRow` | number[24] | Индекс/объём по часам 0…23 (строка «В среднем») |
| `regions` | `{ name: string, hours: number[24] }[]` | Федеральные округа |

Без этого поля `regionHourHeatmap` в ответе будет `null`, LLM-слот 17.2 не вызывается.

### Пример request

```json
{
  "meta": {
    "periodFrom": "2026-06-16",
    "periodTo": "2026-06-22",
    "compareEnabled": true,
    "prevFrom": "2026-06-09",
    "prevTo": "2026-06-15",
    "sellerKey": "wb-supplier-778899",
    "fallbackAvgPrice": 1180.0,
    "filters": {
      "nms": [123456789],
      "brands": [],
      "subjects": [],
      "tagIds": [],
      "buyerType": "all",
      "countries": []
    }
  },
  "current": {
    "totals": { "shows": 12000, "clicks": 840, "carts": 210, "orders": 95 },
    "points": [
      {
        "id": "search",
        "name": "Поиск",
        "group": "Поиск",
        "shows": 8000,
        "clicks": 400,
        "carts": 120,
        "orders": 55,
        "avgPrice": 1250,
        "daily": [
          { "date": "2026-06-16", "shows": 1100, "clicks": 55, "carts": 16, "orders": 7 },
          { "date": "2026-06-17", "shows": 1200, "clicks": 60, "carts": 18, "orders": 8 }
        ],
        "children": [
          {
            "id": "search-listing",
            "name": "Поисковая выдача",
            "group": "Поиск",
            "shows": 7000,
            "clicks": 350,
            "carts": 100,
            "orders": 48,
            "avgPrice": 1250
          }
        ]
      }
    ]
  },
  "previous": {
    "totals": { "shows": 11000, "clicks": 700, "carts": 180, "orders": 80 },
    "points": [
      {
        "id": "search",
        "name": "Поиск",
        "group": "Поиск",
        "shows": 7500,
        "clicks": 300,
        "carts": 90,
        "orders": 40,
        "avgPrice": 1200
      }
    ]
  },
  "activityByRegionHour": {
    "averageRow": [0.1, 0.1, 0.05, 0.05, 0.1, 0.2, 0.4, 0.6, 0.9, 1, 0.95, 0.9, 0.85, 0.8, 0.75, 0.7, 0.8, 0.95, 1, 0.9, 0.7, 0.5, 0.3, 0.2],
    "regions": [
      {
        "name": "ЦФО",
        "hours": [0.1, 0.1, 0.05, 0.05, 0.1, 0.2, 0.4, 0.6, 0.9, 1, 0.95, 0.9, 0.85, 0.8, 0.75, 0.7, 0.8, 0.95, 1, 0.9, 0.7, 0.5, 0.3, 0.2]
      }
    ]
  }
}
```

### Response `202 Accepted`

```json
{
  "data": {
    "reportId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "status": "ready",
    "insightsStatus": "pending",
    "report": { }
  },
  "meta": {}
}
```

| Поле | Тип | Описание |
|------|-----|----------|
| `reportId` | uuid | Идентификатор отчёта для GET |
| `status` | `"ready"` | Sync всегда готов при 202 |
| `insightsStatus` | см. статусы | Состояние LLM-фазы |
| `report` | SyncReport | Полная sync-часть (см. ниже) |

`insightsStatus` сразу может быть:

| Значение | Когда |
|----------|--------|
| `pending` | Есть candidates / matrix, job в очереди |
| `skipped` | Нечего обогащать (нет rules и нет region×hour) |
| `failed` | Не удалось поставить job (редко); sync всё равно валиден |

---

## 2. Sync-отчёт (повторно)

```http
GET /api/v1/entry-points/reports/{reportId}
Authorization: Bearer <accessToken>
```

Ответ `200`:

```json
{
  "data": {
    "reportId": "…",
    "status": "ready",
    "insightsStatus": "running",
    "report": { },
    "createdAt": "2026-06-22T10:00:00Z",
    "expiresAt": "2026-06-29T10:00:00Z"
  }
}
```

Чужой / неизвестный id → `404` («Отчёт «Точки входа» не найден»).

Срок жизни строки ~7 дней (`expiresAt`).

---

## 3. Инсайты (полл)

```http
GET /api/v1/entry-points/reports/{reportId}/insights
Authorization: Bearer <accessToken>
```

### Response

| Поле | Тип | Описание |
|------|-----|----------|
| `reportId` | uuid | |
| `insightsStatus` | string | `pending` \| `running` \| `ready` \| `failed` \| `skipped` |
| `insights` | InsightCard[] | До 6 карточек |
| `actionPlan` | object \| null | Блок «Куда бить» |
| `regionHourInsight` | object \| null | Лучший/худший слот РК |
| `errorMessage` | string \| null | User-facing при `failed` |

#### InsightCard

| Поле | Тип | Описание |
|------|-----|----------|
| `tag` | string | Стабильный код (`SEARCH-CTR-LOW`, …) — для аналитики/UTM |
| `severity` | `"critical"` \| `"warn"` \| `"win"` | Из rules, LLM не перезаписывает |
| `pointId` | string | |
| `pointName` | string | |
| `group` | string | |
| `lostRub` | decimal | |
| `title` | string | ≤ 90 симв. |
| `body` | string | ≤ 220 симв. |
| `action` | string | Текст CTA ≤ 64 симв. (обрезка по границе слова) |

#### `actionPlan`

```json
{
  "summary": "Сфокусируйтесь на поиске: CTR 4% при цели 8%.",
  "steps": [
    "A/B-тест главного фото на топ-запросах",
    "Поднять ставки в АРК на пиковые часы",
    "Запустить проверку UTM на ссылочном трафике"
  ]
}
```

Ровно 3 шага. Может быть `null`, если нет critical/warn кандидатов.

#### `regionHourInsight`

```json
{
  "bestSlot": { "region": "ЦФО", "hour": 12, "intensity": 0.95 },
  "worstSlot": { "region": "ДФО", "hour": 3, "intensity": 0.08 },
  "recommendation": "Сдвиньте ставки АРК на 11–13 и 18–21 по ЦФО."
}
```

### Поведение по статусам

| `insightsStatus` | UI |
|------------------|-----|
| `pending` / `running` | Показать sync + скелетон/старые `ruleCandidates` как черновик |
| `ready` | Подставить `insights`, `actionPlan`, `regionHourInsight` |
| `skipped` | Секцию AI можно скрыть или оставить пустой |
| `failed` | Показать `insights` (базовые тексты из rules) + спокойное сообщение из `errorMessage` |

Пример user-facing при fail: «Не удалось уточнить подсказки, показаны базовые».

Ретрая отдельным endpoint нет: новый POST с тем же/новым payload.

---

## Структура `report` (sync)

| Поле | Тип | Назначение в UI |
|------|-----|-----------------|
| `targetCtr` | number | Цель CTR % (P75) — подсказка в шапке |
| `totalLostRub` | decimal | Баннер «упущенная выручка» |
| `peakLostDate` | date \| null | Пик на графике |
| `avgLostPerDay` | decimal | «В среднем −N ₽/день» |
| `kpi` | object | Полоса Показы/Переходы/Корзины/Заказы |
| `points` | EntryPointOut[] | Таблица воронки (дерево) |
| `lostByDay` | `{ date, lostRub }[]` | Столбцы баннера |
| `topProblemPoints` | array (≤5) | Ряд проблемных точек |
| `dayHeatmap` | array | Heatmap по дням |
| `regionHourHeatmap` | object \| null | Нормированная 0…1 матрица |
| `ruleCandidates` | array | Черновик инсайтов до LLM |
| `periodFrom` / `periodTo` | date | |
| `compareEnabled` | boolean | |
| `prevFrom` / `prevTo` | date \| null | Подпись «vs …» |

### `kpi`

```json
{
  "shows": { "value": 12000, "delta": 0.09 },
  "clicks": { "value": 840, "delta": 0.2 },
  "carts": { "value": 210, "delta": null },
  "orders": { "value": 95, "delta": 0.1875 }
}
```

`delta` — относительная доля `(current − previous) / previous`.  
`null`, если previous ≤ 0 (не рисуйте ±0%).

### EntryPointOut

Как вход плюс:

| Поле | Тип | Описание |
|------|-----|----------|
| `ctr` | number | % |
| `crSo` | number | CR показ→заказ (доля) |
| `lostRub` | decimal | Упущ. ₽ по точке |
| `diffs` | `{ shows?, clicks?, carts?, orders? }` | Относительные дельты или `null` |
| `daily` | `{ date, shows, clicks, carts, orders, lostRub }[]` | Раскрытие строки |
| `children` | EntryPointOut[] | |

`lostRub` на детях считается отдельно; **в итог баннера входят только верхний уровень**.

### `topProblemPoints[]`

`{ id, name, group, lostRub, ctr, targetCtr }` — клик скроллит к строке таблицы по `id`.

### `dayHeatmap[]`

`{ date, shows, clicks, carts, orders, ctr, crSo }` — абсолютные суммы по дням.

### `regionHourHeatmap`

```json
{
  "average": [0, 0.1, … 1.0],
  "regions": [{ "name": "ЦФО", "hours": [0, …, 1] }]
}
```

Значения нормированы 0…1 по max строки.

### `ruleCandidates[]`

Черновик до LLM: те же поля, что InsightCard, плюс `description`, `actionHint`.  
Можно показывать сразу; после `insights.ready` заменить на `insights`.

---

## Ошибки

Формат: `{ "error": { "code", "message", "details?" } }` (см. общий API-design).

| HTTP | Типичная причина | Что показать пользователю |
|------|------------------|---------------------------|
| 401 | Нет токена / сессия истекла | Предложить войти снова |
| 403 | Нет фичи `browser_extension` | Предложить обновить тариф / разблокировать расширение |
| 404 | Чужой или неизвестный `reportId` | «Отчёт не найден» / создать заново |
| 400 / 422 | Валидация payload | Текст из `message` (уже без внутренних кодов) |
| 5xx | Сбой сервера | «Не удалось построить отчёт. Попробуйте ещё раз» |

Примеры валидации:

- нет `fallbackAvgPrice`;
- у точки с заказами нет `avgPrice`;
- `compareEnabled` без `previous` / `prevFrom`/`prevTo`;
- период > 31 дня;
- дубли id;
Клиент **не** показывает stack / имена полей схемы / SQL.

---

## Chrome extension: рекомендуемая интеграция

### Доступ

1. Уже используете `GET /extension/entitlements` + gate `browser_extension`.
2. Страница Entry Points остаётся на live WB API в браузере (cookies селлера).
3. После успешного сбора периода — один `POST /entry-points/reports` вместо локального пересчёта lost/insights (или параллельно на переходном этапе).

### Сбор payload из текущих WB-вызовов

| Источник в extension | Куда в payload |
|----------------------|----------------|
| `POST …/customer-profile/entry-points` (текущий + prev) | `current.points` / `previous.points` |
| Funnel / sales AOV | `meta.fallbackAvgPrice`, `avgPrice` по точкам |
| History / by-day (если есть) | `points[].daily` |
| `activity-by-orders` | `activityByRegionHour` |
| Query filters (nms, brands, …) | `meta.filters` |
| Supplier / cabinet id | `meta.sellerKey` |

`id` точки: стабильный slug из `group + name` (или WB-имени), одинаковый в current/previous.

### UI-маппинг

| Блок страницы | Поле ответа |
|---------------|-------------|
| Баннер −₽ | `report.totalLostRub`, `avgLostPerDay`, `peakLostDate`, `lostByDay` |
| «Куда бить» | `insights.actionPlan` (пока null — скрыть или skeleton) |
| Топ-5 проблем | `report.topProblemPoints` |
| KPI-полоса | `report.kpi` |
| AI-карточки | `insights.insights` (черновик: `report.ruleCandidates`) |
| Таблица | `report.points` (+ `daily` при expand) |
| Heatmap дни | `report.dayHeatmap` |
| Heatmap регион×час | `report.regionHourHeatmap` + `insights.regionHourInsight` |
| Excel | Клиентский экспорт из sync (+ insights когда ready) |

### Состояние загрузки

```
WB fetch → POST report → render sync
         ↘ poll insights → patch AI blocks
```

Не блокируйте таблицу ожиданием LLM.

### Аналитика продукта (клиент)

Из ТЗ (отправка в ваш analytics):

- `report.entry_points.opened`
- `report.entry_points.insight.cta_clicked` — props: `tag`, `severity`
- `report.entry_points.row.expanded`
- `report.entry_points.lost_revenue.shown` — props: `totalLost`
- `report.entry_points.lost_revenue.day_clicked` — props: `day`, `lost`
- `report.entry_points.export.clicked`

---

## TypeScript-скелет (extension)

```ts
type CreateEntryPointsReportResponse = {
  reportId: string;
  status: 'ready';
  insightsStatus: 'pending' | 'running' | 'ready' | 'failed' | 'skipped';
  report: SyncReport;
};

type EntryPointsInsightsResponse = {
  reportId: string;
  insightsStatus: CreateEntryPointsReportResponse['insightsStatus'];
  insights: InsightCard[];
  actionPlan: { summary: string; steps: [string, string, string] } | null;
  regionHourInsight: {
    bestSlot: { region: string; hour: number; intensity: number };
    worstSlot: { region: string; hour: number; intensity: number };
    recommendation: string;
  } | null;
  errorMessage: string | null;
};

async function buildEnrichedReport(payload: CreateEntryPointsReportRequest) {
  const created = await api.post<CreateEntryPointsReportResponse>(
    '/entry-points/reports',
    payload,
  );
  // сразу: created.report

  if (
    created.insightsStatus === 'skipped' ||
    created.insightsStatus === 'ready' ||
    created.insightsStatus === 'failed'
  ) {
    return { report: created.report, insights: await fetchInsights(created.reportId) };
  }

  const insights = await pollInsights(created.reportId);
  return { report: created.report, insights };
}
```

Типы лучше генерировать из OpenAPI (`tag: entry-points`).

---

## Чеклист приёмки клиента

- [ ] POST без токена → 401; без фичи → 403
- [ ] Валидный payload → 202, таблица и баннер без ожидания LLM
- [ ] Полл insights до `ready` / `failed` / `skipped`
- [ ] `compareEnabled` + `previous` → дельты KPI не `NaN`/`Inf`
- [ ] Точка с `orders > 0` без `avgPrice` → понятная ошибка
- [ ] Повторный POST того же payload → тот же `reportId`
- [ ] Excel строится из sync (+ insights при наличии), не ждёт серверный файл
- [ ] В UI нет внутренних имён API / статусов job / «OpenRouter»

---

## Связанные документы

- Архитектура модуля: [../architecture/entry-points.md](../architecture/entry-points.md)
- Auth / MFA: [auth-mfa-client.md](./auth-mfa-client.md)
- Entitlements расширения: OpenAPI tag `extension`, `GET /extension/entitlements`
