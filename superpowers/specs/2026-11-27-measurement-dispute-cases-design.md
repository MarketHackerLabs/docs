# Кейсы оспаривания обмеров (parent case + child jobs)

Дата: 2026-11-27  
Статус: approved for planning  
Базовый дизайн: [2026-11-26-measurement-disputes-design.md](./2026-11-26-measurement-disputes-design.md)  
Архитектура: [docs/architecture/measurement-disputes.md](../../architecture/measurement-disputes.md)  
Клиентский контракт: [docs/integrations/measurement-disputes-extension.md](../../integrations/measurement-disputes-extension.md)

## Цель

В списке и виджете пользователь видит **одно оспаривание** (кейс по отчёту), а не отдельные ИИ-запуски «анализ фото» и «разбор ответа». Внутри кейса — история и результаты этих запусков. Открытие из списка ведёт в workspace **этого** кейса, а не в пустой `/measurement-disputes/new`.

## Решения (зафиксировано)

| Тема | Решение |
|------|---------|
| Модель | Родительский кейс + дочерние ИИ-jobs (как сейчас `measurement_disputes`) |
| Граница кейса | Новый файл / «начать заново» → новый кейс; повторные анализы фото и ответа в том же отчёте → тот же кейс |
| Хранение на сервере | Шапка кейса + payloads/результаты jobs; полный выбор обмеров и черновик первого обращения — в браузере |
| Открытие без XLSX | Шапка + лента jobs/результатов; полный сценарий выбора/первого обращения — если есть локальный черновик для `caseId` или после повторной загрузки файла в этот кейс |
| Старые данные | Полное удаление всех существующих `measurement_disputes`; перед удалением — `release` для строк с `quota_reserved = true` |
| Мурлики | Без изменения весов и flow reserve/commit/release; единица списания по-прежнему job |
| DELETE кейса | Вне v1 (достаточно «начать заново») |
| Breaking API | Старые list/active/create/`/{id}` на уровне «job = сущность списка» убираются; префикс `/measurement-disputes` сохраняется |

## Не цели

- Загрузка сырого XLSX на сервер.
- Персистенция полного workbook / всех строк отчёта на кейсе.
- Миграция старых jobs в синтетические кейсы.
- Изменение промптов, схем результатов LLM, admin settings модуля.
- Правки кода `extension-chrome` (только документация контракта).

## Домен

```
measurement_dispute_cases  1 ─── *  measurement_disputes (jobs)
```

### Кейс `measurement_dispute_cases`

| Поле | Тип | Назначение |
|------|-----|------------|
| `id` | UUID PK | Идентификатор оспаривания |
| `user_id` | UUID FK → users | Владелец |
| `organization_id` | UUID FK → organizations \| null | Контекст seat при создании (опционально) |
| `title` | string | Подпись в списке (имя файла или «Демо-отчёт») |
| `file_label` | string | Как на клиенте `WorkspaceFileMeta.fileLabel` |
| `sheet_name` | string | Имя листа |
| `is_demo` | bool | Демо-отчёт |
| `report_summary` | JSONB | Итоги листа в форме клиентского `FileTotals`: `operations`, `uniqueMeasurements`, `articles`, `accrued`, `storno`, `net`, `moneyWarnings` |
| `created_at` / `updated_at` | timestamptz | Список, сортировка; `updated_at` при создании job и при PATCH шапки (если появится) |

Индексы: `(user_id, created_at DESC)`.

**Статус кейса** — только вычисляемый в API (колонки нет), тип:
`draft` \| `active` \| `succeeded` \| `failed` \| `cancelled`:

1. Jobs нет → `draft`.
2. Есть хотя бы один job `queued` или `running` → `active`.
3. Иначе (все терминальные): если есть любой `failed` → `failed`; иначе если есть любой `succeeded` → `succeeded`; иначе → `cancelled`.

Детали по отдельным запускам — в `jobs[]`.

### Job `measurement_disputes`

Существующая таблица + обязательное поле:

| Поле | Изменение |
|------|-----------|
| `case_id` | UUID FK → `measurement_dispute_cases.id`, `ON DELETE CASCADE`, `NOT NULL` |
| Остальное | Без изменений (`kind`, `status`, payloads, quota, timestamps) |

Индекс: `(case_id, created_at DESC)`.

Worker `run_measurement_dispute` продолжает принимать **id job**, не case.

## API

Базовый префикс: `/api/v1/measurement-disputes`.  
Auth / feature / client header — как в базовом дизайне.

### Кейсы

| Метод | Путь | Код | Назначение |
|-------|------|----:|------------|
| `POST` | `/cases` | 201 | Создать кейс при загрузке отчёта / «начать заново» |
| `GET` | `/cases` | 200 | Пагинированный список кейсов пользователя |
| `GET` | `/cases/active` | 200 | Кейсы с хотя бы одним job `queued`/`running` |
| `GET` | `/cases/{caseId}` | 200 | Кейс + summaries вложенных jobs (без `result_payload`) |

Тело `POST /cases` (camelCase):

| Поле | Тип | Обязательный | Описание |
|------|-----|:------------:|----------|
| `title` | string ≤ 256 | да | Подпись |
| `fileLabel` | string ≤ 512 | да | Имя файла / демо |
| `sheetName` | string ≤ 128 | да | Лист |
| `isDemo` | bool | нет (default false) | |
| `reportSummary` | object | да | Поля `FileTotals` |
| `organizationId` | UUID \| null | нет | На будущее / единообразие; квоту резервирует job |

