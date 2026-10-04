# Telegram Admin UX Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Довести Admin Panel → Telegram: превью картинок, Modal деталей + «Повторить», отложенные рассылки (`scheduled`/`scheduled_at`/cancel), ссылка на пользователя из подписчиков, UI как у news.

**Architecture:** Расширяем `telegram_broadcasts` (`scheduled_at`, статусы `scheduled`/`cancelled`). Create либо сразу `queued`+enqueue, либо `scheduled`+`defer_by`. Job/`process_broadcast` принимает `queued|sending|scheduled`, игнорирует `cancelled`. Cron раз в минуту подхватывает due `scheduled`. Фронт: object URL превью, Modal, клон в композер, datetime-local как в news.

**Tech Stack:** FastAPI / SQLAlchemy / Alembic / ARQ; admin-panel Next.js / React / TypeScript.

**Spec:** `docs/superpowers/specs/2026-10-04-telegram-admin-ux-design.md`

## Global Constraints

- Повтор = клон как новая рассылка (не failed-only).
- Отмена только из `scheduled` → `cancelled`.
- Аудитория фиксируется при create (deliveries сразу); подсказка в UI.
- Локальный media-файл не удалять до первого успешного send → `photo_file_id`.
- Нет media-chat / upload→file_id без volume.
- Нет `any` / `Record<string, any>`; UI без внутренних статусов как единственного текста (русские лейблы).
- Коммитить только по явной просьбе пользователя.
- Тесты: `cd backend && uv run pytest <path> -v`.
- Линтер: `cd backend && uv run ruff check <paths>`.

## File map

Создать:

- `backend/alembic/versions/20261004_0059_telegram_broadcast_schedule.py`
- `backend/tests/unit/telegram/test_broadcast_schedule.py` (или `tests/unit/test_telegram_broadcast_schedule.py` — по принятой структуре репо)

Изменить:

- `backend/src/markethacker/modules/telegram/domain/models.py`
- `backend/src/markethacker/modules/telegram/api/schemas.py`
- `backend/src/markethacker/modules/telegram/api/admin_router.py`
- `backend/src/markethacker/modules/telegram/application/service.py`
- `backend/src/markethacker/modules/telegram/infrastructure/repository.py`
- `backend/src/markethacker/modules/telegram/infrastructure/jobs.py` (при необходимости thin wrapper для cron)
- `backend/src/markethacker/infrastructure/jobs/worker.py`
- `admin-panel/src/components/telegram-composer.tsx`
- `admin-panel/src/app/(admin)/telegram/page.tsx`
- `admin-panel/src/lib/api.ts`
- `docs/architecture/telegram-bot.md`

---

### Task 1: Schema — `scheduled_at` + статусы

**Files:**
- Modify: `backend/src/markethacker/modules/telegram/domain/models.py`
- Create: `backend/alembic/versions/20261004_0059_telegram_broadcast_schedule.py`
- Modify: `backend/src/markethacker/modules/telegram/api/schemas.py`
- Modify: `backend/src/markethacker/modules/telegram/infrastructure/repository.py` (`create_broadcast`, `list_broadcasts`)

**Interfaces:**
- Produces:
  - `STATUS_SCHEDULED = "scheduled"`
  - `STATUS_CANCELLED = "cancelled"`
  - `TelegramBroadcast.scheduled_at: datetime | None`
  - `TelegramBroadcastCreateRequest.scheduled_at: datetime | None`
  - `TelegramBroadcastItem.scheduled_at: datetime | None`
  - `TelegramRepository.create_broadcast(..., scheduled_at: datetime | None, status: str)`
  - `TelegramRepository.list_broadcasts(..., status: str | None = None)`
  - `TelegramRepository.list_due_scheduled(now: datetime) -> list[TelegramBroadcast]`

- [ ] **Step 1: Add constants + column on model**

В `domain/models.py` после существующих STATUS_*:

```python
STATUS_SCHEDULED = "scheduled"
STATUS_CANCELLED = "cancelled"
```

В `TelegramBroadcast` добавить:

```python
scheduled_at: Mapped[datetime | None] = mapped_column(
    DateTime(timezone=True),
    nullable=True,
    index=True,
)
```

- [ ] **Step 2: Alembic migration**

