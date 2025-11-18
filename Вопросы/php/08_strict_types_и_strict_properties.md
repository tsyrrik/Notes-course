# declare(strict_types)
Простыми словами: `declare(strict_types=1);` заставляет PHP выбрасывать TypeError вместо тихого приведения аргументов/возвратов. По умолчанию PHP пытается конвертировать типы.
Также: `strict_properties` (PHP 8.3+) — строгая типизация свойств (опция, пока RFC/эксперимент).

Пример: TypeError без неявного приведения
```php
declare(strict_types=1);
function sum(int $a, int $b): int { return $a + $b; }
sum(1, "2"); // TypeError: Argument 2 must be of type int, string given
```
