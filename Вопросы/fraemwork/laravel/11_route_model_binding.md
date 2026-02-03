## Вопрос: Route Model Binding
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Route Model Binding сам находит модель по параметру URL и подставляет в контроллер.

## Ответ

Route Model Binding — это механизм Laravel, который автоматически находит модель Eloquent по параметру маршрута и внедряет её в контроллер. Вместо того чтобы вручную писать `User::findOrFail($id)` в каждом методе, Laravel делает это за вас. Если модель не найдена, автоматически возвращается ответ 404.

**Implicit Binding (неявная привязка)** — самый распространённый способ. Laravel определяет модель по type-hint параметра в методе контроллера. Имя параметра маршрута `{user}` должно совпадать с именем переменной `$user`:

```php
// routes/web.php
Route::get('/users/{user}', [UserController::class, 'show']);
Route::get('/users/{user}/edit', [UserController::class, 'edit']);
Route::put('/users/{user}', [UserController::class, 'update']);

// UserController.php — Laravel сам найдёт User::findOrFail($id)
public function show(User $user)
{
    return view('users.show', compact('user'));
}
```

**Кастомный ключ поиска.** По умолчанию Laravel ищет модель по `id`. Можно изменить ключ:

```php
// Способ 1: глобально для модели — переопределить getRouteKeyName()
class Post extends Model
{
    public function getRouteKeyName(): string
    {
        return 'slug'; // теперь /posts/{post} ищет по slug
    }
}

// Способ 2: для конкретного маршрута (Laravel 9+)
Route::get('/posts/{post:slug}', [PostController::class, 'show']);

// Способ 3: разные ключи для разных маршрутов
Route::get('/posts/{post:slug}', [PostController::class, 'show']);    // по slug
Route::get('/admin/posts/{post}', [AdminPostController::class, 'edit']); // по id
```

**Scoped Binding (привязка с ограничением по родителю):**

```php
// Laravel автоматически ограничит комментарий принадлежностью к посту
Route::get('/posts/{post}/comments/{comment}', function (Post $post, Comment $comment) {
    // $comment точно принадлежит $post, иначе 404
    return $comment;
});

// Явное включение/отключение scoping
Route::get('/users/{user}/posts/{post}', function (User $user, Post $post) {
    //
})->scopeBindings();   // включить

Route::get('/users/{user}/posts/{post:slug}', function (User $user, Post $post) {
    //
})->withoutScopedBindings();  // отключить
```

**Explicit Binding (явная привязка)** — регистрируется в `boot()` метода `AppServiceProvider` (или `RouteServiceProvider` в Laravel 10):

```php
// AppServiceProvider.php (Laravel 11)
use Illuminate\Support\Facades\Route;

public function boot(): void
{
    // Простая привязка модели
    Route::model('user', User::class);

    // Кастомная логика поиска
    Route::bind('user', function (string $value) {
        return User::where('slug', $value)
            ->where('active', true)
            ->firstOrFail();
    });

    // Привязка с учётом soft deletes
    Route::bind('post', function (string $value) {
        return Post::withTrashed()->findOrFail($value);
    });
}
```

**Кастомизация поведения при отсутствии модели:**

```php
// В модели — вернуть значение по умолчанию вместо 404
Route::get('/users/{user}', [UserController::class, 'show'])
    ->missing(function (Request $request) {
        return redirect()->route('users.index')
            ->with('error', 'Пользователь не найден');
    });
```

**Enum Binding (Laravel 11)** — можно привязывать PHP Enum к параметрам маршрута:

```php
enum UserStatus: string
{
    case Active = 'active';
    case Inactive = 'inactive';
}

Route::get('/users/status/{status}', function (UserStatus $status) {
    return User::where('status', $status)->get();
});
// GET /users/status/active — работает
// GET /users/status/unknown — 404
```

**Практический совет:** используйте implicit binding везде, где это возможно — это стандарт Laravel и сокращает код. Для публичных URL используйте `{post:slug}` вместо числовых ID — это улучшает SEO и безопасность. Scoped bindings (вложенные ресурсы) избавляют от ручной проверки принадлежности. Для API всегда учитывайте, что binding возвращает 404 без сообщения — кастомизируйте через `->missing()`.

## Примеры

1. `Route::get('/users/{user}', fn (User $user) => ...)`.
2. Переопределение ключа: `getRouteKeyName(): 'slug'`.
3. Явная привязка в `RouteServiceProvider::boot()`.

## Доп. теория

1. Если модель не найдена — 404 автоматически.
2. Scoped bindings защищают от доступа к чужим данным.
