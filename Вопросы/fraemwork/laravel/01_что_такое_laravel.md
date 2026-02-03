## Вопрос: Что такое Laravel и основные особенности
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Laravel — фреймворк на PHP, который даёт готовые инструменты (ORM, шаблоны, маршруты, IoC) и упрощает сборку веб-приложений.

## Ответ

Laravel — это полнофункциональный PHP-фреймворк с открытым исходным кодом, построенный на архитектуре MVC (Model-View-Controller). Он был создан Тейлором Отвеллом в 2011 году и с тех пор стал самым популярным PHP-фреймворком в мире. Laravel предоставляет элегантный синтаксис и богатую экосистему инструментов, которые значительно ускоряют разработку веб-приложений.

**Ключевые компоненты фреймворка:**
- **Eloquent ORM** — ActiveRecord-реализация для работы с базами данных через модели
- **Blade** — мощный шаблонизатор с наследованием и компонентами
- **Artisan CLI** — консольный инструмент для генерации кода, миграций, запуска очередей и других задач
- **Service Container (IoC)** — контейнер инверсии управления для Dependency Injection
- **Middleware** — прослойки для фильтрации HTTP-запросов (аутентификация, CORS, throttling)
- **Queue System** — система очередей для асинхронной обработки задач (email, уведомления)
- **Event System** — система событий и слушателей для слабосвязанной архитектуры
- **Service Providers** — центральное место регистрации сервисов приложения

**Что изменилось в Laravel 11:**
Laravel 11 значительно упростил структуру проекта. Убран файл `app/Http/Kernel.php` — middleware теперь регистрируются в `bootstrap/app.php`. Убраны стандартные сервис-провайдеры (кроме `AppServiceProvider`). Появились новые возможности: health-check endpoint из коробки, ротация ключей шифрования, упрощённая конфигурация. Минимальная версия PHP — 8.2.

```php
// Структура bootstrap/app.php в Laravel 11
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        api: __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
    )
    ->withMiddleware(function (Middleware $middleware) {
        $middleware->alias([
            'admin' => \App\Http\Middleware\EnsureUserIsAdmin::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        //
    })
    ->create();
```

**Экосистема Laravel** включает официальные пакеты: **Sanctum** (API-аутентификация), **Horizon** (мониторинг очередей Redis), **Telescope** (отладка), **Forge** / **Vapor** (деплой), **Nova** (админ-панель), **Livewire** (реактивные компоненты без JS), **Breeze** / **Jetstream** (стартовые наборы аутентификации), **Pennant** (feature flags).

```php
// Проверка версии
php artisan --version

// Создание нового проекта Laravel 11
composer create-project laravel/laravel my-app
// или через installer
laravel new my-app
```

**Практический совет:** при старте проекта на Laravel 11 используйте `laravel new` с интерактивным установщиком — он предложит выбрать стартовый набор (Breeze/Jetstream), систему тестирования (Pest/PHPUnit), базу данных и другие настройки. Актуальная ветка — Laravel 11; LTS сейчас 10 (поддержка до 2026); мажорные релизы выходят ежегодно.

## Примеры

1. Создание проекта: `composer create-project laravel/laravel my-app`.
2. Запуск локально: `php artisan serve`.
3. Маршрут: `Route::get('/', fn () => view('welcome'));`.

## Доп. теория

1. Laravel состоит из компонентов Illuminate и поверхностного DX слоя.
2. Facade — статический прокси к сервисам контейнера.
