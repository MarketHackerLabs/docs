# Telegram admin: превью, детали рассылки, повтор, отложенные, UX

Дата: 2026-10-04  
Статус: approved for planning  

## Цель

Довести раздел Admin Panel → Telegram до уровня остальных админ-страниц: нормальное превью медиа, просмотр отправленной рассылки, повтор (клон), отложенная отправка по образцу новостей, переход к пользователю из подписчиков.

## Решения (зафиксировано)

| Тема | Решение |
|---|---|
| Повтор рассылки | Клон как **новая** рассылка (не досылка failed) |
| Отложенная отправка | Как новости: статус `scheduled` + `scheduled_at`, отмена до старта |
| Просмотр рассылки | Modal + кнопка «Повторить» |
| Служебный чат для медиа | Не используем |
| Хранение медиа | Общий Docker volume api+worker; после первого `file_id` локальный файл удаляется |

## Scope

### In

1. Превью изображения в композере (реальный `<img>`, не только имя файла).
2. Modal деталей рассылки (статус, аудитория, текст, кнопки, статистика доставки, ошибки, даты).
3. «Повторить» → заполнить композер + перейти на вкладку создания.
4. Отложенные рассылки: `scheduled_at`, статус `scheduled`, UI datetime, отмена.
5. Ссылка с подписчика на `/users/{id}`.
6. Выравнивание UI страницы под паттерн news/billing (`section`, русские лейблы, pagination, фильтр статуса).

### Out

- Досылка только failed-доставок.
- Отдельный маршрут `/telegram/broadcasts/[id]`.
- Служебный Telegram-чат / обязательный `file_id` на upload.
- Preview уже отправленных фото через Telegram `getFile` (в деталях — текст/метаданные; картинка, если есть `photoUrl`).

## Backend

### Модель `telegram_broadcasts`

Добавить:

- `scheduled_at: datetime | None` (timestamptz)
- статус `scheduled` (и при отмене — `cancelled` либо возврат в `draft`; **рекомендация: `cancelled`**, чтобы не путать с черновиком композера)

Существующие: `draft`, `queued`, `sending`, `done`, `failed`.

Миграция Alembic: колонка + без ломки существующих строк (`scheduled_at` null).

### Create broadcast

`POST /admin/telegram/broadcasts`:

- опциональный `scheduledAt` (ISO datetime, timezone-aware; naive → UTC);
- если `scheduledAt` в будущем → статус `scheduled`, создаются deliveries сразу (аудитория фиксируется на момент создания), job через `enqueue_job(..., defer_by=…)`, стабильный `job_id` вида `telegram-broadcast:{id}`;
- если `scheduledAt` отсутствует или ≤ now → как сейчас: `queued` + немедленный enqueue;
- если `scheduledAt` в прошлом → ValidationError («время уже прошло»).

### Cancel scheduled

`POST /admin/telegram/broadcasts/{id}/cancel`:

- только из `scheduled`;
- статус → `cancelled`;
- worker при старте job проверяет статус: если не `queued`/`sending`/`scheduled` (ожидающий) — no-op;
- для due `scheduled` job переводит в `sending` как сейчас из `queued`.

Уточнение жизненного цикла job:

1. Create scheduled → status `scheduled`, deferred job.
2. Job стартует → если status != `scheduled`, exit; иначе → `sending` → send.
3. Cancel → `cancelled`; deferred job при срабатывании увидит `cancelled` и выйдет.

### Cron safety net

Cron ~каждую минуту: выбрать `status=scheduled AND scheduled_at <= now()`, для каждой:

- если ещё не в работе — поставить `queued` и `enqueue_job` с тем же `job_id` (идемпотентность ARQ job_id), либо вызвать тот же `process_broadcast` path.

Цель: переживать рестарт Redis/воркера без потерянных отложенных.

### Get broadcast

Уже есть `GET /admin/telegram/broadcasts/{id}`. Расширить ответ:

- `scheduledAt`
- русские/сырые статусы как есть (лейблы на фронте)
- при необходимости краткий `replyMarkup` (уже есть)

Список deliveries в v1 **не** отдаём (достаточно sent/failed/total + errorSummary).

### Accounts

`GET /admin/telegram/accounts` без обязательного обогащения email (можно оставить `userId`). Фронт делает `Link` на `/users/{id}`. Опционально позже: `userEmail` / `userFullName` — **out of scope**, если не дёшево в том же PR.

## Frontend (admin-panel)

### Composer preview

- При upload файла: `photoPreviewUrl = URL.createObjectURL(file)` + `photoStorageKey` с API.
- В превью: `<img src={photoPreviewUrl || photoUrl}>` с object-fit.
- При смене/удалении фото: `revokeObjectURL`.
- Подпись лимита 1024 при наличии превью/URL/storage key.

### Страница `/telegram`

- `PageHeader` с `section="// telegram"`.
- Вкладки: Подписчики / Сообщение / Рассылки (визуально ближе к остальным страницам; не «голые» primary без контекста).
- Подписчики: `Link` на пользователя, Pagination (`offset`/`limit`/`total`).
- Рассылки: человекочитаемые статусы и аудитории, Pagination, фильтр по статусу, клик → Modal.
- Сообщение: композер + direct send + блок аудитории + выбор «Сейчас» / «Запланировать» (datetime-local, МСК в UI как у новостей — те же хелперы `toDatetimeLocal` / `fromDatetimeLocal`, если уже есть).

### Modal рассылки

- Контент + метаданные + «Повторить» + для `scheduled` — «Отменить».
- «Повторить»: `composerToPayload` наоборот → `setComposer`, `setAudience*`, `setTab("message")`, закрыть Modal. Не создавать рассылку автоматически.

## Риски и краевые случаи

- Аудитория при schedule фиксируется при create (доставки создаются сразу) — поведение явно: состав не «пересчитывается» в момент send. Зафиксировать в UI подсказкой.
- Локальный файл на volume должен жить до фактической отправки: **не** удалять при create scheduled, только после первого успешного send → `photo_file_id` (уже есть логика).
- Cancel после того, как job уже взял задачу (`sending`) — cancel отклоняется.
- Timezone: хранение UTC; UI как у новостей.

## Критерии приёмки

1. Загрузка фото в композере показывает картинку в превью.
2. Клик по рассылке открывает Modal с содержимым и статистикой.
3. «Повторить» заполняет форму новой рассылки без немедленной отправки.
4. Можно запланировать на будущее, увидеть статус «Запланировано», отменить до старта; после `scheduled_at` уходит в отправку.
5. Из «Подписчики» переход на карточку пользователя.
6. Страница визуально и по паттернам близка к news (section, лейблы, pagination, modal).

## Не делать в этом изменении

- Resend failed-only.
- Media chat / upload→file_id без volume.
- Переписывание support/notify ботов.
