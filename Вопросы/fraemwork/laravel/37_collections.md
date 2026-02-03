## Вопрос: Коллекции (Collections)
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

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

## Примеры

1. collect($users)->filter(...)->values().
2. `groupBy('status')`.
3. `Collection::macro('toIds', ...)`.

## Доп. теория

1. Коллекции неизменяемы — методы возвращают новую коллекцию.
2. LazyCollection экономит память на больших наборах.
