## Вопрос: Конфигурация и окружения
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

## Простой ответ
настройки лежат в `config/` и зависят от окружения (`.env`, `.env.local`). Параметры доступны через `%env(VAR)%` и `parameters`.

## Ответ
- Окружения: `dev`, `prod`, `test`.
- Ключи в `.env` не коммитят секреты — используют `.env.local` или vault.

```env
APP_ENV=dev
APP_DEBUG=1
DATABASE_URL=postgresql://user:pass@127.0.0.1:5432/db
```
