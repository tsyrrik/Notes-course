## Вопрос: Dependency Injection vs Service Locator

## Простой ответ
- DI явным образом передаёт зависимости в конструктор/метод, их видно и легко подменить.
- Service Locator прячет зависимости за глобальным контейнером, усложняет тесты и нарушает IoC.
```php
// DI
class UserService {
    public function __construct(private Mailer $mailer) {}
    public function register() { $this->mailer->send('Welcome!'); }
}
```

## Ответ

### Dependency Injection (DI)

Зависимости **передаются извне** через конструктор, сеттер или метод. Класс декларирует что ему нужно, но не знает откуда это берётся.

```php
class OrderService {
    public function __construct(
        private PaymentGateway $payment,
        private LoggerInterface $logger,
    ) {}

    public function place(Order $order): void {
        $this->payment->charge($order->total());
        $this->logger->info('Order placed', ['id' => $order->id]);
    }
}
```

**Преимущества DI:**
- Все зависимости видны в конструкторе — прозрачность
- Легко тестировать — подставляем mock
- Соблюдает SOLID (DIP, SRP)
- IDE видит зависимости, статический анализ работает

### Service Locator

Объект **сам запрашивает** зависимости из глобального контейнера:

```php
class OrderService {
    public function place(Order $order): void {
        $payment = App::make(PaymentGateway::class);  // скрытая зависимость!
        $logger = App::make(LoggerInterface::class);   // скрытая зависимость!
        $payment->charge($order->total());
    }
}
```

**Проблемы Service Locator:**
- Зависимости **скрыты** — не видны в конструкторе
- Тестировать сложнее — нужно настраивать глобальный контейнер
- Нарушает DIP — зависит от конкретного контейнера
- Любой класс может получить любой сервис — нет контроля
- Невозможно понять зависимости без чтения всего кода

### Сравнение

| Аспект | DI | Service Locator |
| --- | --- | --- |
| Видимость зависимостей | Явная (конструктор) | Скрытая (внутри методов) |
| Тестирование | Простое (mock в конструктор) | Сложное (настройка контейнера) |
| SOLID | Соблюдает DIP | Нарушает DIP |
| Refactoring | Безопасный | Рискованный |

### Исключения

Service Locator допустим в точках, где DI невозможен:
- **Entry point** приложения (контроллеры в некоторых фреймворках)
- **Factory-классы**, где нужно создавать разные объекты по условию
- Legacy-код как переходный шаг к DI

## Примеры
```text
DI: зависимости передаются в конструктор и видны сразу.
Service Locator: зависимости скрыты внутри методов.
```

## Доп. теория
- DI упрощает тестирование и делает зависимости явными.
- Service Locator часто считается анти‑паттерном из‑за скрытых зависимостей.
