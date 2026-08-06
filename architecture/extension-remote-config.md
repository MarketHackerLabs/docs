# Remote config расширения (DOM-селекторы)

Удалённая конфигурация Chromium-расширения: CSS-селекторы и параметры
монтирования виджетов на страницах маркетплейсов. Публикуется из Admin Panel
(**Расширение → DOM-селекторы**), читается расширением без auth.

Не путать с:

- **bootstrap** `GET /extension/config` — статичные URL API, min version, poll
  entitlements (см. [Дизайн API](./api-design.md#extension-browser),
  [Аутентификация](./authentication.md));
- **entitlements** `GET /extension/entitlements` — фичи тарифа и capabilities;
- **продуктовыми промо** (`/promotions/*`) — баннеры / CTA.

Таблицы: `extension_config_revisions`, `extension_config_active`
(миграция `20260708_0024`).

## Namespaces

Конфиг разбит по namespace. Сейчас зарегистрирован один:

| Namespace | `schemaVersion` | Назначение |
|-----------|:---------------:|------------|
| `dom_selectors` | `1` | CSS-селекторы и mount-параметры виджетов |

Новые namespace добавляются в каталог на бэкенде без смены URL-схемы
(`GET /extension/config/{namespace}`).

Константы (сервер):

| Параметр | Значение | Назначение |
|----------|----------|------------|
| `cacheTtlSeconds` | `300` | Рекомендуемый интервал poll / HTTP `max-age` |
| `selectorMissRefetchThreshold` | `3` | После N промахов селектора — форс-рефетч без кэша |
| Max document size | 256 KiB | Лимит JSON `content` |
| Max leaf entries | 500 | Лимит виджетов в одном документе |

## Публичный API (клиент расширения)

Auth не требуется. Rate limit namespace-эндпоинта: `60/minute`.

Обёртка ответа: `{ "data": T, "meta": {} }`. Поля в **camelCase**.

### Bootstrap (не remote config)

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/extension/config` | — | `apiBaseUrl`, `minExtensionVersion`, `entitlementsPollIntervalSeconds`, `capabilitiesVersion` |

В bootstrap **нет** `cacheTtlSeconds` / `selectorMissRefetchThreshold` — они только
в ответе namespace.

### Опубликованный namespace

| Метод | Путь | Auth | Описание |
|-------|------|:----:|----------|
| GET | `/api/v1/extension/config/{namespace}` | — | Активная ревизия namespace |

**Path:**

| Параметр | Тип | Обязательный | Описание |
|----------|-----|:------------:|----------|
| `namespace` | string | да | Ключ из каталога. Для селекторов: `dom_selectors`. Неизвестный → `404` |

**Заголовки запроса (опционально):**

| Заголовок | Пример | Назначение |
|-----------|--------|------------|
| `If-None-Match` | `"abc123…"` | ETag прошлой версии; при совпадении — `304` без тела |

**Заголовки ответа (`200`):**

| Заголовок | Пример | Назначение |
|-----------|--------|------------|
| `ETag` | `"abc123…"` | = `data.contentHash` (с кавычками) |
| `Cache-Control` | `public, max-age=300` | HTTP-кэш; клиенту всё равно стоит хранить локально |

**Тело `data` (`200`):**

| Поле | Тип | Описание |
|------|-----|----------|
| `namespace` | string | Ключ namespace (`dom_selectors`) |
| `version` | number | Монотонный номер ревизии |
| `contentHash` | string | Хэш содержимого; сохранять вместе с конфигом |
| `schemaVersion` | number | Версия схемы leaf-структуры |
| `publishedAt` | string \| null | ISO datetime публикации |
| `cacheTtlSeconds` | number | Интервал poll (сейчас `300`) |
| `selectorMissRefetchThreshold` | number | Порог промахов для форс-рефетча (сейчас `3`) |
| `content` | object | Дерево селекторов (см. ниже) |

**Коды ответа:**

| Код | Когда |
|-----|--------|
| `200` | Полный JSON + новый `ETag` |
| `304` | `If-None-Match` совпал с текущим `contentHash`; тело пустое |
| `404` | Неизвестный namespace или нет активной ревизии |
| `429` | Rate limit |

#### Пример запроса

```http
GET /api/v1/extension/config/dom_selectors HTTP/1.1
If-None-Match: "abc123def456..."
```

#### Пример ответа `200` (фрагмент `data`)

```json
{
  "namespace": "dom_selectors",
  "version": 1,
  "contentHash": "abc123def456...",
  "schemaVersion": 1,
  "publishedAt": "2026-07-08T12:00:00Z",
  "cacheTtlSeconds": 300,
  "selectorMissRefetchThreshold": 3,
  "content": {
    "wb": {
      "client": {
        "catalog-cards": {
          "selector": [
            ".product-card-list .product-card",
            ".cards-list__container .product-card"
          ],
          "query": {
            "all": true,
            "isInfinity": true
          }
        }
      }
    }
  }
}
```

Поле `content` — это дерево виджетов. Полный seed по умолчанию включает
`wb.client.*` и `wb.seller.*` (см. [Seed-ключи](#seed-ключи-dom_selectors)).

## Схема `content` (`dom_selectors`)

Дерево ровно из **трёх уровней**:

```
{marketplace} → {cabinet} → {widgetKey} → SelectorEntry
```

| Уровень | Пример ключа | Правило ключа |
|---------|--------------|---------------|
| marketplace | `wb` | `^[a-zA-Z0-9_-]{1,64}$` |
| cabinet | `client`, `seller` | то же |
| widget | `catalog-cards` | то же |

Набор маркетплейсов, кабинетов и виджетов **не ограничен API**: новые ключи
добавляются публикацией конфига без смены эндпоинта. Клиент должен читать
виджет по стабильному `widgetKey`, а не хардкодить CSS.

### `SelectorEntry`

Допустимы три формы на входе валидации; в ответе обычно нормализованный объект.

| Форма | Результат |
|-------|-----------|
| строка `"css"` | `{ "selector": "css" }` |
| массив `["a", "b"]` | `{ "selector": ["a", "b"] }` — fallback по порядку |
| объект | поля ниже |

| Поле | Тип | Обязательный | По умолчанию | Описание |
|------|-----|:------------:|--------------|----------|
| `selector` | string \| string[] | да | — | CSS-селектор или до 10 fallback по порядку |
| `fallback` | string[] | нет | — | Доп. fallback (1–10), отдельно от массива в `selector` |
| `query` | object | нет | — | Параметры поиска |
| `query.all` | boolean | нет | `false` | `true` — `querySelectorAll`; `false` — один якорь |
| `query.isInfinity` | boolean | нет | `false` | `true` — перепроверять при изменениях DOM (SPA); `false` — статичный элемент |
| `mount` | object | нет | — | Куда вставить контейнер виджета |
| `mount.position` | `"before"` \| `"after"` \| `"append"` | нет | — | `after` — под якорем; `before` — над блоком; `append` — внутрь |
| `scope` | `"document"` \| `"host"` | нет | `document` | Корень поиска: документ или host-элемент (карточка) |

Ограничения селектора: длина ≤ 512; whitelist CSS-символов; запрещены фрагменты
`javascript:`, `expression(`, `url(`, `<script`, `` ` ``. Неизвестные поля leaf —
ошибка валидации при publish.

### Связь с хуками расширения

| Поле конфига | Хук / поведение |
|--------------|-----------------|
| `selector` (+ массив / `fallback`) | `useQuerySelector(selector \| [...], options)` |
| `query.all`, `query.isInfinity` | опции `useQuerySelector` |
| `scope: "host"` | `root` = элемент карточки/хоста, не `document` |
| `mount.position` | `useMountContainer(host, { position })` |

`isInfinity: true` нужен для SPA маркетплейса, когда DOM догружается после
навигации. `all: true` — списки карточек; `all: false` — один хеадер/контейнер.

## Как обновлять конфиг в клиенте

Эндпоинт поддерживает `ETag` / `If-None-Match`.

Рекомендуемая логика:

1. При запуске расширения загрузить
   `GET /api/v1/extension/config/dom_selectors` и сохранить локально
   (`content` + `contentHash` / ETag + `version`).
2. Раз в `cacheTtlSeconds` (сейчас 300 с) повторять запрос с
   `If-None-Match: "<сохранённый contentHash>"`.
3. Если ничего не изменилось — ответ `304`, тело пустое (экономия трафика);
   локальный кэш не трогать.
4. Если селектор несколько раз подряд не нашёл элемент (после
   `selectorMissRefetchThreshold`, сейчас 3 промаха) — принудительно
   перезапросить конфиг **без** `If-None-Match` (обход локального/HTTP кэша).
5. Если пришла новая версия (`200` + новый `contentHash`) — обновить
   локальный кэш и заново запустить поиск всех mount-точек.

### ETag

Бэкенд отдаёт:

```http
ETag: "abc123def456..."
Cache-Control: public, max-age=300
```

Клиент сохраняет этот hash вместе с конфигом (значение = `data.contentHash`).

### If-None-Match

При следующем запросе клиент сообщает, что у него уже есть запись с таким хэшем:

```http
GET /api/v1/extension/config/dom_selectors
If-None-Match: "abc123def456..."
```

Сервер сравнивает хэш:

- совпал → `304 Not Modified`, тело пустое;
- не совпал → `200 OK` + полный JSON + новый `ETag`.

### Пример клиента (TypeScript)

```typescript
interface ExtensionConfigNamespace {
  namespace: string;
  version: number;
  contentHash: string;
  schemaVersion: number;
  publishedAt: string | null;
  cacheTtlSeconds: number;
  selectorMissRefetchThreshold: number;
  content: Record<
    string,
    Record<string, Record<string, SelectorEntry>>
  >;
}

interface SelectorEntry {
  selector: string | string[];
  fallback?: string[];
  scope?: "document" | "host";
  query?: { all?: boolean; isInfinity?: boolean };
  mount?: { position?: "before" | "after" | "append" };
}

async function fetchDomSelectors(
  apiBase: string,
  etag?: string,
): Promise<"not-modified" | ExtensionConfigNamespace> {
  const response = await fetch(
    `${apiBase}/api/v1/extension/config/dom_selectors`,
    {
      headers: etag ? { "If-None-Match": `"${etag}"` } : undefined,
    },
  );

  if (response.status === 304) {
    return "not-modified";
  }

  if (!response.ok) {
    throw new Error(`dom_selectors config failed: ${response.status}`);
  }

  const body = (await response.json()) as { data: ExtensionConfigNamespace };
  return body.data;
}
```

## Seed-ключи (`dom_selectors`)

Начальная ревизия (миграция / defaults в коде). Клиент может опираться на эти
ключи; CSS внутри них меняется из админки без релиза расширения.

| Путь | Назначение |
|------|------------|
| `wb.client.tariff-banner-anchor` | Якорь под хеадером клиентского WB (`header.header`, mount `after`) |
| `wb.client.catalog-page-anchor` | Контейнер страницы каталога |
| `wb.client.catalog-cards` | Список карточек товара (`all` + `isInfinity`, массив fallback) |
| `wb.client.card-image-wrap` | Обёртка изображения внутри карточки (`scope: host`) |
| `wb.client.card-price-wrap` | Блок цены внутри карточки (`scope: host`, mount `after`) |
| `wb.seller.funnel-page-anchor` | Якорь страницы воронки в кабинете продавца |
| `wb.seller.lost-revenue-header` | Хеадер seller (`div[data-testid="header-view"]`) |

## Admin API

Только superuser. UI: **Расширение → DOM-селекторы** (`/extension/dom-selectors`).

| Метод | Путь | Описание |
|-------|------|----------|
| GET | `/api/v1/admin/extension-config` | Список namespace |
| GET | `/api/v1/admin/extension-config/{namespace}` | Активная ревизия + defaults + meta |
| GET | `/api/v1/admin/extension-config/{namespace}/revisions` | История (`page`, `page_size`) |
| GET | `/api/v1/admin/extension-config/{namespace}/revisions/{version}` | Деталь ревизии |
| GET | `/api/v1/admin/extension-config/{namespace}/export` | Экспорт активной (как detail) |
| POST | `/api/v1/admin/extension-config/{namespace}/validate` | Валидация без публикации |
| POST | `/api/v1/admin/extension-config/{namespace}/publish` | Новая ревизия → active |
| POST | `/api/v1/admin/extension-config/{namespace}/import` | Импорт = publish |
| POST | `/api/v1/admin/extension-config/{namespace}/rollback` | Откат к `targetVersion` (новая ревизия с тем же content) |

**Тело publish / validate / import:**

| Поле | Тип | Обязательный | Описание |
|------|-----|:------------:|----------|
| `content` | object | да | Полное дерево namespace |
| `comment` | string \| null | нет | Комментарий ревизии, max 500 |

**Тело rollback:**

| Поле | Тип | Обязательный | Описание |
|------|-----|:------------:|----------|
| `targetVersion` | number (≥ 1) | да | Версия, к которой откатываемся |
| `comment` | string \| null | нет | По умолчанию `rollback to v{N}` |

После publish/rollback сервер коммитит транзакцию и инвалидирует Redis-кэш
namespace (`extension:config:active`).

## Хранение и кэш (бэкенд)

| Сущность | Назначение |
|----------|------------|
| `extension_config_revisions` | Иммутабельные ревизии (`namespace` + `version`, JSONB `content`, `content_hash`) |
| `extension_config_active` | Указатель на активную ревизию по namespace |
| Redis `@cached_read` | TTL 300 с, stale 3600 с; ключ по namespace |

Публичный GET не сидирует конфиг: начальная ревизия создаётся миграцией.
