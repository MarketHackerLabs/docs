# Отчёт «Точки входа» (enrichment API)

Модуль: `modules/entry_points/`.

Chrome extension сам собирает данные WB Seller API и отправляет
нормализованный payload. Backend считает упущенную выручку, KPI, heatmaps,
rules и асинхронно обогащает инсайты через OpenRouter.

Клиентская интеграция (контракт, полл, маппинг UI):  
[../integrations/entry-points-client.md](../integrations/entry-points-client.md).

## Эндпоинты

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| POST | `/api/v1/entry-points/reports` | JWT + `browser_extension` | Sync-отчёт, 202 |
| GET | `/api/v1/entry-points/reports/{reportId}` | JWT + `browser_extension` | Sync-часть |
| GET | `/api/v1/entry-points/reports/{reportId}/insights` | JWT + `browser_extension` | Полл LLM |

## Двухфазный поток

1. **Sync** — валидация payload → P75 CTR → `lostRub` → KPI/дельты → day heatmap →
   region×hour normalize → rules (§12) → PG row + Redis payload (TTL 2ч) →
   enqueue ARQ `enrich_entry_points_insights` (если есть кандидаты / matrix).
2. **Async** — OpenRouter (`google/gemini-2.5-flash`, fallback `2.0-flash`) →
   инсайты 17.1 → action plan 17.3 → region hour 17.2 → PG + Redis insights (TTL 6ч).

`insightsStatus`: `pending` → `running` → `ready` | `failed` | `skipped`.

При `failed` sync-отчёт остаётся валидным; GET insights отдаёт rules-only тексты.

## Формула упущенной выручки

Канон — ТЗ (без ниши):

- `targetCtr` = P75 CTR верхнеуровневых точек с `shows ≥ 50`
- lost = 0 если `targetCtr ≤ 0`, `shows < 50` или `ctrFact ≥ targetCtr`
- иначе `missedClicks × crSO × price × weight`
- `price` = `avgPrice` точки или обязательный `meta.fallbackAvgPrice` (без хардкода 900)
- итог Σ только по верхнему уровню; деньги `ROUND_HALF_UP`, 2 знака на точке

## Payload (кратко)

Обязательно: `meta.fallbackAvgPrice`, `meta.sellerKey`, период, `current.points[]`.

У точки с `orders > 0` обязателен `avgPrice`. Опционально: `previous`,
`activityByRegionHour`, `daily[]` по точкам.

Лимиты: ≤ 80 верхнеуровневых точек, ≤ 400 детей, период ≤ 31 день.

## Идемпотентность и кэш

Ключ: `userId + sellerKey + period + filtersHash + payloadHash`.

Повторный POST в окне TTL возвращает тот же `reportId` без дубля LLM job,
если insights уже `ready`/`running`/`skipped`.

## Admin: модель LLM

| Метод | Путь | Доступ |
|-------|------|--------|
| GET | `/api/v1/admin/entry-points/settings` | superuser |
| PATCH | `/api/v1/admin/entry-points/settings` | superuser |

Singleton `entry_point_settings.llm_model` (default `google/gemini-2.5-flash`).  
Job enrichment читает модель отсюда; fallback при ошибке OpenRouter — константа `google/gemini-2.0-flash`.

Admin UI: раздел «Точки входа» → «Параметры».

## Вне scope v1

- Excel (остаётся в extension)
- Server-side fetch WB / wb_gateway
- Отдельный billing feature key
- Retry endpoint для insights