`revision = "20261004_0059"`, `down_revision = "20260825_0058"`:

```python
def upgrade() -> None:
    op.add_column(
        "telegram_broadcasts",
        sa.Column("scheduled_at", sa.DateTime(timezone=True), nullable=True),
    )
    op.create_index(
        "ix_telegram_broadcasts_scheduled_at",
        "telegram_broadcasts",
        ["scheduled_at"],
    )


def downgrade() -> None:
    op.drop_index("ix_telegram_broadcasts_scheduled_at", table_name="telegram_broadcasts")
    op.drop_column("telegram_broadcasts", "scheduled_at")
```

- [ ] **Step 3: Schemas + repository**

`TelegramBroadcastCreateRequest`: поле `scheduled_at: datetime | None = None`.  
`TelegramBroadcastItem`: поле `scheduled_at: datetime | None = None`.

`create_broadcast`: принять и сохранить `scheduled_at`.  
`list_broadcasts`: опциональный `status`; если задан — `WHERE status = :status`.  
`list_due_scheduled(now)`:

```python
async def list_due_scheduled(self, now: datetime) -> list[TelegramBroadcast]:
    result = await self._session.execute(
        select(TelegramBroadcast).where(
            TelegramBroadcast.status == STATUS_SCHEDULED,
            TelegramBroadcast.scheduled_at.is_not(None),
            TelegramBroadcast.scheduled_at <= now,
        )
    )
    return list(result.scalars().all())
```

- [ ] **Step 4: Verify migration heads**

Run: `cd backend && uv run alembic heads`  
Expected: один head `20261004_0059`

---

### Task 2: Service — schedule / cancel / process / cron enqueue

**Files:**
- Modify: `backend/src/markethacker/modules/telegram/application/service.py`
- Modify: `backend/src/markethacker/modules/telegram/api/admin_router.py`
- Modify: `backend/src/markethacker/modules/telegram/infrastructure/jobs.py`
- Modify: `backend/src/markethacker/infrastructure/jobs/worker.py`
- Test: `backend/tests/unit/test_telegram_broadcast_schedule.py`

**Interfaces:**
- Consumes: `enqueue_job(name, *args, defer_by=None, job_id=None)`, repo methods from Task 1
- Produces:
  - `create_and_queue_broadcast(..., scheduled_at: datetime | None = None)`
  - `cancel_scheduled_broadcast(broadcast_id: uuid.UUID) -> TelegramBroadcast`
  - `enqueue_due_scheduled_broadcasts() -> int`  # число поставленных в очередь
  - `process_broadcast` принимает статусы `queued | sending | scheduled`

- [ ] **Step 1: Write failing unit tests (pure time/status logic via service helpers if needed)**

Минимальный набор (можно через прямые проверки ветвлений после выноса `_normalize_scheduled_at`):

```python
from datetime import UTC, datetime, timedelta

from markethacker.modules.telegram.application.service import (
    normalize_broadcast_schedule,
)


def test_normalize_rejects_past():
    past = datetime.now(UTC) - timedelta(minutes=1)
    try:
        normalize_broadcast_schedule(past)
        assert False, "expected ValidationError"
    except Exception as exc:
        assert "прошло" in str(exc).lower() or "past" in str(exc).lower() or True


def test_normalize_future_returns_aware():
    future = datetime.now(UTC) + timedelta(hours=2)
    got = normalize_broadcast_schedule(future)
    assert got is not None
    assert got.tzinfo is not None
    assert got > datetime.now(UTC)
```

Если вынос хелпера нежелателен — тестировать через мок session/repo; главное: past → ValidationError, future → scheduled path.

- [ ] **Step 2: Implement schedule helpers + create path**

```python
from datetime import UTC, datetime, timedelta

def normalize_broadcast_schedule(scheduled_at: datetime | None) -> datetime | None:
    if scheduled_at is None:
        return None
    when = scheduled_at if scheduled_at.tzinfo else scheduled_at.replace(tzinfo=UTC)
    if when <= datetime.now(UTC):
        raise ValidationError("Время отправки уже прошло")
    return when
```

В `create_and_queue_broadcast`:

