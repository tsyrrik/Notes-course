## Вопрос: Middleware
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Middleware — фильтры запросов (auth, логирование, CORS, throttle) до/после контроллера.

## Ответ

Middleware (промежуточное ПО) — это фильтры, через которые проходит HTTP-запрос до (или после) обработки контроллером. Middleware позволяют инспектировать, модифицировать или отклонять запрос на любом этапе. Типичные применения: аутентификация, авторизация, CORS, rate limiting, логирование, проверка заголовков, обслуживание (maintenance mode).

Middleware работает по принципу «луковицы» (onion): запрос проходит через каждый слой middleware внутрь (к контроллеру), а ответ проходит обратно через эти же слои наружу. Это позволяет middleware обрабатывать как входящий запрос (before middleware), так и исходящий ответ (after middleware).

```bash
php artisan make:middleware EnsureUserIsAdmin
```

**Before и After middleware:**

```php
// Before middleware — обрабатывает ДО контроллера
class EnsureUserIsAdmin
{
    public function handle(Request $request, Closure $next): Response
    {
        if (! $request->user()?->isAdmin()) {
            abort(403, 'Доступ запрещён');
        }

        return $next($request); // передаём запрос дальше
    }
}

// After middleware — обрабатывает ПОСЛЕ контроллера
class AddResponseHeaders
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request); // сначала контроллер

        $response->headers->set('X-Custom-Header', 'MyApp');
        $response->headers->set('X-Request-Id', Str::uuid());

        return $response;
    }
}

// Terminable middleware — выполняется ПОСЛЕ отправки ответа клиенту
class LogRequest
{
    public function handle(Request $request, Closure $next): Response
    {
        return $next($request);
    }

    public function terminate(Request $request, Response $response): void
    {
        // Логируем после отправки ответа — не влияет на время ответа
        Log::info('Request processed', [
            'url' => $request->fullUrl(),
            'status' => $response->getStatusCode(),
            'duration' => microtime(true) - LARAVEL_START,
        ]);
    }
}
```

**Middleware с параметрами:**

```php
class CheckRole
{
    public function handle(Request $request, Closure $next, string ...$roles): Response
    {
        if (! $request->user()?->hasAnyRole($roles)) {
            abort(403);
        }

        return $next($request);
    }
}

// Использование
Route::get('/admin', [AdminController::class, 'index'])->middleware('role:admin,moderator');
```

## Типы и регистрация

**В Laravel 11** middleware регистрируются в `bootstrap/app.php` (файл `app/Http/Kernel.php` удалён):

```php
// bootstrap/app.php
return Application::configure(basePath: dirname(__DIR__))
    ->withMiddleware(function (Middleware $middleware) {
        // Глобальные middleware — применяются ко ВСЕМ запросам
        $middleware->append(\App\Http\Middleware\LogRequest::class);
        $middleware->prepend(\App\Http\Middleware\TrustProxies::class);

        // Алиасы для маршрутов
        $middleware->alias([
            'admin' => \App\Http\Middleware\EnsureUserIsAdmin::class,
            'role'  => \App\Http\Middleware\CheckRole::class,
        ]);

        // Группы middleware
        $middleware->group('custom-web', [
            \Illuminate\Cookie\Middleware\EncryptCookies::class,
            \Illuminate\Session\Middleware\StartSession::class,
        ]);

        // Добавить middleware к группе web
        $middleware->web(append: [
            \App\Http\Middleware\HandleInertiaRequests::class,
        ]);

        // Добавить middleware к группе api
        $middleware->api(prepend: [
            \App\Http\Middleware\EnsureJsonResponse::class,
        ]);
    })
    ->create();
```

**Применение middleware к маршрутам:**

```php
// К отдельному маршруту
Route::get('/admin', [AdminController::class, 'index'])->middleware('admin');

// К группе маршрутов
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});

// Исключение middleware
Route::middleware('auth')->group(function () {
    Route::get('/profile', [ProfileController::class, 'show']);
    Route::get('/public', [PageController::class, 'show'])->withoutMiddleware('auth');
});

// В контроллере
class UserController extends Controller
{
    public function __construct()
    {
        $this->middleware('auth')->except(['index', 'show']);
        $this->middleware('admin')->only(['destroy']);
    }
}
```

**Встроенные middleware Laravel:**
- `auth` — проверка аутентификации
- `auth.basic` — HTTP Basic Auth
- `guest` — только неаутентифицированные
- `verified` — email подтверждён
- `throttle:60,1` — rate limiting (60 запросов в минуту)
- `signed` — проверка подписанного URL
- `can:permission` — авторизация через Gate/Policy

**Практический совет:** в Laravel 11 не нужно создавать Kernel.php — всё настраивается в `bootstrap/app.php`. Используйте terminable middleware для логирования и аналитики — это не замедляет ответ. Rate limiting через `throttle` middleware защищает API от злоупотреблений. Для сложной авторизации комбинируйте middleware `can:` с Policy-классами.

## Примеры

1. `Route::middleware('auth')->group(...)`.
2. Rate limit: `throttle:60,1`.
3. Свой middleware: `php artisan make:middleware EnsureAdmin`.

## Доп. теория

1. Порядок middleware влияет на результат.
2. Есть глобальные, групповые и маршрутные middleware.
