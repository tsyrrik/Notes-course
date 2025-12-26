## Вопрос: Eager Loading и проблема N+1
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
- `with()`/`load()` подготавливают связи одним-двумя запросами, чтобы не стрелять SQL в цикле по каждой строке (решаем N+1).
```php
// Плохо (N+1)
$users = User::all();
foreach ($users as $user) echo $user->profile->bio;

// Хорошо
$users = User::with('profile')->get();

// Условная загрузка
User::with(['posts' => fn($q) => $q->where('published', true)])->get();

// Lazy eager loading
$users = User::all();
$users->load('profile');
```

## Ответ
- N+1: при доступе к связям в цикле делает N доп. запросов.
- Eager loading загружает связи заранее.
