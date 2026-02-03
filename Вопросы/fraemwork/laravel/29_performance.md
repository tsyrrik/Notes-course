## Вопрос: Оптимизация производительности
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Базовые приёмы ускорения: индексы, eager loading, кэш, очереди, chunk/cursor.

## Ответ
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

## Примеры

1. `php artisan route:cache`.
2. `php artisan config:cache`.
3. Eager loading вместо N+1.

## Доп. теория

1. Route/config/view cache ускоряет cold‑start.
2. N+1 — частая причина деградации.
