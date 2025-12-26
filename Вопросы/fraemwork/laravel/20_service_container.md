## Вопрос: Service Container
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

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
