## Вопрос: Мониторинг и профилирование
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Смотрим на медленные запросы и метрики (Telescope, DB::listen, логи).

## Ответ
- Laravel Telescope (dev): `composer require laravel/telescope --dev`, `php artisan telescope:install`.
- Логирование медленных запросов:
```php
DB::listen(function ($query) {
    if ($query->time > 1000) {
        Log::warning('Slow query', ['sql'=>$query->sql,'bindings'=>$query->bindings,'time'=>$query->time]);
    }
});
```
- Кастомные метрики через middleware (время/память).

## Примеры

1. Telescope для запросов и очередей.
2. Horizon для мониторинга Redis‑очередей.
3. Каналы логов в `config/logging.php`.

## Доп. теория

1. Horizon и Telescope — dev/ops инструменты.
2. APM‑системы дают трассировки и метрики.
