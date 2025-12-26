## Вопрос: Query Scopes
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Scope — сохранённый фильтр запроса, чтобы не повторять where/joins в каждом месте.

## Ответ
Переиспользуемые фильтры в моделях.
```php
class User extends Model {
    public function scopeActive($q) { return $q->where('active', 1); }
    public function scopeByRole($q, $role) { return $q->where('role', $role); }
}

$admins = User::active()->byRole('admin')->get();
```

## Global Scope
```php
class ActiveScope implements Scope {
    public function apply(Builder $builder, Model $model) {
        $builder->where('active', 1);
    }
}

class User extends Model {
    protected static function booted() { static::addGlobalScope(new ActiveScope); }
}
```