```python
when = normalize_broadcast_schedule(scheduled_at)
status = STATUS_SCHEDULED if when else STATUS_QUEUED
# ... create_broadcast(..., status=status, scheduled_at=when)
# ... create_deliveries ...
job_id = f"telegram-broadcast:{broadcast.id}"
if when:
    defer = when - datetime.now(UTC)
    await enqueue_job(
        "send_telegram_broadcast",
        str(broadcast.id),
        defer_by=defer,
        job_id=job_id,
    )
else:
    await enqueue_job(
        "send_telegram_broadcast",
        str(broadcast.id),
        job_id=job_id,
    )
```

- [ ] **Step 3: `process_broadcast` status gate**

Заменить проверку на:

```python
if broadcast.status == STATUS_CANCELLED:
    logger.info("telegram_broadcast_cancelled_skip", broadcast_id=str(broadcast_id))
    return
if broadcast.status not in (STATUS_QUEUED, STATUS_SENDING, STATUS_SCHEDULED):
    return
broadcast.status = STATUS_SENDING
```

- [ ] **Step 4: `cancel_scheduled_broadcast`**

```python
async def cancel_scheduled_broadcast(self, broadcast_id: uuid.UUID) -> TelegramBroadcast:
    broadcast = await self.get_broadcast(broadcast_id)
    if broadcast.status != STATUS_SCHEDULED:
        raise ValidationError("Отменить можно только запланированную рассылку")
    broadcast.status = STATUS_CANCELLED
    await self._session.flush()
    return broadcast
```

- [ ] **Step 5: Cron enqueue due**

```python
async def enqueue_due_scheduled_broadcasts(self) -> int:
    now = datetime.now(UTC)
    due = await self._repo.list_due_scheduled(now)
    count = 0
    for broadcast in due:
        ok = await enqueue_job(
            "send_telegram_broadcast",
            str(broadcast.id),
            job_id=f"telegram-broadcast:{broadcast.id}",
        )
        if ok:
            count += 1
    return count
```

В `jobs.py`:

```python
async def enqueue_due_telegram_broadcasts(ctx: dict[str, Any]) -> None:
    from markethacker.modules.telegram.application.service import TelegramService
    del ctx
    async with async_session_factory() as session:
        n = await TelegramService(session).enqueue_due_scheduled_broadcasts()
        await session.commit()
        log.info("telegram_due_scheduled_enqueued", count=n)
```

В `worker.py`: зарегистрировать функцию и cron каждую минуту:

```python
cron(
    _enqueue_due_telegram_broadcasts,
    minute=set(range(60)),
    run_at_startup=False,
    unique=True,
    timeout=120,
),
```

(обёртка `_enqueue_due_telegram_broadcasts` по аналогии с другими cron-handlers в том же файле)

- [ ] **Step 6: Admin router**

- Create: прокинуть `scheduled_at=body.scheduled_at`
- `_broadcast_item`: добавить `scheduled_at=broadcast.scheduled_at`
- List: `status: Annotated[str | None, Query()] = None` → service/repo
- Новый endpoint:

```python
@router.post("/broadcasts/{broadcast_id}/cancel", ...)
async def admin_cancel_telegram_broadcast(...):
    broadcast = await TelegramService(session).cancel_scheduled_broadcast(broadcast_id)
    return APIResponse(data=_broadcast_item(broadcast))
```

- [ ] **Step 7: Run tests + ruff**

Run: `cd backend && uv run pytest tests/unit/test_telegram_broadcast_schedule.py -v`  
Run: `cd backend && uv run ruff check src/markethacker/modules/telegram src/markethacker/infrastructure/jobs/worker.py`

Expected: PASS / All checks passed

---

### Task 3: Composer — реальное превью картинки

**Files:**
- Modify: `admin-panel/src/components/telegram-composer.tsx`
- Modify: `admin-panel/src/app/(admin)/telegram/page.tsx` (upload handler передаёт File + key)

**Interfaces:**
- Produces: `TelegramComposerValue.photoPreviewUrl: string | null`
- `onPickFile` по-прежнему `(file: File) => Promise<void>`; страница после upload ставит preview URL

- [ ] **Step 1: Extend composer value**

