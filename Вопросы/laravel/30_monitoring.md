# Мониторинг и профилирование

Простыми словами: Смотрим на медленные запросы и метрики (Telescope, DB::listen, логи).
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
