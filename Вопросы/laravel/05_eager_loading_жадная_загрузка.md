# Eager Loading и проблема N+1
- N+1: при доступе к связям в цикле делает N доп. запросов.
- Eager loading загружает связи заранее.
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
