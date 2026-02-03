## Вопрос: Service Providers
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Service Provider — место, где регистрируются сервисы, middleware, события, конфиг.

## Ответ

Service Provider (сервис-провайдер) -- это центральное место в Laravel, где происходит начальная загрузка (bootstrapping) приложения. Каждый провайдер отвечает за регистрацию биндингов в service container, настройку event listeners, middleware, маршрутов и любых других компонентов фреймворка. Все провайдеры перечислены в массиве `providers` файла `config/app.php` (или, начиная с Laravel 11, регистрируются через `bootstrap/providers.php`).

Провайдер содержит два ключевых метода: `register()` и `boot()`. Метод `register()` вызывается первым для всех провайдеров -- в нём следует только привязывать (bind) классы в контейнер. На этом этапе другие сервисы могут быть ещё не зарегистрированы, поэтому обращаться к ним нельзя. Метод `boot()` вызывается после того, как все провайдеры отработали `register()`, и здесь уже можно использовать любые сервисы: настраивать маршруты, middleware, view composers, публиковать ресурсы пакетов и т.д.

Laravel различает несколько типов привязок в контейнере. `bind()` создаёт новый экземпляр при каждом resolve. `singleton()` создаёт объект один раз и возвращает его при последующих вызовах. `scoped()` действует как singleton в рамках одного request lifecycle. Выбор типа привязки зависит от того, должен ли сервис хранить состояние между вызовами.

```php
use Illuminate\Support\ServiceProvider;

class PaymentServiceProvider extends ServiceProvider
{
    /**
     * Регистрируем биндинги в контейнере.
     * Здесь НЕЛЬЗЯ обращаться к другим сервисам -- они могут быть ещё не загружены.
     */
    public function register(): void
    {
        // Singleton -- один экземпляр на всё приложение
        $this->app->singleton(PaymentGateway::class, function ($app) {
            return new PaymentGateway(
                apiKey: $app['config']['payment.api_key'],
                sandbox: $app['config']['payment.sandbox'],
            );
        });

        // Bind к интерфейсу -- подмена реализации через конфиг
        $this->app->bind(
            PaymentProcessorInterface::class,
            fn($app) => match($app['config']['payment.driver']) {
                'stripe' => new StripeProcessor($app[PaymentGateway::class]),
                'paypal' => new PayPalProcessor($app[PaymentGateway::class]),
                default  => throw new \InvalidArgumentException('Unknown payment driver'),
            }
        );

        // Мержим конфиг пакета с пользовательским
        $this->mergeConfigFrom(__DIR__.'/../config/payment.php', 'payment');
    }

    /**
     * Boot -- все сервисы уже зарегистрированы, можно их использовать.
     */
    public function boot(): void
    {
        // Регистрируем middleware
        $this->app['router']->aliasMiddleware('payment', PaymentMiddleware::class);

        // Публикуем конфиг для пользователя (vendor:publish)
        $this->publishes([
            __DIR__.'/../config/payment.php' => config_path('payment.php'),
        ], 'payment-config');

        // Публикуем миграции
        $this->publishes([
            __DIR__.'/../database/migrations/' => database_path('migrations'),
        ], 'payment-migrations');

        // View composer -- передаём данные во все views payment.*
        View::composer('payment.*', PaymentComposer::class);

        // Загрузка маршрутов пакета
        $this->loadRoutesFrom(__DIR__.'/../routes/payment.php');

        // Загрузка views пакета
        $this->loadViewsFrom(__DIR__.'/../resources/views', 'payment');
    }
}
```

### Deferred Providers (отложенные провайдеры)

Если провайдер регистрирует только биндинги в контейнере, его можно сделать отложенным (deferred). Такой провайдер загружается только тогда, когда один из его сервисов реально запрашивается. Это ускоряет загрузку приложения.

```php
use Illuminate\Contracts\Support\DeferrableProvider;

class HeavyReportServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void
    {
        $this->app->singleton(ReportGenerator::class, function ($app) {
            return new ReportGenerator($app['config']['reports']);
        });
    }

    // Указываем, какие сервисы предоставляет этот провайдер
    public function provides(): array
    {
        return [ReportGenerator::class];
    }
}
```

### Регистрация в Laravel 11

В Laravel 11 структура упростилась: файл `config/app.php` больше не содержит массив `providers`. Вместо этого провайдеры регистрируются в `bootstrap/providers.php`:

```php
// bootstrap/providers.php
return [
    App\Providers\AppServiceProvider::class,
    App\Providers\PaymentServiceProvider::class,
];
```

### Практические советы

При создании сервис-провайдеров следуйте принципу единственной ответственности: один провайдер -- один домен (платежи, уведомления, аналитика). Не перегружайте `AppServiceProvider` -- для крупных проектов создавайте отдельные провайдеры. Используйте `php artisan make:provider MyServiceProvider` для генерации. Помните, что порядок провайдеров в массиве имеет значение: если провайдер B зависит от сервисов провайдера A, A должен быть выше в списке.

## Примеры

1. Регистрация биндингов в `register()`.
2. Подписка на события в `boot()`.
3. Подключение провайдера в `config/app.php`.

## Доп. теория

1. Провайдеры загружаются на этапе bootstrap.
2. Часть провайдеров может быть deferred.
