## Вопрос: Query Scopes
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Scope — сохранённый фильтр запроса, чтобы не повторять where/joins в каждом месте.

## Ответ

Scopes (скоупы) — это механизм Eloquent для инкапсуляции часто используемых условий запросов в именованные, переиспользуемые методы. Вместо того чтобы повторять одни и те же `where()`, `join()` или `orderBy()` в разных местах кода, вы определяете scope один раз в модели и используете его через цепочку вызовов. Это улучшает читаемость кода и устраняет дублирование.

**Local Scopes (локальные скоупы)** — вызываются явно при построении запроса. Определяются как методы модели с префиксом `scope`, но вызываются без него:

```php
class User extends Model
{
    // Простой scope без параметров
    public function scopeActive(Builder $query): Builder
    {
        return $query->where('active', true);
    }

    // Scope с параметрами
    public function scopeByRole(Builder $query, string $role): Builder
    {
        return $query->where('role', $role);
    }

    // Scope с несколькими условиями
    public function scopeRecentlyVerified(Builder $query, int $days = 7): Builder
    {
        return $query->whereNotNull('email_verified_at')
                     ->where('email_verified_at', '>=', now()->subDays($days));
    }

    // Scope с JOIN
    public function scopeWithActiveSubscription(Builder $query): Builder
    {
        return $query->whereHas('subscription', function ($q) {
            $q->where('expires_at', '>', now());
        });
    }
}

// Использование — scopes отлично комбинируются в цепочку
$admins = User::active()->byRole('admin')->get();
$recent = User::active()->recentlyVerified(30)->paginate(20);
$premium = User::active()->withActiveSubscription()->count();
```

## Global Scope

**Global Scopes (глобальные скоупы)** автоматически применяются ко всем запросам модели. Классический пример — `SoftDeletes`, который автоматически добавляет `WHERE deleted_at IS NULL` к каждому запросу.

```php
// 1. Через класс, реализующий интерфейс Scope
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Scope;

class TenantScope implements Scope
{
    public function apply(Builder $builder, Model $model): void
    {
        $builder->where('tenant_id', auth()->user()?->tenant_id);
    }
}

class Order extends Model
{
    protected static function booted(): void
    {
        static::addGlobalScope(new TenantScope);
    }
}

// 2. Через анонимную функцию (для простых случаев)
class Post extends Model
{
    protected static function booted(): void
    {
        static::addGlobalScope('published', function (Builder $builder) {
            $builder->where('published', true);
        });
    }
}

// 3. Через атрибут (Laravel 11)
use Illuminate\Database\Eloquent\Attributes\ScopedBy;

#[ScopedBy(TenantScope::class)]
class Order extends Model
{
    // ...
}
```

**Снятие Global Scope** — иногда нужно выполнить запрос без глобального фильтра:

```php
// Снять конкретный scope
Order::withoutGlobalScope(TenantScope::class)->get();
Post::withoutGlobalScope('published')->get();

// Снять все global scopes
Order::withoutGlobalScopes()->get();

// Снять несколько конкретных
Order::withoutGlobalScopes([TenantScope::class, AnotherScope::class])->get();
```

**Практический совет:** используйте local scopes для переиспользуемых фильтров (статусы, роли, даты). Global scopes подходят для multi-tenancy, soft deletes и любых условий, которые должны применяться всегда. Будьте осторожны с global scopes — они могут вызвать неожиданное поведение, если разработчик забудет о них. Всегда документируйте наличие global scopes в модели. Тип-хинт `Builder $query` в сигнатуре scope повышает поддержку IDE и читаемость кода.

## Примеры

1. Локальный scope: `scopeActive($q)` и вызов `User::active()`.
2. Глобальный scope: SoftDeletes.
3. Отключение scope: `withoutGlobalScope(SoftDeletingScope::class)`.

## Доп. теория

1. Local scopes читаемы и переиспользуемы.
2. Global scopes можно отключать для админ‑запросов.
