## Вопрос: Service Container
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Service Container — IoC-коробка, из которой Laravel создаёт объекты и внедряет зависимости.

## Ответ
IoC-контейнер для управления зависимостями.
```php
// Binding в провайдере
class AppServiceProvider extends ServiceProvider {
    public function register() {
        $this->app->bind(
            UserRepositoryInterface::class,
            DatabaseUserRepository::class
        );
        $this->app->singleton('twitter', fn($app) => new TwitterAPI($app['config']['twitter']));
    }
}

// Resolving
$repo = app(UserRepositoryInterface::class);
```

## Примеры

1. `app()->make(Service::class)`.
2. `$this->app->singleton(Logger::class, ...)`.
3. Привязка интерфейса к реализации.

## Доп. теория

1. Container — основа DI и Facade‑ов.
2. Поддерживаются singleton и scoped биндинги.
