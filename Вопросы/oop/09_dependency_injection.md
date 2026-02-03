## Вопрос: Dependency Injection и DI-контейнер

## Простой ответ
- Не создаём зависимости внутри, а принимаем готовые. Контейнер может построить и выдать их автоматически.

## Ответ
Простыми словами: объект не создаёт зависимости сам, их дают извне. Это снижает связность и упрощает тесты (можно подменять моки). DI‑контейнер автоматизирует создание и выдачу зависимостей.

### Виды внедрения

**1. Через конструктор (рекомендуемый)**

Зависимости передаются при создании объекта. Объект не может существовать без них.

```php
class OrderService {
    public function __construct(
        private readonly OrderRepository $repo,
        private readonly PaymentGateway $payment,
        private readonly LoggerInterface $logger,
    ) {}
}
```

**Преимущества**: зависимости явные, объект всегда в валидном состоянии, immutable.

**2. Через сеттер (для опциональных зависимостей)**

```php
class ReportGenerator {
    private ?CacheInterface $cache = null;

    public function setCache(CacheInterface $cache): void {
        $this->cache = $cache;
    }
}
```

**3. Через метод (method injection)**

Когда зависимость нужна только в конкретном методе:

```php
class Controller {
    public function index(Request $request, UserRepository $repo): Response {
        // $repo внедряется фреймворком только для этого метода
    }
}
```

### DI-контейнер

Автоматизирует создание объектов и разрешение зависимостей:

```php
// Регистрация
$container->bind(PaymentGateway::class, StripeGateway::class);
$container->singleton(LoggerInterface::class, fn() => new FileLogger('/var/log/app.log'));

// Разрешение — контейнер сам создаёт объект и все его зависимости
$service = $container->make(OrderService::class);
// Контейнер через Reflection анализирует конструктор:
// 1. OrderRepository нужен → создаёт/находит
// 2. PaymentGateway → берёт StripeGateway (binding)
// 3. LoggerInterface → берёт singleton FileLogger
```

### Autowiring

Современные контейнеры (Laravel, Symfony) автоматически разрешают зависимости по type-hint без явной регистрации:

```php
// Symfony: autowiring включен по умолчанию
// Если конструктор требует LoggerInterface — Symfony подставит Monolog
// Без единой строки конфигурации (если реализация одна)

// Если реализаций несколько — нужно указать явно:
// services.yaml:
// App\Service\OrderService:
//     arguments:
//         $payment: '@stripe_gateway'
```

### Тестирование с DI

```php
class OrderServiceTest extends TestCase {
    public function testPlaceOrder(): void {
        // Подставляем mock вместо реальной зависимости
        $payment = $this->createMock(PaymentGateway::class);
        $payment->expects($this->once())
            ->method('charge')
            ->with(100.0);

        $service = new OrderService(
            new InMemoryOrderRepository(),
            $payment,
            new NullLogger(),
        );

        $service->place(new Order(total: 100.0));
    }
}
```

## Примеры
```text
DI: зависимости передаются в конструктор и легко подменяются в тестах.
```

## Доп. теория
- Контейнер — это инструмент; DI работает и без него.
- Избегайте service locator в бизнес‑коде.
