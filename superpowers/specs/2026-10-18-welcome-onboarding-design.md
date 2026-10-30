# Welcome / быстрый старт после регистрации

Дата: 2026-10-18  
Статус: approved for planning  

## Цель

После регистрации (сайт и Telegram) показать единую страницу быстрого старта в Manager Portal: дозаполнение почты/пароля для TG-аккаунтов (если нужно) и два шага — установить расширение и открыть кабинет маркетплейса. Визуально — в дизайне портала; по смыслу — как референс «Быстрый старт».

## Решения (зафиксировано)

| Тема | Решение |
|---|---|
| Страница | `/welcome` в Manager Portal |
| TG + форма | Одна страница: сначала почта/пароль, затем шаги с расширением |
| Регистрация на сайте | Тот же `/welcome` без формы — сразу шаги |
| Повторный показ | Пока onboarding не завершён; после завершения — обычный вход |
| Выход | Всегда `/dashboard` (сайт и Telegram) |
| Хранение статуса | `users.onboarding_completed_at` + зеркало в `localStorage` |
| Старый `/complete-profile` | Постоянный редирект на `/welcome` |
| Демо-ссылка «как это работает» | Вне scope v1 |
| Шаг 02 кабинет | CTA на `https://seller.wildberries.ru`; отдельная кнопка Ozon — нет |

## Scope

### In

1. Страница `/welcome` (auth-layout без сайдбара).
2. Редиректы: `/register` → `/welcome`; Telegram login при `needsProfileCompletion` → `/welcome`; гейт в `ManagerShell` / login-потоках при незавершённом onboarding.
3. Бэкенд: колонка `onboarding_completed_at`, backfill существующих пользователей, поля в me, `POST /auth/onboarding/complete`.
4. Перенос UX дозаполнения профиля с `/complete-profile` на `/welcome`; `/complete-profile` → redirect.
5. Клиентское зеркало в `localStorage` и синхронизация с me.
6. Обновление клиентской доки Telegram-auth (путь после complete / needsProfileCompletion).

### Out

- Ссылка на демо / «Посмотреть, как это работает».
- Онбординг внутри Chrome-расширения и market-navigators.
- Изменение текста отложенного TG-welcome в боте (ссылка на `/welcome` — отдельная задача).
- Отдельная CTA на кабинет Ozon.
- Пиксель-в-пиксель копия chatgpt-референса (только структура и смысл).

## Флоу

```
Сайт /register
  → tokens
  → /welcome (needsProfileCompletion=false, needsOnboarding=true)
  → шаги 01/02
  → POST /auth/onboarding/complete + localStorage
  → /dashboard

Telegram /login/telegram?token=
  → exchange
  → если needsProfileCompletion → /welcome
  → форма email/password → POST /auth/telegram/complete-profile
  → шаги 01/02
  → POST /auth/onboarding/complete + localStorage
  → /dashboard

Повторный вход
  → me: needsOnboarding=false → портал (не /welcome)
  → me: needsOnboarding=true  → /welcome (только шаги, если профиль уже полный)
  → needsProfileCompletion    → /welcome (форма, затем шаги)

Прямой заход на /welcome при завершённом onboarding
  → redirect /dashboard
```

Гейт клиента (после auth):

- если `needsProfileCompletion || needsOnboarding` и pathname ≠ `/welcome` → `replace("/welcome")`;
- `/welcome` не оборачивается в `ManagerShell` с сайдбаром (как login / прежний complete-profile).

## Backend

### Модель `users`

Добавить:

- `onboarding_completed_at: datetime | None` (timestamptz, nullable)

Семантика:

- `NULL` — onboarding не завершён (`needsOnboarding = true`);
- не-NULL — завершён (`needsOnboarding = false`).

Миграция Alembic:

1. Добавить колонку nullable.
2. Backfill: для всех существующих строк на момент миграции  
   `onboarding_completed_at = created_at` (или `now()`), чтобы не загнать текущую базу на `/welcome`.
3. Новые пользователи после деплоя создаются с `NULL` (register + Telegram `/start`).

Точки создания пользователя (`AuthService.register`, Telegram create user) **не** выставляют `onboarding_completed_at` — остаётся `NULL`.

### `GET` текущего пользователя (me)

В ответ добавить:

| Поле | Тип | Описание |
|---|---|---|
| `onboardingCompletedAt` | `string \| null` (ISO datetime) | Когда завершён onboarding |
| `needsOnboarding` | `boolean` | `onboardingCompletedAt === null` |

Существующие поля не меняются.  
`needsProfileCompletion` остаётся на `GET /auth/telegram/status` и в ответе `POST /auth/telegram/exchange` — без изменения контракта.

### `POST /auth/onboarding/complete`

**Auth:** Bearer access token.  
**Тело:** пустое или `{}`.

Поведение:

