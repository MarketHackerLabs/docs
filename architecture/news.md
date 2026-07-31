# Новости платформы

Единый редакционный feed для клиентских приложений
(Manager Portal в v1; другие клиенты — через поле `clients` без смены схемы).

Не путать с:

- **продуктовыми промо** (`product_promotions`) — CTA-баннеры / upsell;
- **объявлениями поддержки** (`support_announcements`) — строка в шапке чата;
- **режимом обслуживания** (`platform_settings.maintenance`).

## Модель

Таблицы: `news_items`, `news_item_reads` (миграции `20260731_0042`, `20260731_0043`).

| Поле | Назначение |
|------|------------|
| `title` / `summary` / `body` | Заголовок, краткое описание, Markdown-текст |
| `type` | `update` \| `maintenance` \| `warning` \| `announcement` \| `promo` \| `news` |
| `status` | `draft` \| `scheduled` \| `published` \| `archived` |
| `is_pinned` / `priority` | Закрепление и порядок |
| `publish_at` / `expires_at` | Окно показа (query-time) |
| `clients` | JSONB-список ключей клиентов (`manager_portal`, …) |
| `min_app_version` | Semver floor (для будущих клиентов) |
| `cover_image_url` | URL обложки (без upload в v1) |
| `tags` / `slug` / `extra` | Метки, стабильный ключ, запас под правила |
| `author_id` | Кто создал (админ) |

Прочтение: уникальная пара `(user_id, news_item_id)` в `news_item_reads`.

## Жизненный цикл

```
draft ──publish──► published
  │                    │
  └──schedule──► scheduled ──(publish_at ≤ now, query-time)──► видно клиенту
                     │
published / * ──unpublish──► draft
любой (кроме уже archived) ──archive──► archived
```

- **Создание** — обычно `draft`.
- **Публикация** — `POST …/publish`: `status=published`, `publish_at=now` если пусто или в будущем.
- **Планирование** — `POST …/schedule` с `publishAt` в будущем → `scheduled`.
- **Снятие** — `POST …/unpublish` → `draft`.
- **Архив** — `DELETE /admin/news/{id}` (soft): `archived`. Hard delete нет.
- **Восстановление** — `POST …/restore`: только из `archived` → `draft`.

Клиент видит запись, если:

1. `status ∈ {published, scheduled}`;
2. `publish_at ≤ now`;
3. `expires_at` пуст или `> now`;
4. запрошенный `client` ∈ `clients`;
5. пройден `min_app_version` (если задан).

Черновики и архив клиенту не отдаются.

Targeting по `audience` / billing-фичам у новостей нет (в отличие от продуктовых промо).

## Права

| Действие | Кто |
|----------|-----|
| Admin CRUD / publish / schedule / preview | Superuser **или** staff с `news:manage` |
| Клиентский list / detail / read | Любой аутентифицированный пользователь |

Permission `news:manage` входит в каталог staff grants (`SUPPORT_PERMISSIONS`) и выдаётся на странице «Сотрудники».

## API

### Клиент

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/api/v1/news` | Список (без полного `body`): pagination, `type`, `pinnedOnly`, `unreadOnly` |
| GET | `/api/v1/news/{id}` | Деталь + **авто mark-read** |
| GET | `/api/v1/news/unread-count` | `{ count }` |
| POST | `/api/v1/news/{id}/read` | Явная отметка |
| POST | `/api/v1/news/read-all` | Все видимые для client |

Query: `client` (по умолчанию `manager_portal`), `orgId`, `page`, `pageSize`, `clientVersion`.

Сортировка: `is_pinned DESC`, `priority DESC`, `publish_at DESC`.

### Admin

| Метод | Путь | Описание |
|-------|------|----------|
| GET/POST | `/api/v1/admin/news` | Список / создание |
| GET/PATCH | `/api/v1/admin/news/{id}` | Детали / обновление |
| GET | `/api/v1/admin/news/{id}/preview` | Превью (в т.ч. черновик) |
| POST | `/api/v1/admin/news/{id}/publish` | Опубликовать |
| POST | `/api/v1/admin/news/{id}/unpublish` | Снять |
| POST | `/api/v1/admin/news/{id}/schedule` | Запланировать |
| POST | `/api/v1/admin/news/{id}/restore` | Вернуть из архива в черновик |
| DELETE | `/api/v1/admin/news/{id}` | Архивировать |

## Клиенты

| Клиент | Роль |
|--------|------|
| **Admin Panel** | `/news` — CRUD, Markdown-редактор, превью |
| **Manager Portal** | `/news`, `/news/[id]`, блок на дашборде, badge непрочитанных в сайдбаре, всплывающие уведомления о непрочитанных (dismiss в `localStorage`) |
| Extension / лендинг | Вне scope v1 |

## Производительность

- Список без `body`; полный текст только в detail.
- Индексы: `(status, publish_at)`, `(is_pinned, priority)`, reads по `user_id`.
- Unread-count — по уже отфильтрованному eligible-набору + anti-join reads.
- Обложки — внешние URL, `loading="lazy"` на клиенте.

## Развитие

- Клиент `browser_extension` уже в допустимых `clients`.
- Upload обложек — отдельный media-pipeline.
- Push / email при публикации — отдельные каналы, не часть feed.
- Staff permission уже есть; при необходимости — более гранулярные права.

## Связанные документы

| Документ | Связь |
|----------|--------|
| [Продуктовые промо](./product-promotions.md) | Targeting / dismiss — другая подсистема |
| [Support Chat](./support-chat.md) | Объявления виджета |
| [Контроль доступа](./access-control.md) | Staff grants, billing-фичи |
| [Дизайн API](./api-design.md) | Общий контракт |
