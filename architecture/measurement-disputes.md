# Оспаривание обмеров WB

## Назначение

Модуль `measurement_dispute` помогает селлеру оспорить контрольный обмер Wildberries. Отчёт «Отчёт по удержаниям» (XLSX) разбирается **в браузере**: итоги по файлу, выбор обмера, срок, первое обращение по шаблону. На backend уходят только две ИИ-задачи, оплачиваемые **Мурликами**:

| `kind` | Что делает |
|--------|------------|
| `analyze_response` | Разбор ответа поддержки WB: категория, аргумент WB, короткий ответ и подробная повторная претензия |
| `photo_analysis` | Вердикт по фото каждого активного обмера: `appeal`, `borderline`, `no_reason`, `insufficient` |

MarketHacker ничего не отправляет в кабинет WB. Пользователь копирует текст и отправляет его сам.

Требования: [tz_osparivanie.md](../../tz_osparivanie.md) (v1.0 + §22). Решения по реализации: [дизайн-спека](../superpowers/specs/2026-11-26-measurement-disputes-design.md).

## Что где происходит

| Этап | Где | ИИ |
|------|-----|:--:|
| Разбор XLSX, листы, колонки, группировка по «номенклатура + номер замера», «Итого по файлу» | Manager Portal (браузер) | нет |
| Срок: дата размещения отчёта + 7 календарных дней | клиент | нет |
| Первое обращение (один обмер или общая претензия) | клиент, шаблон | нет |
| Разбор ответа WB | backend + worker | да, текст |
| Анализ фото обмеров | backend + worker | да, vision |

Исходный XLSX на сервер и в модель не попадает. В запрос уходят выбранные структурированные поля, текст ответа WB и ссылки на фото.

Клиенты:

- Manager Portal `/measurement-disputes` — реализовано (список, `/new`, `/[id]`).
- Chrome extension — код в этой поставке **не менялся**. Контракт и приёмка: [measurement-disputes-extension.md](../integrations/measurement-disputes-extension.md).

## Поток

```mermaid
flowchart LR
  Client -->|estimate-cost| API
  Client -->|POST kind + payload| API
  API -->|PII scrub, reserve Мурлики, enqueue| ARQ
  ARQ --> Worker[run_measurement_dispute]
  Worker --> LLM[OpenRouter text / vision]
  LLM -->|JSON по схеме| Worker
  Worker -->|commit или release| PG[(measurement_disputes)]
  Client -->|GET /id, GET /id/result| API
```

1. Клиент вызывает `POST /estimate-cost` и показывает стоимость в Мурликах.
2. `POST /measurement-disputes` проверяет, включён ли `kind` в настройках, лимит пачки фото и фичу тарифа. Затем чистит текст от персональных данных, резервирует Мурлики, создаёт задачу `queued` и ставит её в очередь. Ответ — `202`.
3. Worker переводит задачу в `running` и вызывает OpenRouter. При невалидном JSON делается **одна** повторная попытка с подсказкой о нарушении схемы.
4. Успех: `result_payload`, статус `succeeded`, **commit** Мурликов.
5. Ошибка модели, невалидный JSON дважды, таймаут пайплайна (1500 с), прерывание: статус `failed`, безопасный `errorMessage`, **release** Мурликов.
6. Отмена (`POST /{id}/cancel`) возможна пока задача `queued`; резерв возвращается. Для `running` — `409`.

Фото: по одному запросу на обмер, до 4 параллельно. Если один обмер не удалось проанализировать, вся задача `failed`, Мурлики возвращаются. Агрегаты (счётчики и суммы по вердиктам) считаются кодом, а не моделью.

## API

