## Вопрос: IoC (Inversion of Control)

## Простой ответ
- Не объект решает, как получить зависимости, а внешняя система управляет ими и отдаёт готовые.

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

## Ответ

### Что такое IoC

Inversion of Control — принцип, при котором **управление потоком выполнения и созданием объектов передаётся внешней системе** (фреймворку, контейнеру), а не контролируется самим объектом.

«Не вы вызываете фреймворк, фреймворк вызывает вас» (Hollywood Principle: «Don't call us, we'll call you»).

### Без IoC vs с IoC

```php
// БЕЗ IoC: объект сам управляет зависимостями
class UserService {
    private DatabaseConnection $db;

    public function __construct() {
        $this->db = new MySQLConnection('localhost', 'root', 'pass'); // жёсткая связь
    }
}

// С IoC: зависимость приходит извне
class UserService {
    public function __construct(
        private DatabaseConnection $db, // не знает, кто и как её создал
    ) {}
}
```

### Реализации IoC

1. **Dependency Injection** (рекомендуемая) — зависимости передаются в конструктор/метод
2. **Service Locator** — объект запрашивает зависимости из контейнера (нежелательно)
3. **Event-driven** — фреймворк вызывает ваш код через события/хуки
4. **Template Method** — базовый класс управляет потоком, наследник реализует шаги

### IoC-контейнер

IoC-контейнер — это автоматизированная фабрика, которая:
1. Хранит регистрации (bindings): интерфейс → реализация
2. Автоматически разрешает зависимости через Reflection
3. Управляет временем жизни: transient (каждый раз новый), singleton (один на приложение), scoped (один на запрос)

```php
// Laravel Service Container
$this->app->bind(CacheInterface::class, RedisCache::class);
$this->app->singleton(Logger::class, fn() => new FileLogger('/var/log/app.log'));

// Symfony DI Container (services.yaml)
// App\Service\Mailer:
//     arguments: ['@mailer.transport']

// Автоматическое разрешение (autowiring):
// Контейнер видит конструктор, находит подходящие bindings, собирает объект
$service = $container->get(UserService::class);
```

### Преимущества

- **Низкая связанность** — классы зависят от абстракций
- **Тестируемость** — легко подставить mock
- **Гибкость** — смена реализации через конфигурацию, без правки кода
- **Управление временем жизни** — singleton, transient, scoped

## Примеры
```text
IoC: объект не создаёт зависимости сам, их предоставляет контейнер.
```

## Доп. теория
- IoC — общий принцип; DI — наиболее популярная реализация IoC.
- Важно держать зависимости явными, даже если используется контейнер.
