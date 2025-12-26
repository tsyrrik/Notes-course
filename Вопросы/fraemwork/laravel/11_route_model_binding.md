## Вопрос: Route Model Binding
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Route Model Binding сам находит модель по параметру URL и подставляет в контроллер.

## Ответ
Автонaхождение модели по параметру маршрута.
```php
// Имплисит
Route::get('/users/{user}', [UserController::class, 'show']);
public function show(User $user) { return view('users.show', compact('user')); }

// Кастомный ключ
public function getRouteKeyName() { return 'slug'; }

// Explicit binding
Route::model('user', User::class);
Route::bind('user', fn($value) => User::where('slug',$value)->firstOrFail());
```
