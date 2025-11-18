# declare(strict_types)
`declare(strict_types=1);` в начале файла включает строгие проверки типов аргументов/возвратов.
Без директивы PHP пытается приводить типы.
Дополнительно: `strict_properties` (PHP 8.3+) для строгих типов свойств.

Пример: TypeError без неявного приведения
```php
declare(strict_types=1);
function sum(int $a, int $b): int { return $a + $b; }
sum(1, "2"); // TypeError: Argument 2 must be of type int, string given
```
