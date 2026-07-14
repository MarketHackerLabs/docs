# MarketHacker — Документация

Центральный репозиторий документации проекта [MarketHacker](https://github.com/MarketHackerLabs).

MarketHacker — платформа для управления продажами на маркетплейсах (Wildberries, Ozon и др.). Включает Chromium-расширение, API-сервер, панель администратора и портал менеджеров с встроенным WB Gateway.

## Репозитории проекта

| Репозиторий | Назначение |
|-------------|------------|
| [extension-chrome](https://github.com/MarketHackerLabs/extension-chrome) | Chromium-расширение (React, TypeScript, MV3) |
| [backend](https://github.com/MarketHackerLabs/backend) | API-сервер (Python, FastAPI) + WB Gateway + WB Connect |
| [admin-panel](https://github.com/MarketHackerLabs/admin-panel) | Панель администратора (Next.js, `admin.markethacker.ru`) |
| [manager-portal](https://github.com/MarketHackerLabs/manager-portal) | Портал менеджеров (Next.js, `team.markethacker.ru`) |
| [parser](https://github.com/MarketHackerLabs/parser) | Платформа фоновых задач (Python, FastAPI, ARQ) |
| **docs** (этот репозиторий) | Архитектура, спецификации, руководства |

## Содержание

### Архитектура бекенда

- [Обзор](./architecture/README.md) — цели, ограничения, ключевые решения
- [Технологический стек](./architecture/tech-stack.md)
- [Структура проекта](./architecture/structure.md)
- [Модель данных](./architecture/data-model.md)
- [Контроль доступа](./architecture/access-control.md)
- [Доступ к кабинетам MP](./architecture/marketplace-access-model.md) — section permissions, WB Gateway
- [WB Gateway & Guided Connect](./architecture/wb-portal-proxy.md) — reverse proxy к seller.wildberries.ru, Guided Connect, JS-инжект
- [Аутентификация и авторизация](./architecture/authentication.md)
- [Безопасность](./architecture/security.md)
- [Дизайн API](./architecture/api-design.md)
- [Кэширование](./architecture/caching.md) — Redis response cache, `@cached_read`, инвалидация
- [Биллинг и оплата](./architecture/billing.md) — ЮKassa, webhook, промокоды, фоновая сверка, автопродление
- [Партнёры](./architecture/partners.md) — кампании, атрибуция, комиссии, аналитика
- [Продуктовые промо](./architecture/product-promotions.md) — баннеры / CTA, placements, targeting
- [Фоновые задачи](./architecture/background-jobs.md)
- [Parser Service](./architecture/parser.md) — платформа фоновых задач и аналитики
- [Разработка парсеров](./architecture/parser-development.md) — Kafka, новые парсеры, чеклист деплоя
- [Инфраструктура](./architecture/infrastructure.md)
- [Дорожная карта](./architecture/roadmap.md)

## Как читать

Документация организована от общего к частному. Для быстрого ознакомления начните с [обзора архитектуры](./architecture/README.md), затем переходите к интересующим разделам.

## Статус

Документация отражает текущее состояние реализации (июль 2026):

- **WB Gateway** — развёрнут на `wb-proxy.markethacker.ru`: reverse proxy, JWT-сессия с мгновенным Redis-отзывом, JS guard, default-deny ACL, 6 групп section permissions, блокировка профиля WB
- **WB Connect (Guided Connect)** — развёрнут на `wb-connect.markethacker.ru` + onboarding-поддомены: привязка WB-сессии через popup без DevTools; HttpOnly cookies накапливаются server-side
- **Manager Portal** — `team.markethacker.ru`: кабинеты, команда, приглашения, section grants, биллинг, `WbConnectModal`
- **Биллинг** — ЮKassa (RUB) + Stripe, webhook, фоновая сверка платежей, автопродление
- **Search Tags API** — `GET /api/v1/search-tags/queries` и `/queries/monthly`: read-only данные парсера WB из ClickHouse, фича `search_tags`
- **JSON camelCase** — все REST request/response bodies в camelCase (`CamelModel`); query-параметры URL — snake_case
- **Parser: market niche** — парсер `wb_market_niche` (отчёт «Анализ ниш»): ZIP/XLSX → Kafka → `wb_market_niche` в ClickHouse; API parser-сервиса `POST /api/v1/jobs/wb-market-niche`
- **Кэширование** — Redis response cache для read-only данных (ClickHouse, каталоги PG), декоратор `@cached_read`
- **Безопасность прокси** — server-only cookies (`wbx-validation-key`, `x-supplier-id`) не попадают в браузер менеджера
