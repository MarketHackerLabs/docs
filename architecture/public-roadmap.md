# Публичный роадмап продукта

Единый CMS-управляемый план развития для лендинга и других клиентов.
Backend — единственный источник данных. Настройки показа живут в модуле
`roadmap` (`roadmap_settings`), **не** в `platform_settings`.

Не путать с:

- **новостями** (`news`) — редакционный feed авторизованных клиентов;
- **продуктовыми промо** (`product_promotions`) — CTA-баннеры;
- **объявлениями поддержки** (`support_announcements`);
- **платформенной дорожной картой разработки** (`docs/architecture/roadmap.md`, если появится).

## Модель

Таблицы (миграция `20260803_0044`):

| Таблица | Назначение |
|--------|------------|
| `roadmap_settings` | Singleton (`id=1`): `is_public_enabled` |
| `roadmap_periods` | Периоды таймлайна (`month` \| `quarter` \| `custom`) |
| `roadmap_categories` | Категории карточек (seed + CRUD) |
| `roadmap_progress_statuses` | Этапы разработки (seed + CRUD) |
| `roadmap_items` | Карточки |

### Две оси статусов

| Ось | Поле | Значения |
|-----|------|----------|
| Публикация | `visibility` | `draft` \| `scheduled` \| `published` \| `archived` |
| Разработка | `progress_status_id` → справочник | seed: research, planned, in_progress, testing, released, cancelled |

Фаза периода (`past` \| `current` \| `future`) **вычисляется** по `starts_at` / `ends_at`, не хранится.

### Карточка

| Поле | Назначение |
|------|------------|
| `title` / `summary` / `body` | Заголовок, краткое и подробное описание |
| `period_id` | Период колонки |
| `category_id` | Категория |
| `progress_status_id` | Этап разработки |
| `visibility` | Черновик / расписание / опубликовано / архив |
| `sort_order` | Порядок в периоде |
| `publish_at` / `published_at` | Окно и факт публикации |
| `links` / `media` | JSONB-списки для детали |
| `slug` | Опциональный стабильный ключ |

Контент при миграции: справочники в `0044`; пример май–октябрь 2026 в `0045`/`0046`
(вставка по slug периода, без пропуска всей миграции из‑за чужих ручных записей).
`0046` досеивает базы, где `0045` уже noop-нулась.

## Жизненный цикл карточки

```
draft ──publish──► published
  │
  └──schedule──► scheduled ──(publish_at ≤ now)──► видно публично
published ──unpublish──► draft
* ──archive──► archived ──restore──► draft
```

Публично видно, если:

1. `roadmap_settings.is_public_enabled = true`;
2. `visibility ∈ {published, scheduled}`;
3. `publish_at ≤ now`;
4. период `is_visible`;
5. категория и progress-статус `is_active`.

## Права

| Действие | Кто |
|----------|-----|
| Admin CRUD / publish / settings | Superuser **или** staff с `roadmap:manage` |
| Публичный snapshot / detail | Без аутентификации |

Permission входит в `SUPPORT_PERMISSIONS` и выдаётся на странице «Сотрудники».

## API

### Публичный

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/api/v1/roadmap` | Снимок: `enabled`, справочники, периоды с карточками |
| GET | `/api/v1/roadmap/items/{id}` | Деталь (только если публично видима) |

Query снимка: `category`, `progressStatus` (repeatable), `limitPeriods` (по умолчанию 5).

При `enabled=false` ответ: пустые списки, без утечки черновиков.

### Admin

Префикс `/api/v1/admin/roadmap`, право `roadmap:manage`.

| Ресурс | Операции |
|--------|----------|
| `/settings` | GET / PATCH (`isPublicEnabled`) |
| `/periods` | list / create / patch / delete / reorder |
| `/categories` | list / create / patch / delete |
| `/progress-statuses` | list / create / patch / delete |
| `/items` | list / create / get / preview / patch / publish / unpublish / schedule / archive / restore / reorder |

## Клиенты

| Клиент | Роль |
|--------|------|
| **Admin Panel** | `/roadmap`, `/roadmap/periods`, `/roadmap/catalog` |
| **Лендинг** (`market-navigators`) | Секция `#roadmap`; скрывается если `!enabled` или нет опубликованных периодов с карточками |
| Manager Portal / Extension | Могут использовать тот же публичный контракт |

Лендинг: `VITE_API_URL` (dev — `.env`; prod Docker — build-arg, по умолчанию `https://api.markethacker.ru/api/v1`).

## Производительность

- Один round-trip на публичный снимок; `body` только в detail.
- Индексы: `(visibility, publish_at)`, `(period_id, sort_order)`, даты периодов.
- Redis-кэш не обязателен в v1; при росте — ключ по snapshot + инвалидация на admin-мутациях.

## Развитие

- Связь карточек с новостями.
- Upload media через общий media-pipeline.
- HTTP Cache-Control / CDN для публичного GET.
- Клиентские фильтры уже на снимке; серверные query — для других клиентов.

## Связанные документы

- [Новости](./news.md)
- [Дизайн API](./api-design.md)
- [Модель данных](./data-model.md)
