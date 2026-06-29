# MarketHacker — Документация

Центральный репозиторий документации проекта [MarketHacker](https://github.com/MarketHackerLabs).

MarketHacker — платформа для управления продажами на маркетплейсах (Wildberries, Ozon и др.). Включает Chromium-расширение, API-сервер, панель администратора и портал менеджеров с встроенным WB Portal Proxy.

## Репозитории проекта

| Репозиторий | Назначение |
|-------------|------------|
| [extension-chrome](https://github.com/MarketHackerLabs/extension-chrome) | Chromium-расширение (React, TypeScript, MV3) |
| [backend](https://github.com/MarketHackerLabs/backend) | API-сервер (Python, FastAPI) + WB Portal Proxy |
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
- [Доступ к кабинетам MP](./architecture/marketplace-access-model.md) — section permissions, proxy, extension
- [WB Portal Proxy](./architecture/wb-portal-proxy.md) — reverse proxy к seller.wildberries.ru, onboarding, JS-инжект
- [Аутентификация и авторизация](./architecture/authentication.md)
- [Безопасность](./architecture/security.md)
- [Дизайн API](./architecture/api-design.md)
- [Биллинг и оплата](./architecture/billing.md) — ЮKassa, webhook, промокоды, фоновая сверка, автопродление
- [Фоновые задачи](./architecture/background-jobs.md)
- [Parser Service](./architecture/parser.md) — платформа фоновых задач и аналитики
- [Разработка парсеров](./architecture/parser-development.md) — Kafka, новые парсеры, чеклист деплоя
- [Инфраструктура](./architecture/infrastructure.md)
- [Дорожная карта](./architecture/roadmap.md)

## Как читать

Документация организована от общего к частному. Для быстрого ознакомления начните с [обзора архитектуры](./architecture/README.md), затем переходите к интересующим разделам.

## Статус

Документация отражает текущее состояние реализации (июнь 2025):

- **WB Portal Proxy** — развёрнут на `wb-proxy.markethacker.ru`: reverse proxy, JS guard, 6 групп section permissions, блокировка профиля WB
- **Manager Portal** — `team.markethacker.ru`: кабинеты, команда, приглашения, section grants, биллинг
- **Биллинг** — ЮKassa (RUB) + Stripe, webhook, фоновая сверка платежей, автопродление
- **Безопасность прокси** — server-only cookies (`wbx-validation-key`, `x-supplier-id`) не попадают в браузер менеджера
