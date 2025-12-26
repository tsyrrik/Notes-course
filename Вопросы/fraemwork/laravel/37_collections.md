## Вопрос: Коллекции (Collections)
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Коллекции дают удобные методы map/filter/reduce поверх массивов.

## Ответ
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
