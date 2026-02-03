## Вопрос: Mutators и Accessors
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Accessor форматирует поле при чтении, Mutator меняет значение перед сохранением.

## Ответ

Accessors и Mutators — это механизм Eloquent для автоматического преобразования значений атрибутов модели при чтении (accessor) и записи (mutator). Они позволяют инкапсулировать логику форматирования и нормализации данных внутри модели, обеспечивая единообразие представления данных во всём приложении.

**Accessor** вызывается при обращении к атрибуту модели (`$user->name`) и позволяет трансформировать значение из базы данных перед возвратом. **Mutator** вызывается при присвоении значения (`$user->name = 'john'`) и преобразует данные перед сохранением в базу.

**Старый синтаксис (до Laravel 9)** — методы `get{Attribute}Attribute` и `set{Attribute}Attribute`:

```php
class User extends Model
{
    // Accessor — вызывается при чтении $user->name
    public function getNameAttribute(string $value): string
    {
        return ucfirst($value);
    }

    // Mutator — вызывается при записи $user->password = '...'
    public function setPasswordAttribute(string $value): void
    {
        $this->attributes['password'] = bcrypt($value);
    }

    // Accessor для виртуального атрибута (не существует в БД)
    public function getFullNameAttribute(): string
    {
        return "{$this->first_name} {$this->last_name}";
    }
}
```

**Новый синтаксис (Laravel 9+ / 11) — рекомендуемый.** Метод `Attribute::make()` объединяет accessor и mutator в одном месте:

```php
use Illuminate\Database\Eloquent\Casts\Attribute;

class User extends Model
{
    // Accessor + Mutator для одного поля
    protected function name(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => ucfirst($value),
            set: fn (string $value) => strtolower($value),
        );
    }

    // Виртуальный атрибут (только accessor)
    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn (mixed $value, array $attributes) =>
                $attributes['first_name'] . ' ' . $attributes['last_name'],
        );
    }

    // Mutator с множественным присвоением
    protected function address(): Attribute
    {
        return Attribute::make(
            get: fn (mixed $value, array $attributes) => new Address(
                street: $attributes['address_street'],
                city: $attributes['address_city'],
            ),
            set: fn (Address $value) => [
                'address_street' => $value->street,
                'address_city' => $value->city,
            ],
        );
    }

    // Accessor с кэшированием — результат вычисляется один раз
    protected function hash(): Attribute
    {
        return Attribute::make(
            get: fn (string $value) => md5($value),
        )->shouldCache();
    }

    // Атрибут без кэширования (по умолчанию новый синтаксис кэширует)
    protected function currentTime(): Attribute
    {
        return Attribute::make(
            get: fn () => now()->toDateTimeString(),
        )->withoutObjectCaching();
    }
}
```

**Чтобы виртуальный атрибут попадал в JSON/массив**, добавьте его в `$appends`:

```php
class User extends Model
{
    protected $appends = ['full_name'];

    protected function fullName(): Attribute
    {
        return Attribute::make(
            get: fn (mixed $value, array $attributes) =>
                $attributes['first_name'] . ' ' . $attributes['last_name'],
        );
    }
}

// Теперь $user->toArray() и $user->toJson() включают full_name
```

**Casts vs Accessors/Mutators.** В Laravel 11 для стандартных преобразований типов (дата, boolean, JSON, enum) предпочтительнее использовать `casts()`, а не аксессоры. Casts проще и оптимизированы для типичных сценариев. Accessor/Mutator нужны для кастомной логики:

```php
protected function casts(): array
{
    return [
        'is_admin' => 'boolean',       // вместо accessor
        'settings' => 'array',          // JSON -> array
        'status' => UserStatus::class,  // Enum casting (PHP 8.1+)
        'password' => 'hashed',         // авто-хэширование
    ];
}
```

**Практический совет:** в новых проектах используйте только новый синтаксис `Attribute::make()`. Помните, что accessor/mutator вызывается при каждом обращении к атрибуту, поэтому избегайте тяжёлых вычислений — используйте `shouldCache()` или выносите логику в отдельные методы. Второй аргумент `$attributes` в callback даёт доступ ко всем атрибутам модели, что удобно для виртуальных полей.

## Примеры

1. Accessor: `getFullNameAttribute()`.
2. Mutator: `setPasswordAttribute()` с `bcrypt`.
3. Casts: `$casts = ['is_admin' => 'boolean']`.

## Доп. теория

1. Accessors влияют на сериализацию и `toArray()`.
2. Mutators применяются при присвоении атрибутов.