- если `onboarding_completed_at` уже задан — идемпотентный `200`, состояние без изменений;
- иначе `onboarding_completed_at = now(UTC)`, сохранить;
- ответ:

```json
{
  "onboardingCompletedAt": "2026-10-18T12:00:00Z",
  "needsOnboarding": false
}
```

Ошибки: стандартные auth (`401`); пользователь неактивен — как в прочих auth-ручках.

### Без изменений API

- `POST /auth/telegram/complete-profile` — тот же контракт.
- `GET /auth/telegram/status` — `needsProfileCompletion` без изменений.
- `POST /auth/register` — без новых полей в ответе токенов; клиент после register идёт на `/welcome` и читает me.

## Manager Portal

### Страница `/welcome`

Компоненты и стиль: `GridBackdrop`, `BrandLogo`, `Button`, `Input`, lime/карточки портала.

Содержание:

1. Шапка: логотип; справа имя / инициал из me.
2. Бейдж: «Регистрация завершена» после полного профиля; пока `needsProfileCompletion` — «Почти готово».
3. Заголовок: `{Имя}, давайте запустим MarketHacker` — первое слово `fullName`, иначе local-part email, иначе заголовок без имени.
4. Подзаголовок: два шага до инструментов в кабинете маркетплейса.
5. Блок профиля **только если** `needsProfileCompletion`: email, пароль, повтор пароля → `complete-profile`. До успеха шаги 01/02 приглушены и без рабочих CTA.
6. Шаг 01: установить расширение; primary → `TELEGRAM_WELCOME_PLUGIN_URL` / общая константа Chrome Web Store в portal; secondary «Уже установил».
7. Шаг 02: открыть кабинет; CTA → `https://seller.wildberries.ru` (новая вкладка).
8. «Уже установил» / завершение шагов: `POST /auth/onboarding/complete`, запись `localStorage`, `router.replace("/dashboard")`.

### `localStorage`

- Ключ вида `mh_onboarding_completed` (значение: userId или ISO timestamp — один выбранный формат на всю реализацию).
- После успешного complete — записать.
- При загрузке me: если `needsOnboarding` — удалить устаревшее зеркало; если не needs — синхронизировать зеркало.
- Зеркало не заменяет проверку me при гейтах; ускоряет/сглаживает UX, источник правды — бэкенд.

### Редиректы и гейты

| Место | Поведение |
|---|---|
| `/register` success | → `/welcome` (вместо `/dashboard`) |
| `/login/telegram` finish | `needsProfileCompletion` или (после me) `needsOnboarding` → `/welcome`; иначе `/dashboard` |
| `/complete-profile` | `replace("/welcome")` |
| `ManagerShell` | если `needsProfileCompletion` или `needsOnboarding` → `/welcome` |
| Login email/password success | если `needsOnboarding` → `/welcome`, иначе прежний target / `/dashboard` |

Точный путь чтения me после login — по существующим паттернам portal (не плодить второй auth bootstrap).

### Константа URL расширения

Вынести URL Chrome Web Store в общую константу portal (значение = тот же URL, что `TELEGRAM_WELCOME_PLUGIN_URL` в backend). Не хардкодить разные ссылки в welcome UI и доках бота.

## Ошибки (UI)

| Ситуация | Поведение |
|---|---|
| Нет auth на `/welcome` | redirect login |
| Ошибка complete-profile | текст у формы; шаги закрыты |
| Ошибка onboarding/complete | сообщение + «Повторить»; не уходить в dashboard |
| Ошибка me/status | «Не удалось загрузить» + повтор |
| Технические детали | только логи; в UI — «что произошло» и «что сделать» |

## Тесты

**Backend**

- Миграция: колонка есть; backfill для «старых» строк; новые create → `NULL`.
- `POST /auth/onboarding/complete`: первый вызов ставит timestamp; повтор идемпотентен.
- Me отдаёт `onboardingCompletedAt` / `needsOnboarding`.

**Portal (unit/интеграция по принятому в репо стилю)**

- Register ведёт на `/welcome`.
- TG с `needsProfileCompletion` ведёт на `/welcome`; шаги недоступны до submit формы.
- Сайт-пользователь видит шаги сразу.
- Complete onboarding → dashboard; повторный гейт не отправляет на welcome.

## Документация

- Обновить `docs/integrations/telegram-auth-client.md`: клиент после exchange / needsProfileCompletion открывает `/welcome` (не обязательно `/complete-profile`); complete-profile API без изменений.
- При необходимости краткая пометка в `docs/architecture/telegram-bot.md` про web-onboarding после входа.
- Этот файл — источник правды по продуктовому решению.

## Риски

| Риск | Митигация |
|---|---|
| Все существующие юзеры попадут на welcome | Backfill в миграции |
| Расхождение localStorage и сервера | Гейт всегда сверяет me |
| Закладки на `/complete-profile` | Redirect на `/welcome` |
| Дубли URL расширения | Одна константа, совпадающая с backend TG welcome |