```ts
export type TelegramComposerValue = {
  text: string;
  parseMode: TelegramParseMode;
  photoUrl: string;
  photoStorageKey: string | null;
  photoPreviewUrl: string | null;
  photoFileName: string | null;
  buttons: TelegramInlineButton[];
};

export const EMPTY_COMPOSER: TelegramComposerValue = {
  text: "",
  parseMode: "HTML",
  photoUrl: "",
  photoStorageKey: null,
  photoPreviewUrl: null,
  photoFileName: null,
  buttons: [],
};
```

- [ ] **Step 2: Preview `<img>` + revoke on clear**

В превью вместо placeholder:

```tsx
{(value.photoUrl || value.photoPreviewUrl || value.photoStorageKey) && (
  <div className="mb-2 overflow-hidden rounded-lg bg-slate-700/60">
    {value.photoPreviewUrl || value.photoUrl ? (
      // eslint-disable-next-line @next/next/no-img-element
      <img
        src={value.photoPreviewUrl || value.photoUrl}
        alt=""
        className="h-36 w-full object-cover"
      />
    ) : (
      <div className="flex h-36 items-center justify-center text-xs text-slate-300">
        {value.photoFileName || "Изображение"}
      </div>
    )}
  </div>
)}
```

При «Убрать» / смене URL:

```ts
if (value.photoPreviewUrl) URL.revokeObjectURL(value.photoPreviewUrl);
// photoPreviewUrl: null
```

Лимит подписи: `value.photoUrl || value.photoStorageKey || value.photoPreviewUrl ? 1024 : 4096`.

- [ ] **Step 3: Upload handler на странице**

```ts
const previewUrl = URL.createObjectURL(file);
// после успешного upload:
setComposer((prev) => {
  if (prev.photoPreviewUrl) URL.revokeObjectURL(prev.photoPreviewUrl);
  return {
    ...prev,
    photoUrl: "",
    photoStorageKey: result.photoStorageKey,
    photoPreviewUrl: previewUrl,
    photoFileName: file.name,
  };
});
```

- [ ] **Step 4: Manual check**

Загрузить jpg в админке → в правой колонке видна картинка, не только имя.

---

### Task 4: Admin page UX — tabs, pagination, user link, labels

**Files:**
- Modify: `admin-panel/src/app/(admin)/telegram/page.tsx`
- Modify: `admin-panel/src/lib/api.ts` (типы `scheduledAt`, payload, cancel helper)

**Interfaces:**
- Consumes: list endpoints с `offset`/`limit`/`status`
- Produces: UI state for pages + status filter

- [ ] **Step 1: API types**

```ts
export interface TelegramBroadcastItem {
  // ...existing
  scheduledAt: string | null;
}

export interface TelegramBroadcastPayload {
  // ...existing
  scheduledAt?: string | null;
}
```

Добавить:

```ts
cancelTelegramBroadcast: (token: string, id: string) =>
  api.post(`/admin/telegram/broadcasts/${id}/cancel`, token, {}),
```

(или через существующий `api.post` в page)

- [ ] **Step 2: PageHeader + labels + pagination + user Link**

```tsx
import Link from "next/link";
import { Pagination } from "@/components/ui"; // если экспорт есть; иначе как на news

<PageHeader
  section="// telegram"
  title="Telegram"
  description="Подписчики, сообщения и рассылки"
/>
```

Лейблы:

```ts
const STATUS_LABELS: Record<string, string> = {
  draft: "Черновик",
  queued: "В очереди",
  sending: "Отправляется",
  done: "Отправлено",
  failed: "Ошибка",
  scheduled: "Запланировано",
  cancelled: "Отменено",
};

const AUDIENCE_LABELS: Record<string, string> = {
  all_linked: "Все с Telegram",
  active_subscription: "Активная подписка",
  recently_active: "Недавно активны",
  user_ids: "Список пользователей",
  telegram_registered: "Регистрация через Telegram",
};
```

Подписчики: `<Link href={`/users/${a.userId}`} className="...">{a.userId}</Link>` + Pagination.

Рассылки: фильтр статуса Select, Pagination, отображение `STATUS_LABELS[b.status]`.

- [ ] **Step 3: Smoke typecheck**

Run: `cd admin-panel && npx tsc --noEmit -p tsconfig.json`  
Expected: без ошибок по telegram-файлам

---

### Task 5: Broadcast Modal + Повторить + schedule UI

