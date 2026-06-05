# MarketHacker — Документация

Центральный репозиторий документации проекта [MarketHacker](https://github.com/MarketHackerLabs).

MarketHacker — расширение для браузеров на движке Chromium, расширяющее функционал маркетплейсов для продавцов (Wildberries, Ozon и др.).

## Репозитории проекта

| Репозиторий | Назначение |
|-------------|------------|
| [extension-chrome](https://github.com/MarketHackerLabs/extension-chrome) | Chromium-расширение (React, TypeScript, MV3) |
| [backend](https://github.com/MarketHackerLabs/backend) | API-сервер (Python, FastAPI) |
| **docs** (этот репозиторий) | Архитектура, спецификации, руководства |

## Содержание

### Архитектура бекенда

- [Обзор](./architecture/README.md) — цели, ограничения, ключевые решения
- [Технологический стек](./architecture/tech-stack.md)
- [Структура проекта](./architecture/structure.md)
- [Модель данных](./architecture/data-model.md)
- [Контроль доступа](./architecture/access-control.md)
- [Доступ к кабинетам MP](./architecture/marketplace-access-model.md) — section permissions, proxy, extension
- [Аутентификация и авторизация](./architecture/authentication.md)
- [Безопасность](./architecture/security.md)
- [Дизайн API](./architecture/api-design.md)
- [Фоновые задачи](./architecture/background-jobs.md)
- [Инфраструктура](./architecture/infrastructure.md)
- [Дорожная карта](./architecture/roadmap.md)

## Как читать

Документация организована от общего к частному. Для быстрого ознакомления начните с [обзора архитектуры](./architecture/README.md), затем переходите к интересующим разделам.

## Статус

Документация отражает проектные решения на этапе проектирования (MVP). Обновляется по мере реализации.