Prefix: `/api/v1/measurement-disputes`. Auth: `Authorization: Bearer <accessToken>`. Доступ: активный пользователь + фича тарифа `measurement_dispute`. Ответы — camelCase, обёртка `{ "data": …, "meta": {} }`. Подробные поля — в [клиентской доке](../integrations/measurement-disputes-extension.md#3-контракт-api).

| Метод | Путь | Назначение |
|-------|------|------------|
| `POST` | `/estimate-cost` | Стоимость в Мурликах без создания задачи |
| `POST` | `/` | Создать задачу → `202`, резерв Мурликов |
| `GET` | `/` | Список задач пользователя (`page`, `pageSize` ≤ 100, `status`) |
| `GET` | `/active` | Задачи `queued` / `running` (виджет «Задачи в работе») |
| `GET` | `/{id}` | Статус и метаданные |
| `GET` | `/{id}/result` | Результат; `409`, пока статус не `succeeded` |
| `POST` | `/{id}/cancel` | Отмена с возвратом резерва |

Admin (superuser):

| Метод | Путь | Назначение |
|-------|------|------------|
| `GET` / `PATCH` | `/api/v1/admin/measurement-dispute/settings` | Параметры модуля |

Чужая задача отдаётся как `404`.

## Данные

Таблица `measurement_disputes`: `user_id`, `organization_id` (организация, с квоты которой зарезервированы Мурлики), `kind`, `status` (`queued` \| `running` \| `succeeded` \| `failed` \| `cancelled`), `request_payload` и `result_payload` (JSONB), `prompt_version`, `model`, `usage_input_tokens` / `usage_output_tokens`, `quota_reserved`, `quota_units`, `error_code`, `error_message`, `created_at` / `started_at` / `finished_at`.

Таблица `measurement_dispute_settings` — одна строка (`id = 1`).

`request_payload` хранит структурированный JSON, без XLSX. Полный текст ответа WB допустим там для повторного прогона, но не пишется в логи и аналитику.

## Мурлики и биллинг

Фича тарифа: `measurement_dispute` («Оспаривание обмеров WB»), `user_or_org_seat`. Ключ нужно включить в нужных тарифах при выкладке.

Веса в каталоге (`/billing/mh-credit-weights`, оверрайды — в админке):

| `action_key` | `param_key` | Default | Что оплачивает |
|--------------|-------------|--------:|----------------|
| `measurement_dispute_analyze_response` | `base` | 5 | Одна задача разбора ответа WB |
| `measurement_dispute_photo_analysis` | `per_measurement` | 3 | Один обмер с фото в задаче |

Стоимость:

- `analyze_response` = `base`;
- `photo_analysis` = `measurement_count × per_measurement`, где `measurement_count` — число элементов `items` в задаче.

Списание: `resolve` + `reserve` при создании → `commit` при `succeeded` → `release` при `failed` и `cancelled`. Нет фичи — `403 PERMISSION_DENIED`. Не хватает Мурликов — `402 PLAN_LIMIT_EXCEEDED`. Несколько организаций с квотой и не передан `organizationId` — `409` с `details.code = "quota_org_required"`. Квоты работают так же, как у [card-audit](./card-audit.md) и [reviews-analysis](./reviews-analysis.md); общий каталог весов — в [billing.md](./billing.md#мурлики-и-веса).

## Настройки модуля

Admin-panel: `/measurement-dispute/settings`. API: `GET` / `PATCH /api/v1/admin/measurement-dispute/settings`. `PATCH` частичный: непереданное поле не меняется, пустая `ruleSourceUrl` очищает источник.

| Поле | Тип | Default | Назначение |
|------|-----|---------|------------|
| `llmModel` | string | `openai/gpt-5.4` | Модель OpenRouter для разбора ответа WB |
| `photoLlmModel` | string | `openai/gpt-5.4` | Vision-модель для фото |
| `promptVersion` | string | `v1` | Версия промпта разбора ответа |
| `photoPromptVersion` | string | `v1` | Версия промпта фото |
| `wbResponseMaxChars` | int | 8000 | Сколько символов ответа WB уходит в модель (остальное отрезается) |
| `maxBulkPhotos` | int | 50 | Максимум обмеров в одной фото-задаче |
| `ruleContext` | string | пусто | Проверенная выдержка из правил WB; модель не выдумывает правила |
| `ruleSourceUrl` | string \| null | `null` | Ссылка на источник выдержки |
| `analyzeResponseEnabled` | boolean | `true` | Разрешён ли `analyze_response` |
| `photoAnalysisEnabled` | boolean | `true` | Разрешён ли `photo_analysis` |

Секреты (`OPENROUTER_API_KEY` и связанные) — только в env backend. В клиентский бандл не попадают. Перед production `ruleContext` нужно сверить с актуальной офертой и инструкцией WB (ТЗ §21).

## Приватность

- XLSX читается только в браузере и не загружается.
- Перед моделью `scrub_pii` заменяет на `[УДАЛЕНО]` e-mail, телефоны, Bearer/JWT и похожие на токены строки в `wbResponseText`, `discrepancyReason` и переданном клиентом `ruleContext`.
- В фото-задаче модели уходят: фото, номер обмера, артикул, габариты, процент расхождения. Сумма в модель не передаётся.
- В технические логи пишутся `request_id`, категория, длительность, статус, `prompt_version`. Текст ответа WB и размеры товара не логируются.
- Продуктовая аналитика клиента не содержит текст обращений и значения из XLSX.
- Ссылки на фото принимаются только `https://`.
- Ответы с ошибкой содержат понятное сообщение без внутренних кодов; код причины сохраняется в `error_code` для логов.

## Файлы

| Область | Путь |
|---------|------|
| Backend-модуль | `backend/src/markethacker/modules/measurement_dispute/` |
| Входные схемы | `…/application/dto.py` |
| API-схемы | `…/api/schemas.py` |
| Схемы результата ИИ | `…/domain/result_schemas.py` |
| Worker job | `…/infrastructure/jobs/run_dispute.py` (`run_measurement_dispute`) |
| Веса Мурликов | `backend/src/markethacker/modules/billing/domain/mh_credit_weights.py` |
| Manager Portal | `manager-portal/src/app/(manager)/measurement-disputes/`, `manager-portal/src/lib/measurement-dispute/` |
| Admin-panel | `admin-panel/src/app/(admin)/measurement-dispute/settings/` |
