# Redesign: аудит карточек v2 + визуал анализа отзывов

Дата: 2026-10-29  
Статус: approved for planning  

## Цель

Переработать аудит карточек товара (системный промпт + визуализация отчёта) и визуализацию анализа отзывов по эталонам в `new-audit-and-new-reviews-analyze/`. Результат доступен в Manager Portal с корректной A4-печатью; маркетинговый мок анализа отзывов в market-navigators синхронизирован. Chrome-расширение не трогаем.

## Решения (зафиксировано)

| Тема | Решение |
|---|---|
| Визуализация | UI в приложении + `@media print` / A4; без серверной генерации PDF |
| Старые аудиты | Единый UI v2 + адаптер v1→view-model; без перегонки LLM |
| Генерация из аудита | Пользователь выбирает кадр из `generationPrompts[]` |
| Страницы аудита | Продуктовые блоки эталона; без archive/trace |
| Картинки в visual evidence | Реальные URL галереи (`imageUrls`), не SVG-заглушки |
| Поверхности | Manager Portal + market-navigators (reviews mock); admin — сырой JSON |
| Подход реализации | Print-first листовые секции + shared layout-kit |
| Промпт аудита | Контракт и frame из `wb_card_audit_prompt_v2.txt`; expert md через существующий `knowledge_loader` |
| Анализ отзывов (backend) | Схему и промпты не меняем |

## Эталоны (источник правды)

| Артефакт | Путь |
|---|---|
| Промпт аудита v2 | `new-audit-and-new-reviews-analyze/wb_card_audit_prompt_v2.txt` |
| Макет аудита | `new-audit-and-new-reviews-analyze/audit_report_final.html` (+ `.pdf` как визуальный референс) |
| Макет отзывов | `new-audit-and-new-reviews-analyze/reviews_analysis_redesign.html` (+ `.pdf`) |

## Scope

### In

**Аудит карточек**

1. Backend: обновить system frame + JSON-контракт ответа в `prompts.py` до v2; расширить `CardAuditReport` и связанные модели; parser/pipeline/result payload; совместимость чтения v1.
2. Manager Portal: пересобрать отчёт под продуктовые A4-блоки эталона + print CSS; превью реальных кадров галереи; выбор frame-промпта для card-generate.
3. Handoff в card-generate: передача выбранного промпта / `frameId` (не авто-первый).
4. Типы в `manager-portal/src/lib/api.ts`; при необходимости точечные правки страницы detail.
5. Документация архитектуры `docs/architecture/card-audit.md` (и card-generate, если меняется handoff).

**Анализ отзывов**

1. Manager Portal: переверстать `reviews-analysis-report.tsx` под 5 страниц эталона + print CSS.
2. market-navigators: обновить `src/routes/reviews-analysis.tsx` под тот же визуальный язык (мок-данные).

### Out

- `extension-chrome` (любые правки).
- Серверный PDF / отдельный export-pipeline.
- Страницы archive / audit-trace из эталона аудита.
- Миграция/пересчёт старых аудитов через LLM.
- Rich-UI в admin-panel (остаётся JSON).
- Отдельный marketing-мок аудита карточек в market-navigators (сейчас его нет — не создавать без отдельной задачи).
- Изменение биллинга / весов мурликов / фичефлагов без необходимости.
- Изменение LLM-схемы и промптов `reviews_analysis` (кроме случая, если при реализации всплывёт блокер отображения — тогда отдельное согласование).

## Архитектура

```
Эталон v2 prompt + HTML
        │
        ▼
card_audit prompts.py ──► LLM ──► CardAuditReport (v2) ──► result JSONB
        │                              │
        │                              ▼
        │                     view-model adapter (v1|v2)
        │                              │
        └──── knowledge_loader ──►     ▼
                              Manager Portal A4 report (+ print)
                                          │
                                          └─► card-generate (выбранный frame)

reviews_analysis (без смены контракта)
        │
        ▼
Manager Portal A4 report (+ print)
        │
        └─► market-navigators mock (тот же layout)
```

## Backend: аудит карточек

### Промпт

- Файл: `backend/src/markethacker/modules/card_audit/infrastructure/llm/prompts.py`.
- Заменить `_SYSTEM_FRAME` и `_RESPONSE_SCHEMA` содержимым контракта из `wb_card_audit_prompt_v2.txt` (версия инструкции 2.0, маркировка доказательности, запрет масштабирования рекламы, JSON-only).
- Блок экспертной базы **не** вшивать целиком в `prompts.py`: по-прежнему `knowledge_loader.load_expert_knowledge_block()` + категорийный reference.
- При расхождении «Визуальный разбор» (thumbnailRef, annotations…) и упрощённого `slides` в хвосте эталонного txt — **расширить схему `slides`** (и при необходимости отдельные поля visual evidence), чтобы UI мог показать кадр + разбор. Координаты аннотаций — optional; если нет — только текстовые пометки.

### Модель ответа (ключевые поля v2)

Добавить / расширить (camelCase в JSON):

- `reportVersion`: `"2.0"`
- `platform`, `mode`, `advertisingDecision`
- `confidenceScore`
- `scoreBreakdown[]`: component, score (nullable), weight, confidence, evidence, deductions
- `signals[]`: mentions, sampleSize, share, variants, examples, confidence (+ совместимость со старым `count`)
- `prioritizedActions[]`: actionId, priority, action, impact, effort, confidence, dependencies, requiredAssets, ownerRole, acceptanceCriteria, metric, doNotStartBefore
- `expectationGaps[]`, `returnRisk`, `funnelDiagnosis`
- `productLock[]`, `generationPrompts[]`, `productLockQa`
- `reportLayout[]` (порядок секций; UI может использовать как подсказку)
- `slides[]`: сохранить n/title/verdict/state/notes; добавить optional `thumbnailRef`, role, evidence, annotations, actionId, acceptanceCriteria

