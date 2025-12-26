## Вопрос: Аутентификация API (Sanctum)
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Sanctum выдаёт токены для API и проверяет их на запросах.

## Ответ
```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```
```php
// Модель User
use Laravel\Sanctum\HasApiTokens;
class User extends Authenticatable { use HasApiTokens, HasFactory, Notifiable; }

// Kernel api group
'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    'throttle:api',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],

$token = $user->createToken('api-token', ['read','write'])->plainTextToken;

Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', fn(Request $r) => $r->user());
});

// Проверка и отзыв
if ($request->user()->tokenCan('read')) { /* ... */ }
$request->user()->currentAccessToken()->delete();
```
