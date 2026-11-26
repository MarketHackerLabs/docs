# Оспаривание обмеров WB (measurement-disputes)

Дата: 2026-11-26  
Статус: approved for planning  
Источник требований: `tz_osparivanie.md` (v1.0 + §22)

## Цель

Инструмент в TEAM (manager-portal), который из локального XLSX «Отчёт по удержаниям» WB строит проверяемый сценарий оспаривания контрольного обмера: итоги по файлу, первое обращение без ИИ, async-разбор ответа поддержки и async-фотоанализ активных обмеров за Мурлики. Отправку в кабинет WB MarketHacker не выполняет.

## Решения (зафиксировано)

| Тема | Решение |
|---|---|
| Поверхность продукта | manager-portal (`team.markethacker.ru`), не market-navigators |
| Маршрут | `/measurement-disputes` (без префикса `wb-`) |
| Расширение | Код `extension-chrome` не меняем; только интеграционная документация (вариант C) |
| Переиспользование UI с расширением | Нет общего React-пакета; расширение реализует свой UI по API/UX-доке |
| Модель выполнения ИИ | Async jobs по паттерну card-audit (вариант 2) |
| Провайдер LLM | OpenRouter (как остальные AI-модули); не прямой OpenAI / не `OPENAI_API_KEY` на клиенте |
| API prefix | `/api/v1/measurement-disputes` |
| Feature flag | `measurement_dispute` (`user_or_org_seat`) |
| Биллинг | Веса Мурликов в каталоге + оверрайды в админке; settings модуля в admin-panel |
| XLSX | Только в браузере; на сервер — структурированный JSON выбранных строк / URL фото |

## Scope

### In

1. **Backend** — модуль `measurement_dispute`: ORM job + settings, публичный API, admin API, ARQ worker, OpenRouter pipeline, Pydantic-схемы результатов, PII scrub, квоты Мурликов.
2. **Billing** — feature `measurement_dispute`; action keys и `compute_cost` для text-анализа и фотоанализа; отображение в `/billing/mh-credit-weights`.
3. **Admin-panel** — страница параметров модуля (модель, prompt_version, лимиты, rule_context); веса через существующий UI весов.
4. **Manager-portal** — страницы list / new / `[id]`; локальный XLSX-парсер (§4 + §22); детерминированное первое обращение; UI запуска jobs и результатов; sidebar + plan feature.
5. **Документация** — `docs/architecture/measurement-disputes.md`, `docs/integrations/measurement-disputes-extension.md`.
6. **Тесты** — unit/integration/failure по ТЗ §19 (E2E расширения — только чеклист в доке).

### Out

- Любые правки `extension-chrome`.
- Страницы/демо в `market-navigators`.
- Автоотправка текстов в WB.
- Загрузка исходного XLSX на сервер или в LLM.
- Vision/web-search сверх фотоанализа по URL, описанного в §22.
- Общий npm-пакет UI между portal и extension.

## Архитектура

```
[Manager Portal /measurement-disputes]
  │  local xlsx parse (SheetJS)
  │  first appeal template (deterministic)
  │
  ├─ POST /api/v1/measurement-disputes/estimate-cost
  ├─ POST /api/v1/measurement-disputes          → reserve Мурлики → ARQ job
  ├─ GET  /api/v1/measurement-disputes/{id}
  └─ GET  /api/v1/measurement-disputes/{id}/result
                    │
                    ▼
         measurement_dispute worker
                    │
         OpenRouter (text | vision)
                    │
         validate JSON → commit|release credits
                    │
                    ▼
         result_payload (JSONB)

[Admin]
  /measurement-dispute/settings
  /billing/mh-credit-weights  (новые action_key)
```

## Домен и данные

### Job `measurement_disputes`

| Поле | Назначение |
|---|---|
| `id` | UUID |
| `user_id` | Владелец |
| `organization_id` | Опционально, если seat/org context понадобится по аналогии с другими модулями |
| `kind` | `analyze_response` \| `photo_analysis` |
| `status` | `queued` \| `running` \| `succeeded` \| `failed` \| `cancelled` |
| `request_payload` | Структурированный JSON (без сырого XLSX) |
| `result_payload` | Валидированный результат или null |
| `prompt_version` | Версия промпта |
| `model` | Фактическая модель OpenRouter |
| `usage_input_tokens` / `usage_output_tokens` | Контроль себестоимости |
| `quota_reserved` / `quota_units` | Ledger Мурликов |
| `error_code` / `error_message` | Безопасное сообщение для UI + код для логов |
| timestamps | created/started/finished |

Хранение полного текста ответа WB в `request_payload` допустимо для повторного прогона job, но в продуктовую аналитику и технические логи текст/размеры не пишем (только `request_id`, category, latency, status, `prompt_version`).

