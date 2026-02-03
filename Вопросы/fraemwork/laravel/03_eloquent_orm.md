## Вопрос: Eloquent ORM: что это
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Eloquent — ORM ActiveRecord, таблицы становятся классами, строки — объектами, связи описываются методами.

## Ответ

Eloquent ORM — это реализация паттерна ActiveRecord в Laravel. Каждая таблица в базе данных представлена классом-моделью, а каждая строка таблицы — экземпляром этой модели. Eloquent предоставляет удобный, выразительный синтаксис для выполнения CRUD-операций, работы со связями, фильтрацией и трансформацией данных.

**Основы модели.** По умолчанию Eloquent ожидает, что таблица называется во множественном числе (snake_case) от имени класса: модель `User` -> таблица `users`, `OrderItem` -> `order_items`. Первичный ключ — `id`, автоинкремент. Модель автоматически управляет полями `created_at` и `updated_at`.

```php
class User extends Model
{
    protected $table = 'users';              // явное указание таблицы (необязательно)
    protected $primaryKey = 'id';            // можно изменить
    protected $fillable = ['name', 'email', 'password']; // mass assignment protection
    protected $guarded = [];                 // альтернатива: разрешить всё (осторожно!)
    protected $hidden = ['password', 'remember_token'];  // скрыть при сериализации
    protected $appends = ['full_name'];      // добавить виртуальные атрибуты в JSON

    // Casts — приведение типов (Laravel 11 рекомендует метод casts())
    protected function casts(): array
    {
        return [
            'email_verified_at' => 'datetime',
            'is_admin' => 'boolean',
            'settings' => 'array',          // JSON-поле в массив
            'password' => 'hashed',         // авто-хэширование (Laravel 10+)
        ];
    }
}
```

**CRUD-операции:**

```php
// Create
$user = User::create(['name' => 'John', 'email' => 'john@example.com', 'password' => '123']);
$user = User::firstOrCreate(['email' => 'john@example.com'], ['name' => 'John']);
$user = User::updateOrCreate(['email' => 'john@example.com'], ['name' => 'Updated John']);

// Read
$user = User::find(1);                    // по ID, вернёт null если не найдено
$user = User::findOrFail(1);              // бросит ModelNotFoundException (404)
$users = User::where('active', true)->orderBy('name')->get();
$user = User::where('email', 'john@example.com')->first();

// Update
$user->update(['name' => 'Jane']);
User::where('active', false)->update(['deleted_reason' => 'inactive']);

// Delete
$user->delete();
User::destroy([1, 2, 3]);                 // массовое удаление по ID
```

**Soft Deletes** — логическое удаление (запись остаётся в БД с заполненным полем `deleted_at`):

```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;
}

$post->delete();              // установит deleted_at
$post->restore();             // восстановит
$post->forceDelete();         // физическое удаление
Post::withTrashed()->get();   // включая удалённые
Post::onlyTrashed()->get();   // только удалённые
```

**События модели** позволяют реагировать на жизненный цикл модели: `creating`, `created`, `updating`, `updated`, `saving`, `saved`, `deleting`, `deleted`, `restoring`, `restored`. Их можно определять через Observer:

```php
// php artisan make:observer UserObserver --model=User
class UserObserver
{
    public function creating(User $user): void
    {
        $user->slug = Str::slug($user->name);
    }

    public function deleted(User $user): void
    {
        $user->posts()->delete(); // каскадное удаление
    }
}

// Регистрация в AppServiceProvider (Laravel 11 делает это автоматически через атрибуты)
```

## Типы связей
Eloquent поддерживает все основные типы связей между моделями:

- **hasOne / belongsTo** — один к одному
- **hasMany / belongsTo** — один ко многим
- **belongsToMany** — многие ко многим (через pivot-таблицу)
- **hasOneThrough / hasManyThrough** — связь через промежуточную модель
- **morphOne / morphMany / morphToMany** — полиморфные связи

```php
class User extends Model
{
    public function profile(): HasOne
    {
        return $this->hasOne(Profile::class);
    }

    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }

    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class)
            ->withPivot('assigned_at')     // дополнительные поля в pivot
            ->withTimestamps();            // timestamps в pivot-таблице
    }

    // Полиморфная связь — User может иметь комментарии, как и Post
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }

    // Связь через промежуточную таблицу
    public function country(): HasOneThrough
    {
        return $this->hasOneThrough(Country::class, City::class);
    }
}
```

**Практический совет:** всегда указывайте `$fillable` для защиты от mass assignment. Используйте `casts()` для автоматического приведения типов. Для сложных запросов комбинируйте Eloquent со scopes, а не пишите «сырой» SQL. В Laravel 11 можно использовать атрибут `#[ObservedBy]` для автоматической регистрации Observer.

## Примеры

1. `User::query()->where('active', 1)->get()`.
2. Связь: `$user->posts` через `hasMany`.
3. Массовое присваивание: `$fillable`/`$guarded`.

## Доп. теория

1. Eloquent — Active Record, каждая модель связана с таблицей.
2. Ленивая загрузка может приводить к N+1.