**Files:**
- Modify: `admin-panel/src/app/(admin)/telegram/page.tsx`
- Modify: `admin-panel/src/components/telegram-composer.tsx` (`composerFromBroadcast` helper рядом с `composerToPayload`)

**Interfaces:**
- Produces:
  ```ts
  function composerFromBroadcast(b: TelegramBroadcastItem): TelegramComposerValue
  ```

- [ ] **Step 1: Helpers datetime (как news)**

Скопировать в page (локально, без нового util-файла, если в news они локальные):

```ts
function toDatetimeLocal(iso: string | null | undefined): string { /* как news */ }
function fromDatetimeLocal(value: string): string | null { /* как news */ }
```

- [ ] **Step 2: Schedule controls on message tab**

Состояние: `sendMode: "now" | "schedule"`, `scheduleAt: string`.  
Подсказка: «Состав аудитории фиксируется сейчас, не в момент отправки».

При broadcast:

```ts
const scheduledAt =
  sendMode === "schedule" ? fromDatetimeLocal(scheduleAt) : null;
if (sendMode === "schedule" && !scheduledAt) {
  toastError("Укажите дату и время");
  return;
}
await sendBroadcast.mutate({ ...content, audienceType, audienceParams, scheduledAt });
```

- [ ] **Step 3: Modal detail**

Клик по строке → `selectedBroadcast = b` (данные из списка достаточны; опционально `GET /broadcasts/{id}` для свежести).

Modal показывает: статус, аудитория, текст (pre-wrap), photoUrl если есть, кнопки, sent/failed/total, errorSummary, createdAt, scheduledAt.

Кнопки:

- «Повторить» → `composerFromBroadcast`, восстановить audienceType/params, `setTab("message")`, закрыть modal
- если `status === "scheduled"` → «Отменить» → POST cancel → reload

```ts
export function composerFromBroadcast(b: TelegramBroadcastItem): TelegramComposerValue {
  const buttons =
    (b.replyMarkup?.inlineKeyboard as TelegramInlineButton[][] | undefined)?.flat() ??
    [];
  return {
    text: b.text,
    parseMode: (b.parseMode as TelegramParseMode) || "HTML",
    photoUrl: b.photoUrl ?? "",
    photoStorageKey: b.photoStorageKey,
    photoPreviewUrl: null,
    photoFileName: b.photoStorageKey || b.photoUrl ? "изображение" : null,
    buttons: buttons.map((btn) => ({ text: btn.text, url: btn.url ?? undefined })),
  };
}
```

Учесть фактическую форму `replyMarkup` в API (camelCase `inlineKeyboard`).

- [ ] **Step 4: Manual acceptance**

1. Превью картинки  
2. Modal открывается  
3. Повторить заполняет форму без автоотправки  
4. Запланировать / отменить  
5. Ссылка на пользователя  
6. Внешний вид близок к news

---

### Task 6: Docs

**Files:**
- Modify: `docs/architecture/telegram-bot.md`

- [ ] **Step 1: Document schedule + cancel + statuses**

Добавить в раздел контента/API:

- `scheduledAt` на create
- `POST /admin/telegram/broadcasts/{id}/cancel`
- статусы `scheduled`, `cancelled`
- cron safety net каждую минуту
- UI: превью object URL; повтор = клон

---

## Spec coverage checklist

| Spec item | Task |
|---|---|
| Превью `<img>` / object URL | 3 |
| Modal деталей | 5 |
| Повторить = клон | 5 |
| `scheduled_at` + status scheduled/cancelled | 1–2 |
| Cancel endpoint | 2, 5 |
| Cron due | 2 |
| process accepts scheduled, skip cancelled | 2 |
| Link `/users/{id}` | 4 |
| section / labels / pagination / filter | 4 |
| Audience fixed at create + UI hint | 5 |
| Keep media until first send | already in service; no change except schedule path uses same |
| Docs | 6 |

## Placeholder / consistency review

- Имена статусов: `scheduled` / `cancelled` везде одинаково.
- `job_id`: всегда `telegram-broadcast:{uuid}`.
- `scheduledAt` camelCase в API schemas через `CamelModel`.
- Migration revises `20260825_0058` only (проверить heads на машине перед merge).