### Клиентское (не персистится на backend без явного job)

- Разобранный workbook, выбор листа, ручной маппинг колонок.
- Выбор обмеров, контрольный обмер продавца, дата размещения отчёта, stock/supply.
- Черновик первого обращения и чек-лист приложений.

### Парсер XLSX (manager-portal)

Реализация в `manager-portal/src/lib/measurement-dispute/` (имена уточняются в плане):

- Алиасы колонок из §4; для реального отчёта — колонки из §22 (номенклатура, номер замера, Ш/Д/В, объём, фото, статус, дата штрафа, сумма штрафа, сумма сторно и т.д.).
- Группировка: «номенклатура + номер замера»; сумма обмера = начисления + сторно; нулевой итог после сторно не в сумму к оспариванию.
- «Итого по файлу»: операции, уникальные обмеры, артикулы, начислено, сторно, чистый итог.
- Контрольные фикстуры: демо (3 / 2 / 3 400 ₽) и реальный профиль (153 / 12 / 10 / 165 804,50 / −40 401,20 / 125 403,30).
- `parseMoney` по правилам §5.

### Первое обращение

Детерминированный шаблон из подтверждённых полей (§7). Режимы: один обмер или общая претензия по чекбоксам (§22). Плейсхолдеры `[УКАЖИТЕ …]` без угадывания. Срок = дата размещения отчёта + 7 календарных дней.

## API