Ответ кейса (summary / detail):

```json
{
  "id": "…",
  "title": "Отчёт по удержаниям.xlsx",
  "fileLabel": "Отчёт по удержаниям.xlsx",
  "sheetName": "Лист1",
  "isDemo": false,
  "reportSummary": {
    "operations": 153,
    "uniqueMeasurements": 12,
    "articles": 10,
    "accrued": 165804.5,
    "storno": -40401.2,
    "net": 125403.3,
    "moneyWarnings": 0
  },
  "status": "active",
  "jobs": [
    {
      "id": "…",
      "kind": "photo_analysis",
      "status": "succeeded",
      "quotaUnits": 9,
      "errorMessage": null,
      "createdAt": "…",
      "startedAt": "…",
      "finishedAt": "…"
    }
  ],
  "createdAt": "…",
  "updatedAt": "…"
}
```

В `GET /cases` (список): `status` обязателен; `jobs` — полный список summaries (объём мал: единицы запусков на кейс). То же для `GET /cases/{caseId}` и `GET /cases/active`.

Чужой `caseId` → `404`.

### Jobs

| Метод | Путь | Код | Назначение |
|-------|------|----:|------------|
| `POST` | `/cases/{caseId}/jobs` | 202 | Создать ИИ-job (тело = бывший create по `kind`) |
| `POST` | `/estimate-cost` | 200 | Без изменений |
| `GET` | `/jobs/{jobId}` | 200 | Статус job |
| `GET` | `/jobs/{jobId}/result` | 200 | Результат при `succeeded` |
| `POST` | `/jobs/{jobId}/cancel` | 200 | Отмена `queued` + release |

Тело `POST …/jobs` — как нынешние `AnalyzeResponseRequest` / `PhotoAnalysisRequest` (дискриминатор `kind`), без отдельного `caseId` в теле (он в path).  
`organizationId` в теле job по-прежнему для resolve квоты.

Ответы job summary/result — как сейчас у `MeasurementDisputeSummary` / result payload.

### Удаляемые эндпоинты

- `POST /measurement-disputes` (create job без case)
- `GET /measurement-disputes` (list jobs)
- `GET /measurement-disputes/active` (active jobs)
- `GET /measurement-disputes/{id}`
- `GET /measurement-disputes/{id}/result`
- `POST /measurement-disputes/{id}/cancel`

Admin settings без изменений.

## Manager Portal

| Маршрут | Поведение |
|---------|-----------|
| `/measurement-disputes` | Список **кейсов**; клик → `/measurement-disputes/{caseId}` |
| `/measurement-disputes/new` | Загрузка/демо → `POST /cases` → `router.replace(/measurement-disputes/{caseId})` |
| `/measurement-disputes/{caseId}` | Workspace кейса: прогресс и результаты ИИ inline |

Sidebar: «Оспаривание обмеров» → `/measurement-disputes` (список). С `/new` и `{caseId}` — ссылка «К списку».

Виджет активных задач: **один пункт на активный кейс**, `href = /measurement-disputes/{caseId}`.

### Черновик браузера

- Ключ: `mh_measurement_dispute_workspace:{caseId}`.
- Содержимое: как нынешний `MeasurementWorkspaceDraft` (форма, выбор, wb response, fileMeta, job ids/results).
- Без ключа для `caseId`: UI показывает шапку с API + jobs/результаты; выбор/первое обращение — после загрузки XLSX **в этот кейс** (тот же `caseId`, без `POST /cases`) или «Начать заново» → `/new` → новый кейс.

Повторный анализ фото / ответа WB создаёт **новый** job под тем же `caseId` (как сейчас по смыслу «Повторное»).

## Миграция

Один Alembic revision:

1. Для каждой строки `measurement_disputes` с `quota_reserved = true` выполнить тот же `release`, что при cancel (через сервис квоты / SQL+ledger API — как принято в проекте для data migrations). Не оставлять «висящий» резерв.
2. `DELETE FROM measurement_disputes` (полное очищение).
3. Создать `measurement_dispute_cases`.
4. Добавить `case_id` UUID NOT NULL FK на jobs (таблица jobs пуста — без backfill).
5. Индексы.

Документация: обновить `docs/architecture/measurement-disputes.md` и `docs/integrations/measurement-disputes-extension.md` в том же изменении продукта.

## Тесты (минимум)

- Create case → create photo job + analyze_response job → `GET /cases` одна строка; `GET /cases/{id}` два job summary.
- `GET /cases/active` возвращает кейс при `queued`/`running` job; не возвращает после terminal.
- Create job на чужой/несуществующий case → `404`.
- Cancel queued job → release; case status перестаёт быть `active`.
- Миграция: при наличии reserved job — после upgrade резерва нет, таблица jobs пуста, cases пуста.
- Portal (unit/integration по принятому в portal стилю): draft keyed by caseId; open caseId не редиректит на `/new`.

## Совместимость с базовым дизайном 2026-11-26

Этот документ **заменяет** модель «job = элемент списка» в базовом дизайне. Неизменным остаётся:

- XLSX только в браузере;
- kinds, LLM pipeline, PII scrub, веса Мурликов, admin settings;
- feature `measurement_dispute`.
