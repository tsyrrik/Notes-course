## Вопрос: PSR стандарты
Версия: PHP 8.4; где есть исторические детали — см. по тексту.

## Простой ответ
это общие интерфейсы/правила, чтобы библиотеки и фреймворки были совместимы.

## Ответ

PSR (PHP Standards Recommendations) — рекомендации от PHP-FIG (Framework Interop Group), обеспечивающие совместимость между библиотеками и фреймворками. Не обязательны, но де-факто стандарт индустрии.

### Ключевые PSR

| PSR | Название | Описание |
| --- | --- | --- |
| PSR-1 | Basic Coding Standard | Файлы UTF-8, `<?php`, классы StudlyCaps, методы camelCase |
| PSR-3 | Logger Interface | `Psr\Log\LoggerInterface` — единый интерфейс логгирования (Monolog, Symfony Logger) |
| PSR-4 | Autoloading | Соответствие namespace → путь к файлу. Заменил PSR-0 |
| PSR-6 | Caching Interface | `CacheItemPoolInterface` — интерфейс кеширования (полный) |
| PSR-7 | HTTP Message | `RequestInterface`, `ResponseInterface` — неизменяемые HTTP-сообщения |
| PSR-11 | Container Interface | `ContainerInterface` — единый интерфейс DI-контейнера |
| PSR-12 | Extended Coding Style | Развитие PSR-1/PSR-2: стиль кода, отступы, скобки. Заменил PSR-2 |
| PSR-14 | Event Dispatcher | `EventDispatcherInterface` — стандарт событий |
| PSR-15 | HTTP Handlers | `RequestHandlerInterface`, `MiddlewareInterface` — стандарт middleware |
| PSR-16 | Simple Cache | `CacheInterface` — упрощённое API кеша (`get`/`set`/`delete`) |
| PSR-17 | HTTP Factories | Фабрики для создания PSR-7 объектов |
| PSR-18 | HTTP Client | `ClientInterface` — единый интерфейс HTTP-клиента |
| PSR-20 | Clock | `ClockInterface` — источник времени (тестируемость) |

### Самые важные для собеседования

**PSR-4** — понимание автозагрузки:
```json
{
  "autoload": {
    "psr-4": {
      "App\\": "src/"
    }
  }
}
```
Класс `App\Services\UserService` → файл `src/Services/UserService.php`.

**PSR-7** — неизменяемость HTTP-сообщений:
```php
// PSR-7 объекты immutable — методы возвращают новый объект
$response = $response->withStatus(200);
$response = $response->withHeader('Content-Type', 'application/json');
// Оригинал не изменён!
```

**PSR-3** — логгирование:
```php
use Psr\Log\LoggerInterface;

class OrderService {
    public function __construct(private LoggerInterface $logger) {}

    public function create(Order $order): void {
        $this->logger->info('Order created', ['id' => $order->id]);
        // Уровни: emergency, alert, critical, error, warning, notice, info, debug
    }
}
```

**PSR-11** — контейнер:
```php
use Psr\Container\ContainerInterface;

$service = $container->get(UserService::class);
$container->has(UserService::class); // bool
```

## Примеры
```php
use Psr\Log\LoggerInterface;

class OrderService {
    public function __construct(private LoggerInterface $logger) {}

    public function create(Order $order): void {
        $this->logger->info('Order created', ['id' => $order->id]);
    }
}
```

```php
// PSR-7 immutable
$response = $response->withStatus(200)
    ->withHeader('Content-Type', 'application/json');
```

## Доп. теория
- PSR‑2 и PSR‑0 **deprecated** и заменены PSR‑12 и PSR‑4 соответственно.
- Сейчас PHP‑FIG развивает PER (PHP Evolving Recommendations) для дальнейшей эволюции кодстайла.

### Зачем нужны

- **Совместимость**: Monolog реализует PSR-3, и любой фреймворк может его использовать
- **Заменяемость**: перейти с Guzzle на Symfony HttpClient — оба реализуют PSR-18
- **Стандартизация**: единые правила кода в команде (PSR-12)