Базовый префикс: `/api/v1/measurement-disputes`.  
Auth: Bearer JWT. Клиентский заголовок: `X-MarketHacker-Client: manager_portal` (для extension в доке — `browser_extension`).  
Доступ: активный пользователь + feature `measurement_dispute`.

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/estimate-cost` | Расчёт Мурликов для payload (оба kind) |
| `POST` | `/` | Создать job → `202` + reserve |
| `GET` | `/` | Список jobs текущего пользователя |
| `GET` | `/active` | Активные jobs (для будущего FAB; реализовать по аналогии с card-audit) |
| `GET` | `/{id}` | Статус/метаданные |
| `GET` | `/{id}/result` | Результат при `succeeded` |
| `POST` | `/{id}/cancel` | Отмена + release, если ещё не committed |

Admin:

| Метод | Путь | Назначение |
|---|---|---|
| `GET`/`PATCH` | `/api/v1/admin/measurement-dispute/settings` | Модель, prompt_version, лимиты, rule_context, флаги kinds |

### Payload `analyze_response`

Поля из ТЗ §8 (article, source_sheet/row, dimensions, dates, stock_status, penalty, attachments flags, `wb_response_text` ≤ лимита settings, `previous_appeal_number`, `rule_context` с сервера settings если клиент не передал проверенный).

Перед вызовом модели: PII scrub (токены, телефоны, e-mail). Клиенту до create показывать, какие данные уйдут.

### Результат `analyze_response`

Строго схема §11; категории §12. При невалидном JSON — одна повторная попытка в рамках того же job с указанием нарушения схемы; иначе `failed`, UI показывает ручной шаблон.

### Payload / результат `photo_analysis`

- Только активные обмеры: есть расхождение, положительный итог, есть URL фото.
- Массовый: ≤ `max_bulk_photos` (default 50). Иначе API отклоняет bulk; одиночный анализ разрешён.
- Вердикты: `appeal` \| `borderline` \| `no_reason` \| `insufficient`.
- В result — per-item вердикты + агрегаты счётчиков/сумм для UI.
- ИИ-аргументы попадают в пользовательский черновик претензии только после `succeeded` и остаются формулировками наблюдаемых фактов (без обещания отмены штрафа).

## Мурлики и админка

### Feature

```
measurement_dispute — «Оспаривание обмеров WB»
grant_scope: user_or_org_seat
```

Включить ключ в нужные billing plans (seed/admin) при выкладке.

### Weights (каталог + UI `/billing/mh-credit-weights`)

| action_key | param_key | Смысл | Default (ориентир, уточняется при реализации) |
|---|---|---|---|
| `measurement_dispute_analyze_response` | `base` | Один job разбора ответа WB | фиксированное число ≥ 0 (если 0 — нужен явный product-sign-off; каталог сейчас требует `MIN_WEIGHT = 1`, поэтому либо default ≥ 1, либо согласованное расширение минимума) |
| `measurement_dispute_photo_analysis` | `per_measurement` | За один уникальный обмер с фото в job | тариф за обмер (§22) |

`estimate-cost` / `compute_cost`:

- text: вес `base`;
- photo: `count(unique measurements with photo) × per_measurement`.

Flow job: estimate (опционально на UI) → create → `resolve`+`reserve` → worker success `commit` / fail|cancel `release`.

### Settings (admin UI `/measurement-dispute/settings`)

- `llm_model` (OpenRouter slug) отдельно или общее поле + `photo_llm_model`, если vision-модель отличается.
- `prompt_version` (text), `photo_prompt_version`.
- `wb_response_max_chars` (default 8000).
- `max_bulk_photos` (default 50).
- `rule_context` + `rule_source_url` (проверенная выдержка; модель не выдумывает правила).
- `analyze_response_enabled` / `photo_analysis_enabled`.

Секреты только в env backend (`OPENROUTER_API_KEY` и связанные). В клиентский бандл не попадают.

## UI (manager-portal)

### Навигация

Sidebar → группа «Инструменты»: «Оспаривание обмеров», `href: /measurement-disputes`, feature `measurement_dispute`.

### Страницы

| URL | Содержание |
|---|---|
| `/measurement-disputes` | Список jobs; paywall без feature; квота/Мурлики по аналогии с card-audits |
| `/measurement-disputes/new` | Полный сценарий: upload/demo → итоги → выбор → первое обращение → блок ответа WB / фотоанализ → estimate → create job → redirect на `[id]` |
| `/measurement-disputes/[id]` | Статус, polling, результат, копирование short/detailed, warnings, ручной шаблон при fail, retry |

Состояния и копирайты — по ТЗ §13–§14 и user-facing UI rules проекта (без внутренних имён API/моделей в пользовательских сообщениях).

Аналитика продукта (§16): события через существующий механизм portal (Yandex Metrika goals или принятый в portal трекер), без текста обращений и значений XLSX. Имена событий сохранить смысловые из ТЗ, адаптировав `source=page|extension` (page = portal).

## Интеграция расширения (документация only)

Файл: `docs/integrations/measurement-disputes-extension.md`.

Содержание:

1. Назначение и границы (UI расширения делает другой разработчик).
2. Auth: JWT, refresh, `X-MarketHacker-Client: browser_extension`.
3. Base URL и полный контракт endpoints + схемы request/response.
4. UX-состояния и копирайты (что показать пользователю).
5. Локальный XLSX: файл не отправлять; парсер можно портировать или переиспользовать логику по спецификации полей.
6. Модалка: размеры, закрытие, scroll lock — требования ТЗ §2 как чеклист, без кода в этом репо.
7. Биллинг: estimate → create; обработка нехватки Мурликов и отсутствия feature.
8. Аналитические события.
9. Чеклист приёмки для E2E модалки (выполняет разработчик расширения).

## Ошибки и безопасность

- Таймаут/ошибка LLM → job `failed`, form state на `/new` не теряется до ухода со страницы; на `[id]` — retry через новый job или recreate.
- Нет `OPENROUTER_API_KEY` / выключенный kind → понятное сообщение пользователю; секрет не в сборке.
- PII scrub до модели.
- HTTPS only через backend MarketHacker.
- Не логировать полный WB response и размеры в analytics/tech logs.

## Тесты и приёмка

Покрыть §19 ТЗ в пределах portal+backend. Контрольные цифры §5 и §22. Критерии §18 и §22, с заменой «страница + модалка расширения» на «страница TEAM + дока для модалки».

Перед production: сверка `rule_context` с актуальной офертой/инструкцией WB (§21).

## Соответствие путям ТЗ → репозиторий

| В ТЗ | В реализации |
|---|---|
| `src/routes/wb-measurement-dispute.tsx` | `manager-portal/src/app/(manager)/measurement-disputes/...` |
| `src/lib/wb-measurement-report.ts` | `manager-portal/src/lib/measurement-dispute/...` |
| `src/lib/wb-photo-analysis.ts` | клиентские типы + вызовы API; vision на backend |
| `BurgerNav` / модалка расширения | out of scope кода; только docs |
| `POST /api/wb-measurement-dispute/analyze-response` | `POST /api/v1/measurement-disputes` с `kind=analyze_response` |
| `OPENAI_API_KEY` / `AI_MODEL` | OpenRouter + admin settings |

## Открытые точки на этап плана (не блокируют дизайн)

1. Точные default weights (числа) — зафиксировать в implementation plan после product/ориентира себестоимости §22.
2. Нужен ли `MIN_WEIGHT = 1` exception для бесплатного text-анализа — решить в плане биллинга; до решения text-анализ тарифицируется `base ≥ 1`.
3. Подключать ли `[id]` к существующему FAB активных AI-jobs в portal в первой поставке — да, если трудозатраты сопоставимы с card-audit hook; иначе list+detail polling достаточно для MVP поставки внутри того же scope.
