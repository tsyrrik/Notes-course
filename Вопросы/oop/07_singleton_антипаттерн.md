# Singleton как антипаттерн
- Проблемы: глобальное состояние, скрытые зависимости, нарушение SRP/DIP, сложное тестирование, ограничения расширяемости, гонки в многопоточной среде.
- Допустим редко (ленивая инициализация, ограниченный контекст) и лучше через DI-контейнер с shared-инстансами.
```php
class Singleton {
    private static ?self $instance = null;
    private function __construct() {}
    public static function getInstance(): self {
        return self::$instance ??= new self();
    }
}
```
