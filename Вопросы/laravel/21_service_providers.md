# Service Providers

Простыми словами: Service Provider — место, где регистрируются сервисы, middleware, события, конфиг.
Точки входа для регистрации сервисов/настроек.
```php
class CustomServiceProvider extends ServiceProvider {
    public function register() {
        $this->app->singleton(PaymentGateway::class, fn($app) => new PaymentGateway($app['config']['payment.api_key']));
    }
    public function boot() {
        $this->app['router']->aliasMiddleware('payment', PaymentMiddleware::class);
        $this->publishes([__DIR__.'/../config/payment.php' => config_path('payment.php')]);
        View::composer('payment.*', PaymentComposer::class);
    }
}
// config/app.php: добавить в providers
```
