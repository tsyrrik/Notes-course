## Вопрос: Eager Loading и проблема N+1
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

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

**Проблема N+1** — одна из самых частых причин медленной работы приложений. Она возникает, когда при переборе коллекции моделей каждое обращение к связанной модели генерирует отдельный SQL-запрос. Если у вас 100 пользователей и вы обращаетесь к `$user->profile` в цикле, это создаёт 1 запрос (получить пользователей) + 100 запросов (получить профиль каждого) = 101 запрос вместо 2.

**Eager Loading (жадная загрузка)** решает эту проблему, загружая связанные модели заранее одним или двумя SQL-запросами. Laravel использует стратегию «загрузить все ID, затем WHERE IN» — это эффективно и предсказуемо.

**Типы Eager Loading:**

```php
// 1. Eager loading при выборке — with()
// Выполнит 2 запроса: SELECT * FROM users; SELECT * FROM profiles WHERE user_id IN (1,2,3...)
$users = User::with('profile')->get();

// 2. Множественные связи
$users = User::with(['profile', 'posts', 'roles'])->get();

// 3. Вложенные связи (nested)
$users = User::with('posts.comments.author')->get();

// 4. Условная загрузка — фильтрация связанных моделей
$users = User::with(['posts' => function ($query) {
    $query->where('published', true)
          ->orderBy('created_at', 'desc')
          ->limit(5);
}])->get();

// 5. Lazy Eager Loading — загрузка после получения коллекции
$users = User::all();
$users->load('profile');                    // загрузить связь
$users->loadMissing('profile');             // загрузить только если ещё не загружена

// 6. Eager loading с подсчётом (withCount)
$users = User::withCount('posts')->get();
// Добавляет атрибут posts_count без загрузки самих постов
echo $users->first()->posts_count;

// 7. Условный подсчёт
$users = User::withCount([
    'posts',
    'posts as published_posts_count' => fn($q) => $q->where('published', true),
])->get();

// 8. withSum, withAvg, withMin, withMax (Laravel 8+)
$authors = User::withSum('posts', 'views')->get();
echo $authors->first()->posts_sum_views;
```

**Предотвращение N+1 на уровне модели:**

```php
class User extends Model
{
    // Всегда загружать эти связи (осторожно — может быть избыточно)
    protected $with = ['profile'];
}
```

**Обнаружение N+1 проблем** — в Laravel 11 (и 9+) можно включить строгий режим, который бросает исключение при lazy loading:

```php
// В AppServiceProvider::boot()
Model::preventLazyLoading(! app()->isProduction());

// В production можно логировать вместо исключения
Model::handleLazyLoadingViolationUsing(function ($model, $relation) {
    logger()->warning("Lazy loading {$relation} on {$model::class}");
});
```

**Практический совет:** используйте пакет **Laravel Debugbar** или **Telescope** для отслеживания количества SQL-запросов на странице. Включайте `preventLazyLoading()` в development-окружении, чтобы ловить N+1 на этапе разработки. Метод `loadMissing()` предпочтительнее `load()`, когда связь может быть уже загружена — он не делает повторный запрос.

## Примеры

1. `User::with('posts')->get()`.
2. `loadMissing('profile')` для уже загруженной модели.
3. Фильтр связей: `with(['posts' => fn($q) => $q->where('published', 1)])`.

## Доп. теория

1. Eager loading снижает количество запросов.
2. Связи можно ограничивать условиями внутри `with()`.
