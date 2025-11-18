# Кэширование

Простыми словами: Кэш сохраняет результаты/данные в быструю память (файл/Redis), чтобы меньше ходить в БД.
Поддерживаются file/redis/memcached и др. (`config/cache.php`).
```php
Cache::put('key','value',60);
$value = Cache::get('key','default');
$users = Cache::remember('users', 3600, fn() => User::all());
Cache::forget('users');
// Tags (redis/memcached)
Cache::tags(['people','users'])->put('user:1',$user,60);
Cache::tags(['people'])->flush();
```

## Кэширование запросов
```php
$users = Cache::remember('active_users', 3600, fn() => User::where('active', true)->orderBy('name')->get());
$user = Cache::remember("user:$id", 600, fn() => User::findOrFail($id));
```
