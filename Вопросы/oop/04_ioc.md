# IoC (Inversion of Control)
- Принцип: управление зависимостями/потоком передаем внешней системе (фреймворк/контейнер).
- DI — реализация IoC; Service Locator — тоже вариант, но нежелательный.
- Цель: снизить связанность и упростить тестирование.

```php
class Container {
    private array $services = [];
    public function set(string $id, callable $factory): void { $this->services[$id] = $factory; }
    public function get(string $id): mixed { return ($this->services[$id])($this); }
}

$c = new Container();
$c->set(Logger::class, fn() => new FileLogger('/tmp/app.log'));
$c->set(UserService::class, fn(Container $c) => new UserService($c->get(Logger::class)));

$service = $c->get(UserService::class); // извне управляем зависимостями
```
