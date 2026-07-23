# Восстановление пароля

См. также: [Аутентификация](../architecture/authentication.md), [MFA](./auth-mfa-client.md).

## Сам пользователь (Manager Portal)

1. На `/forgot-password` вводит email → `POST /auth/password-reset/request`.
2. Ответ всегда 204, даже если такого email нет (чтобы нельзя было угадать аккаунты).
3. Если пользователь есть — на почту уходит ссылка `{MANAGER_PORTAL_URL}/reset-password/{token}` (живёт час).
4. На странице сброса: проверка токена, новый пароль → `POST /auth/password-reset/confirm`.
5. Все сессии сбрасываются. MFA, если была включена, остаётся.

Повторно той же ссылкой воспользоваться нельзя. Новый запрос сброса гасит старые неиспользованные ссылки.

## Смена пароля из настроек

`POST /auth/password/change` — нужен текущий пароль. После смены все сессии тоже сбрасываются.

## Админка

Сброс пароля и MFA — у суперадмина и у сотрудников поддержки с нужным правом. Права выдаются там же, где остальные права поддержки: **Поддержка → Сотрудники**.

| Что | Endpoint | Право |
|-----|----------|-------|
| Список / карточка пользователя | `GET /admin/users`, `GET /admin/users/{id}` | `users:read` |
| Сброс пароля (письмо со ссылкой) | `POST /admin/users/{id}/reset-password` | `users:password:reset` |
| Сброс MFA | `POST /admin/users/{id}/mfa/reset` | `users:mfa:reset` |

Временный пароль в ответе API не отдаём — только ссылка на почту.

**Сброс MFA поддержкой:** сначала подтверждение владения аккаунтом — см.
[инструкцию для поддержки](../operations/support-mfa-reset.md).

## Почта

`EMAIL_PROVIDER`: `console` (в лог API через structlog, событие `email.console`), `smtp` (Gmail и т.п.), `unisender_go`. Параметры — в backend `.env.example`.

Для Unisender Go в HTML/plaintext автоматически добавляется своя ссылка
`{{UnsubscribeUrl}}` — иначе провайдер вставляет стандартный блок «Это сообщение
было отправлено … Отписаться от рассылки». См. [документацию Unisender](https://godocs.unisender.ru/unsubscribe-link#customizing).
