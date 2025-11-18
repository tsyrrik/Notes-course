# Коллекции (Collections)

Простыми словами: Коллекции дают удобные методы map/filter/reduce поверх массивов.
Обёртка над массивом с методами `map`, `filter`, `reduce`, `pluck` и др. Eloquent возвращает коллекции.
```php
$users = User::all(); // Collection
$names = $users->pluck('name');
$activeAdmins = $users
    ->where('active', true)
    ->filter(fn($u) => $u->isAdmin())
    ->map(fn($u) => $u->name)
    ->values();
```
Разница: `get()` → коллекция; `first()` → одна модель или null; `pluck()` → коллекция значений.