Обратная совместимость записи:

- Поля v1 (`actionsP0`, `actionsP1`, `generationPrompt`, `roadmap`, …) оставить опциональными или заполнять адаптером из v2 при необходимости downstream.
- В `report_to_result_payload`: отдавать `generationPrompts`; для v1 сохранять чтение `generationPrompt`.

### Адаптер v1 → view-model (чтение)

На стороне portal (и при необходимости тонкий слой в API types):

- Если нет `reportVersion` / `scoreBreakdown` — считать v1.
- Score ring из `score`; confidence скрыть или «нет данных».
- `actionsP0`/`actionsP1` → список действий без полных v2-метаданных.
- `generationPrompt` → один псевдо-кадр `ready` для выбора в генерацию.
- `slides` без URL: подставить `bundle.ownCard.imageUrls[index]` по порядку.
- Блоки expectation/return/funnel/productLock — не рендерить, если пусто.

### Handoff в card-generate

- UI аудита: список `generationPrompts` со статусами; CTA только для `ready` и `draft` (`blocked` — показать причину/`missingReferences`, без запуска).
- Query/body в создание генерации: явный текст выбранного промпта и/или `frameId` + `auditId`.
- Старый deep-link `?auditId=` без frame: fallback — первый `ready`, иначе первый `draft`, иначе legacy `generationPrompt`; не брать `blocked`.

## Frontend: аудит карточек

### Структура отчёта (продуктовые листы)

Ориентир — `audit_report_final.html`, без archive/trace:

1. Executive: headline, summary, metrics strip, score + confidence, top risks, top actions, scoreBreakdown, strengthsToKeep  
2. Visual evidence: сетка кадров с реальными изображениями + status + notes; таблица обоснований при наличии  
3. Expectation gaps + return risk + review signals / missing data  
4. Funnel diagnosis + competitors + critical data requests  
5. Prioritized actions + A/B tests  
6. Production pack: productLock + frame prompts (+ QA checklist)  
7. Audiences / objections / full gallery table / recommended funnel + video / content strategy + SWOT / text bank / limitations — по данным, печать разбивать на листы аналогично эталону  

### Print

- Контейнеры «страница» ~A4, `break-after: page`, стили как в эталоне (токены brand lime, paper, pills).
- Кнопка «Печать» может использовать `window.print()`; отдельный PDF-файл не генерируем.

### Visual evidence ↔ изображения

- Источник URL: `result.bundle` / compact own card `imageUrls` (уже есть в пайплайне).
- Сопоставление: `thumbnailRef` если валидный URL/индекс; иначе `imageUrls[n-1]` / по порядку слайда.
- Если URL недоступен — плейсхолдер с понятным текстом («Нет изображения»), без внутренних ошибок.

## Frontend: анализ отзывов

### Данные

Без изменения backend-контракта. Маппинг:

| Лист эталона | Поля |
|---|---|
| 01 Обзор | `summary`, `stats.*`, `llm.sentiment`, `llm.recommendations` |
| 02 Темы | `llm.strengths`, `llm.weaknesses` |
| 03 Вопросы | `llm.questions.items` |
| 04 Инфографика + товар/упаковка | `infographicRecommendations`, `recommendationSections` |
| 05 Резюме | `finalSummary` |

UX относительно текущего portal:

- Всегда показывать `summary` (lead), даже если есть recommendations.
- В секциях рекомендаций показывать title и recommendation раздельно.
- Визуальный язык: lime / violet как в эталоне; 5 A4-листов + print CSS.

### market-navigators

Обновить только `market-navigators/src/routes/reviews-analysis.tsx` под тот же layout. Мок аудита карточек не добавлять.

## Ошибки и пустые состояния

Пользовательский UI (правило user-facing):

- Что произошло / что сделать дальше.
- Без названий моделей, stack trace, внутренних id схем.
- Недостаток данных — явные «нет данных» / скрытие блока, без выдуманных метрик.

## Тестирование (критерии приёмки)

1. Новый аудит проходит LLM → валидируется как v2 (`reportVersion`, новые блоки при наличии данных).
2. Старый аудит v1 открывается в новом UI без падения; legacy generation prompt доступен для генерации.
3. В visual evidence видны реальные превью галереи (при наличии URL).
4. Выбор кадра `ready`/`draft` создаёт/открывает card-generate с этим промптом; `blocked` не запускается.
5. Печать аудита и отзывов даёт читаемые A4-листы без обрезания критичных блоков.
6. Отчёт отзывов визуально соответствует 5 листам эталона на тех же полях данных.
7. Marketing mock отзывов обновлён; chrome extension не изменён.

## Риски

- Объём JSON v2 увеличивает риск обрезания/галлюцинаций LLM → жёсткая валидация Pydantic + понятный fail на parse.
- Расхождение эталонного txt (visual fields vs slides schema) → в реализации зафиксировать одну расширенную схему slides/visual.
- Печать длинных отчётов в браузере нестабильна → приоритет break-inside/avoid на карточках, ручная разбивка на `.page`.

## Открытых вопросов нет

Все продуктовые развилки закрыты ответами пользователя (B/B/B/A/B + A для картинок + ok по секциям дизайна).
