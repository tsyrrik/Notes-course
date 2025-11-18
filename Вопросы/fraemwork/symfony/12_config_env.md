# Конфигурация и окружения

Простыми словами: настройки лежат в `config/` и зависят от окружения (`.env`, `.env.local`). Параметры доступны через `%env(VAR)%` и `parameters`.

- Окружения: `dev`, `prod`, `test`.
- Ключи в `.env` не коммитят секреты — используют `.env.local` или vault.

```env
APP_ENV=dev
APP_DEBUG=1
DATABASE_URL=postgresql://user:pass@127.0.0.1:5432/db
```
