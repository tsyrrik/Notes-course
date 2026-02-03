## Вопрос: Query Builder
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Query Builder строит SQL методами PHP, чтобы не писать сырой SQL и не забывать экранировать параметры.

## Ответ

Query Builder — это объектно-ориентированный интерфейс для построения и выполнения SQL-запросов в Laravel. Он предоставляет fluent API для формирования запросов через цепочку методов, автоматически экранирует параметры (защита от SQL-инъекций) и работает со всеми поддерживаемыми СУБД (MySQL, PostgreSQL, SQLite, SQL Server). Query Builder является основой, на которой построен Eloquent ORM.

**Ключевое отличие от Eloquent:** Query Builder возвращает объекты `stdClass` (или коллекции из них), а не модели. Это означает, что вы не получите аксессоры, мутаторы, связи и события модели. Но Query Builder работает быстрее, потому что не создаёт экземпляры моделей, и лучше подходит для сложных SQL-запросов, агрегаций и отчётов.

```php
use Illuminate\Support\Facades\DB;

// Базовые выборки
$users = DB::table('users')->get();                           // все записи
$user = DB::table('users')->where('id', 1)->first();          // одна запись
$email = DB::table('users')->where('id', 1)->value('email');  // одно значение
$names = DB::table('users')->pluck('name', 'id');             // коллекция ключ => значение

// Условия WHERE
$users = DB::table('users')
    ->where('age', '>', 18)
    ->where('active', true)                    // AND
    ->orWhere('role', 'admin')                 // OR
    ->whereIn('status', ['active', 'pending']) // IN
    ->whereNotNull('email_verified_at')
    ->whereBetween('created_at', [$from, $to])
    ->whereDate('created_at', '2024-01-01')
    ->get();

// Группировка условий (вложенные WHERE)
$users = DB::table('users')
    ->where('active', true)
    ->where(function ($query) {
        $query->where('role', 'admin')
              ->orWhere('role', 'moderator');
    })
    ->get();
// SQL: SELECT * FROM users WHERE active = 1 AND (role = 'admin' OR role = 'moderator')
```

**Агрегация и группировка:**

```php
$count = DB::table('users')->where('active', true)->count();
$maxPrice = DB::table('products')->max('price');
$avgAge = DB::table('users')->avg('age');

$stats = DB::table('orders')
    ->select('status', DB::raw('COUNT(*) as count'), DB::raw('SUM(total) as revenue'))
    ->groupBy('status')
    ->having('count', '>', 10)
    ->get();
```

**JOIN-запросы:**

```php
$results = DB::table('users')
    ->join('orders', 'users.id', '=', 'orders.user_id')
    ->leftJoin('profiles', 'users.id', '=', 'profiles.user_id')
    ->select('users.name', 'orders.total', 'profiles.avatar')
    ->get();

// Подзапросы
$latestPosts = DB::table('posts')
    ->select('user_id', DB::raw('MAX(created_at) as last_post'))
    ->groupBy('user_id');

$users = DB::table('users')
    ->joinSub($latestPosts, 'latest_posts', function ($join) {
        $join->on('users.id', '=', 'latest_posts.user_id');
    })
    ->get();
```

**Insert, Update, Delete:**

```php
DB::table('users')->insert(['name' => 'John', 'email' => 'john@example.com']);
DB::table('users')->insertOrIgnore([...]);                  // игнорировать дубликаты
DB::table('users')->upsert(                                  // insert or update
    [['email' => 'john@example.com', 'name' => 'John']],    // данные
    ['email'],                                                // уникальные ключи
    ['name']                                                  // поля для обновления
);

DB::table('users')->where('id', 1)->update(['name' => 'Jane']);
DB::table('users')->where('id', 1)->increment('login_count', 1);
DB::table('users')->where('active', false)->delete();
```

**Chunking для больших объёмов данных:**

```php
// Обработка миллионов записей без исчерпания памяти
DB::table('users')->orderBy('id')->chunk(1000, function ($users) {
    foreach ($users as $user) {
        // обработка
    }
});

// Lazy collection (Laravel 8+) — ещё удобнее
DB::table('users')->orderBy('id')->lazy()->each(function ($user) {
    // обработка по одной записи
});
```

**Практический совет:** используйте Query Builder для отчётов, агрегаций и массовых операций, где модели Eloquent создают ненужный overhead. Для обычного CRUD предпочтительнее Eloquent. Метод `DB::table()` можно комбинировать с Eloquent: `User::query()->toBase()` переключает на Query Builder. Всегда используйте параметризированные запросы (`?` или bindings), а не конкатенацию строк в `DB::raw()`.

## Примеры

1. `DB::table('users')->where('active', 1)->get()`.
2. JOIN: `->join('posts', 'users.id', '=', 'posts.user_id')`.
3. Транзакция: `DB::transaction(fn () => ...)`.

## Доп. теория

1. Query Builder возвращает массивы, а не модели.
2. Параметры всегда биндятся — защита от SQL‑инъекций.
