## Вопрос: Кэширование
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Кэш сохраняет результаты/данные в быструю память (файл/Redis), чтобы меньше ходить в БД.

## Ответ

Кэширование в Laravel -- это механизм сохранения результатов вычислений или запросов к БД в быстрое хранилище, чтобы при повторных обращениях не выполнять одну и ту же работу. Laravel предоставляет единый API (facade `Cache`) для работы с различными драйверами: `file`, `redis`, `memcached`, `dynamodb`, `database`, `array` (для тестов). Настройка находится в `config/cache.php`, текущий драйвер задаётся через `CACHE_STORE` в `.env`.

Для продакшена рекомендуется Redis или Memcached -- они хранят данные в оперативной памяти и работают на порядки быстрее файлового кэша. Redis дополнительно поддерживает tags, atomic locks и другие продвинутые возможности. File-кэш подходит для небольших проектов или разработки.

### Базовые операции

```php
use Illuminate\Support\Facades\Cache;

// Записать значение на 60 секунд
Cache::put('key', 'value', 60);

// Записать навсегда
Cache::forever('settings', $settings);

// Получить с дефолтом
$value = Cache::get('key', 'default');

// Проверить наличие
if (Cache::has('key')) { /* ... */ }

// Получить и удалить
$value = Cache::pull('key');

// Удалить
Cache::forget('key');

// Очистить весь кэш
Cache::flush();
```

### remember и rememberForever

Самый частый паттерн -- `remember`: если ключ есть в кэше, вернуть его; если нет, выполнить callback, сохранить результат и вернуть:

```php
// Кэшируем список активных пользователей на 1 час
$users = Cache::remember('active_users', 3600, function () {
    return User::where('active', true)
        ->orderBy('name')
        ->get();
});

// Кэширование конкретной записи
$user = Cache::remember("user:{$id}", 600, fn() => User::findOrFail($id));

// Кэш навсегда (до ручной инвалидации)
$settings = Cache::rememberForever('app_settings', fn() => Setting::all()->pluck('value', 'key'));
```

### Cache Tags (только Redis / Memcached)

Tags позволяют группировать записи и очищать их группой:

```php
// Сохраняем с тегами
Cache::tags(['users', 'profiles'])->put("user:{$id}", $user, 3600);
Cache::tags(['users', 'posts'])->put("user:{$id}:posts", $posts, 3600);

// Очистить все записи с тегом 'users'
Cache::tags(['users'])->flush();

// Очистить конкретный тег
Cache::tags(['profiles'])->flush();
```

### Atomic Locks (Redis)

Атомарные блокировки предотвращают гонки (race conditions):

```php
$lock = Cache::lock('processing-order-' . $orderId, 10); // блокировка на 10 секунд

if ($lock->get()) {
    try {
        // Критическая секция -- только один процесс одновременно
        processOrder($orderId);
    } finally {
        $lock->release();
    }
}

// Или с блокирующим ожиданием
Cache::lock('report-generation', 30)->block(5, function () {
    // Ждём до 5 секунд, если лок занят
    generateReport();
});
```

### Инвалидация кэша

Одна из главных проблем кэширования -- своевременная инвалидация. Типичные стратегии:

```php
// 1. TTL (время жизни) -- кэш сам истекает
Cache::put('stats', $stats, now()->addMinutes(30));

// 2. Ручная инвалидация при изменении данных
class UserObserver
{
    public function updated(User $user): void
    {
        Cache::forget("user:{$user->id}");
        Cache::tags(['users'])->flush();
    }
}

// 3. Версионирование ключей
$version = Cache::increment('users_cache_version');
$users = Cache::remember("users:v{$version}", 3600, fn() => User::all());
```

### Кэширование на уровне фреймворка

```bash
# Кэширование конфигов (объединяет все конфиги в один файл)
php artisan config:cache

# Кэширование маршрутов (ускоряет поиск маршрутов)
php artisan route:cache

# Кэширование views (компилирует Blade-шаблоны)
php artisan view:cache

# Кэширование events (Laravel 11)
php artisan event:cache

# Очистка всего
php artisan optimize:clear
```

### HTTP Cache через middleware

```php
// В маршрутах -- заголовки кэширования для браузера/CDN
Route::get('/api/catalog', [CatalogController::class, 'index'])
    ->middleware('cache.headers:public;max_age=600;etag');
```

### Практические советы

Не кэшируйте данные, которые часто меняются -- стоимость инвалидации перевесит выигрыш. Используйте осмысленные ключи с префиксами (`user:42:profile`, `shop:products:popular`). Для больших проектов применяйте cache tags, чтобы точечно инвалидировать группы. Всегда задавайте TTL -- "вечный" кэш без стратегии инвалидации рано или поздно приведёт к показу устаревших данных. В тестах используйте драйвер `array`, чтобы кэш не влиял на другие тесты. Помните, что `config:cache` игнорирует `.env` -- после кэширования конфига `env()` работает только внутри конфиг-файлов.

## Примеры

1. `Cache::remember('users', 3600, fn () => User::all())`.
2. Теги: `Cache::tags(['users'])->flush()`.
3. Очистка: `php artisan cache:clear`.

## Доп. теория

1. Выбор драйвера влияет на скорость и функции.
2. Теги поддерживаются не во всех драйверах.
