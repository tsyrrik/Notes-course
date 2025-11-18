# Оптимизация производительности

Простыми словами: Базовые приёмы ускорения: индексы, eager loading, кэш, очереди, chunk/cursor.
## БД
- Eager loading вместо N+1.
- Выбирай нужные поля (`select`), ставь индексы.
- Используй `limit/offset` или курсоры для больших выборок.

## Кэширование
- `php artisan config:cache`, `route:cache`, `view:cache`, `event:cache`.
- `Cache::remember` для результатов запросов.
- HTTP cache: `cache.headers` middleware.

## Код
- `chunk`/`cursor` для больших датасетов.
- Тяжёлые операции в очереди.
- Не ставь Xdebug в prod.
```php
User::chunk(1000, fn($users) => $users->each(fn($u) => process($u)));
User::cursor()->each(fn($u) => process($u));
```
