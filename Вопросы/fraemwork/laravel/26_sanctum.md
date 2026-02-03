## Вопрос: Аутентификация API (Sanctum)
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Sanctum выдаёт токены для API и проверяет их на запросах.

## Ответ

Laravel Sanctum -- лёгкий пакет аутентификации для SPA (Single Page Application), мобильных приложений и простых API. В отличие от Passport (полноценный OAuth2-сервер), Sanctum не использует JWT и OAuth2, а предоставляет два механизма: API-токены и cookie-based аутентификацию для SPA. Sanctum идеально подходит, когда не нужна сложная OAuth2-инфраструктура с grant types и scopes.

В Laravel 11 Sanctum включён по умолчанию в стартовом шаблоне. Для более ранних версий требуется установка вручную.

### Установка (Laravel 10 и ранее)

```bash
composer require laravel/sanctum
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"
php artisan migrate
```

### Настройка модели User

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;
    // ...
}
```

### Режим 1: API-токены (для мобильных приложений, сторонних интеграций)

Каждый токен хранится хэшированным в таблице `personal_access_tokens`. Клиент отправляет токен в заголовке `Authorization: Bearer <token>`.

```php
// Создание токена с abilities (аналог scopes)
$token = $user->createToken('mobile-app', ['posts:read', 'posts:write']);

// plainTextToken -- отдаём пользователю ОДИН РАЗ, повторно получить нельзя
return response()->json([
    'token' => $token->plainTextToken,
    'type'  => 'Bearer',
]);
```

```php
// Аутентификация в контроллере
class AuthController extends Controller
{
    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email'    => 'required|email',
            'password' => 'required',
        ]);

        if (! Auth::attempt($credentials)) {
            throw ValidationException::withMessages([
                'email' => ['The provided credentials are incorrect.'],
            ]);
        }

        $user = Auth::user();

        // Удалить старые токены (опционально)
        $user->tokens()->delete();

        return response()->json([
            'user'  => new UserResource($user),
            'token' => $user->createToken('api', ['*'])->plainTextToken,
        ]);
    }

    public function logout(Request $request)
    {
        // Удалить текущий токен
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Logged out']);
    }
}
```

### Защита маршрутов и проверка abilities

```php
// Маршруты, требующие аутентификации
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/user', fn(Request $r) => new UserResource($r->user()));
    Route::get('/posts', [PostController::class, 'index']);

    // Маршрут, требующий конкретной ability
    Route::middleware('ability:posts:write')->group(function () {
        Route::post('/posts', [PostController::class, 'store']);
        Route::put('/posts/{post}', [PostController::class, 'update']);
    });
});
```

```php
// Проверка abilities в коде
if ($request->user()->tokenCan('posts:write')) {
    // Пользователь может создавать посты
}

// Множественная проверка
if ($request->user()->tokenCan('admin')) {
    // ...
}
```

### Режим 2: SPA-аутентификация (cookie-based)

Для SPA на том же домене (или поддомене) Sanctum использует стандартные сессии Laravel через cookie. Это безопаснее токенов, так как cookie автоматически привязывается к домену и защищён от JavaScript (httpOnly).

```php
// Middleware в Kernel (api group) -- Laravel 10
'api' => [
    \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
    'throttle:api',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```

```php
// config/sanctum.php -- указать домены SPA
'stateful' => explode(',', env('SANCTUM_STATEFUL_DOMAINS', 'localhost,127.0.0.1')),
```

```javascript
// Фронтенд (JavaScript) -- сначала получаем CSRF-cookie
await fetch('/sanctum/csrf-cookie', { credentials: 'include' });

// Затем логинимся как обычно
await fetch('/login', {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/json', 'Accept': 'application/json' },
    body: JSON.stringify({ email, password }),
});

// Последующие запросы автоматически аутентифицированы через cookie
const response = await fetch('/api/user', { credentials: 'include' });
```

### Настройка TTL токенов

```php
// config/sanctum.php
'expiration' => 60 * 24, // Токен истекает через 24 часа (null = бессрочно)
```

### Управление токенами

```php
// Получить все токены пользователя
$user->tokens;

// Удалить конкретный токен
$user->tokens()->where('id', $tokenId)->delete();

// Удалить все токены (logout everywhere)
$user->tokens()->delete();

// Удалить текущий токен
$request->user()->currentAccessToken()->delete();
```

### Практические советы

Для SPA на одном домене с backend используйте cookie-режим -- он безопаснее и не требует хранения токенов на клиенте. Для мобильных приложений и сторонних интеграций используйте API-токены с минимальными abilities. Всегда устанавливайте `expiration` для токенов в продакшене. Периодически очищайте устаревшие токены: `$user->tokens()->where('last_used_at', '<', now()->subMonths(3))->delete()`. Sanctum не подходит, если вам нужна OAuth2-авторизация для сторонних приложений -- для этого используйте Laravel Passport.

## Примеры

1. `$user->createToken('api')->plainTextToken`.
2. Middleware: `auth:sanctum`.
3. SPA режим через cookie + CSRF.

## Доп. теория

1. Токены имеют abilities для ограничений.
2. SPA‑режим использует cookie + CSRF.
