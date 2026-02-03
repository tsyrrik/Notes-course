## Вопрос: DDD (Domain-Driven Design)

## Простой ответ
- Сначала понимаем бизнес, затем пишем код его языком.
- Модель домена — главный слой; внешние детали (БД, UI) подстраиваются под неё.
- Хорош для сложных правил и терминов; лишний для простого CRUD.

## Ответ

### Суть подхода

Domain-Driven Design (DDD) -- это подход к проектированию программного обеспечения, в котором центральное место занимает предметная область (domain) и её модель, отражающая реальные бизнес-правила. Подход был предложен Эриком Эвансом в книге *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003). Ключевая идея состоит в том, что сложность программных систем определяется не технологиями, а сложностью самого бизнеса. Поэтому разработка должна начинаться с глубокого понимания предметной области, а код должен быть прямым отражением бизнес-процессов и правил.

DDD разделяется на две большие части: **Strategic Design** (стратегическое проектирование) и **Tactical Design** (тактическое проектирование). Стратегический уровень определяет границы систем, контексты и взаимодействие между командами. Тактический уровень задаёт конкретные паттерны внутри кода: сущности, value objects, агрегаты и т. д.

### Ubiquitous Language (единый язык)

Одна из самых важных концепций DDD -- Ubiquitous Language. Это общий словарь терминов, который используют и разработчики, и бизнес-эксперты. Термины из этого словаря напрямую отражаются в именах классов, методов, переменных и таблиц. Если бизнес говорит "заказ", в коде должен быть `Order`, а не `DataRecord42`. Если бизнес говорит "подтвердить заказ", метод должен называться `confirm()`, а не `setStatus(2)`. Единый язык устраняет "перевод" между бизнесом и разработкой, снижая вероятность ошибок и недопонимания.

### Bounded Context (ограниченный контекст)

Bounded Context -- это явная граница, внутри которой определённая модель домена имеет конкретное, однозначное значение. Например, понятие "Пользователь" в контексте биллинга (имя, платёжные данные, баланс) отличается от "Пользователя" в контексте авторизации (логин, роли, токены). Попытка создать единую модель `User` для всех случаев приводит к "Большому комку грязи" (Big Ball of Mud). Каждый Bounded Context имеет свою модель, свой Ubiquitous Language и, как правило, свою базу данных.

```text
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   Billing Context   │     │   Auth Context       │     │   Catalog Context   │
│                     │     │                      │     │                     │
│  User = {           │     │  User = {            │     │  Product = {        │
│    name,            │     │    login,            │     │    sku,             │
│    balance,         │◄───►│    roles,            │◄───►│    title,           │
│    paymentMethod    │ ACL │    passwordHash      │ ACL │    price            │
│  }                  │     │  }                   │     │  }                  │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
       ACL = Anti-Corruption Layer (слой защиты от чужой модели)
```

### Context Map (карта контекстов)

Context Map описывает, как разные Bounded Contexts взаимодействуют между собой. Основные типы отношений:

| Отношение | Описание |
| --- | --- |
| **Shared Kernel** | Два контекста разделяют общую часть модели (опасно, связывает команды). |
| **Customer-Supplier** | Один контекст поставляет данные другому; поставщик учитывает потребности потребителя. |
| **Conformist** | Потребитель принимает модель поставщика "как есть", без переговоров. |
| **Anti-Corruption Layer (ACL)** | Потребитель строит слой-переводчик, чтобы защитить свою модель от чужой. |
| **Open Host Service** | Поставщик публикует стабильный API для множества потребителей. |
| **Published Language** | Общий формат обмена данными (JSON Schema, Protobuf, Avro). |

### Слои в DDD-приложении

```text
┌──────────────────────────────────────────────┐
│              Interface / UI Layer             │  ← Контроллеры, CLI, GraphQL
├──────────────────────────────────────────────┤
│             Application Layer                │  ← Use-case handlers, команды, DTO
├──────────────────────────────────────────────┤
│              Domain Layer (ядро)             │  ← Entities, VO, Aggregates, Domain Services
├──────────────────────────────────────────────┤
│           Infrastructure Layer               │  ← ORM, Repositories impl, Message Bus
└──────────────────────────────────────────────┘
    Зависимости: UI → Application → Domain ← Infrastructure
    (Infrastructure реализует интерфейсы Domain через Dependency Inversion)
```

1. **Domain Layer** -- сущности, value objects, агрегаты, доменные сервисы, доменные события. Это ядро, которое не зависит ни от чего внешнего.
2. **Application Layer** -- use-case'ы, оркестрация, команды/запросы (CQRS). Не содержит бизнес-логики, а вызывает методы домена.
3. **Infrastructure Layer** -- реализации репозиториев, ORM, отправка email, брокеры сообщений. Зависит от Domain Layer через интерфейсы (Dependency Inversion).
4. **Interface/UI Layer** -- HTTP-контроллеры, CLI-команды, GraphQL-резолверы. Принимает запросы, создаёт DTO, передаёт в Application Layer.

## Элементы модели (Tactical Patterns)

| Элемент | Описание |
| --- | --- |
| **Entity** | Объект с уникальной идентичностью (ID), который живёт во времени и может менять состояние. Два объекта с одинаковыми полями, но разными ID -- разные сущности. |
| **Value Object** | Неизменяемый объект без ID, определяется только своими значениями. Два VO с одинаковыми значениями -- один и тот же объект (Money, Address, Email). |
| **Aggregate** | Кластер связанных Entity и VO, который изменяется как единое целое. Aggregate защищает инварианты: невозможно привести его в невалидное состояние через публичный API. |
| **Aggregate Root** | Единственная точка входа в Aggregate. Внешний код не может напрямую обращаться к внутренним сущностям -- только через Root. |
| **Repository** | Абстракция (интерфейс) для загрузки и сохранения Aggregate. Реализация живёт в Infrastructure. |
| **Domain Service** | Бизнес-логика, которая не принадлежит ни одной конкретной сущности (например, расчёт стоимости доставки с учётом нескольких агрегатов). |
| **Application Service** | Оркестрация use-case'а: получает команду, загружает агрегаты из репозитория, вызывает доменную логику, сохраняет результат. |
| **Domain Event** | Событие, произошедшее в домене (`OrderPlaced`, `PaymentReceived`). Используется для реакции между агрегатами и контекстами. |
| **Factory** | Создание сложных объектов/агрегатов, инкапсуляция логики создания. |

### Правила проектирования Aggregates

- Агрегат должен быть **маленьким** -- один Root + минимум внутренних сущностей.
- Ссылки между агрегатами -- **только по ID**, а не по прямой объектной ссылке.
- Один use-case изменяет **один агрегат** за транзакцию. Если нужно изменить несколько -- используйте Domain Events и eventual consistency.
- Агрегат защищает свои **инварианты** -- бизнес-правила, которые всегда должны быть истинными.

## Отличительные свойства
- Модель привязана к бизнесу, единый язык, изоляция бизнес-логики.
- Инварианты внутри агрегатов, независимость от инфраструктуры.
- Поддерживает сложные правила, удобно при частых изменениях логики.

## Когда применять
- Сложный домен, много правил/исключений, логика часто меняется.
- Несколько команд/ролей, нужен общий язык.
- Нужна чёткая слоистая архитектура и стабильная бизнес-модель.
- Не подходит для простых CRUD/сервисов без логики -- внесёт лишнюю сложность.
- **Правило**: если вся логика сводится к "принять JSON, положить в таблицу, отдать JSON" -- DDD не нужен.

## Примеры кода

### Entity + Value Object + Aggregate

```php
// Value Object -- неизменяемый, сравнивается по значению
final class Money {
    public function __construct(
        public readonly int    $amount,
        public readonly string $currency
    ) {}

    public function add(Money $other): self {
        if ($this->currency !== $other->currency) {
            throw new \DomainException('Нельзя складывать разные валюты');
        }
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function equals(Money $other): bool {
        return $this->amount === $other->amount
            && $this->currency === $other->currency;
    }
}

// Value Object -- идентификатор
final class OrderId {
    public function __construct(public readonly string $value) {}
}

// Entity внутри агрегата
final class OrderLine {
    public function __construct(
        private string $productId,
        private int    $quantity,
        private Money  $price
    ) {
        if ($quantity <= 0) {
            throw new \DomainException('Количество должно быть > 0');
        }
    }

    public function lineTotal(): Money {
        return new Money($this->price->amount * $this->quantity, $this->price->currency);
    }
}

// Aggregate Root -- точка входа, защищает инварианты
final class Order {
    /** @var OrderLine[] */
    private array  $lines = [];
    private string $status = 'draft';
    /** @var object[] */
    private array  $domainEvents = [];

    public function __construct(
        private OrderId $id,
        private string  $customerId
    ) {}

    public function addLine(string $productId, int $qty, Money $price): void {
        if ($this->status !== 'draft') {
            throw new \DomainException('Нельзя менять подтверждённый заказ');
        }
        $this->lines[] = new OrderLine($productId, $qty, $price);
    }

    public function confirm(): void {
        if (empty($this->lines)) {
            throw new \DomainException('Нельзя подтвердить пустой заказ');
        }
        $this->status = 'confirmed';
        $this->domainEvents[] = new OrderConfirmed($this->id);
    }

    public function total(): Money {
        $total = new Money(0, 'RUB');
        foreach ($this->lines as $line) {
            $total = $total->add($line->lineTotal());
        }
        return $total;
    }

    public function pullDomainEvents(): array {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}
```

### Repository (интерфейс в Domain, реализация в Infrastructure)

```php
// Domain Layer -- интерфейс
interface OrderRepository {
    public function byId(OrderId $id): ?Order;
    public function save(Order $order): void;
    public function nextId(): OrderId;
}

// Infrastructure Layer -- реализация
final class DoctrineOrderRepository implements OrderRepository {
    public function __construct(private EntityManagerInterface $em) {}

    public function byId(OrderId $id): ?Order {
        return $this->em->find(Order::class, $id->value);
    }

    public function save(Order $order): void {
        $this->em->persist($order);
        $this->em->flush();
    }

    public function nextId(): OrderId {
        return new OrderId(Uuid::uuid4()->toString());
    }
}
```

### Application Service (use-case)

```php
final class ConfirmOrderHandler {
    public function __construct(
        private OrderRepository   $orders,
        private EventDispatcher   $events
    ) {}

    public function __invoke(ConfirmOrderCommand $cmd): void {
        $order = $this->orders->byId(new OrderId($cmd->orderId));
        if ($order === null) {
            throw new \RuntimeException('Заказ не найден');
        }

        $order->confirm();                       // доменная логика
        $this->orders->save($order);             // сохранение
        foreach ($order->pullDomainEvents() as $event) {
            $this->events->dispatch($event);     // публикация событий
        }
    }
}
```

### Диаграмма потока данных в DDD-приложении

```text
HTTP Request
     │
     ▼
┌─────────────┐    DTO/Command     ┌───────────────────┐
│ Controller  │ ──────────────────►│ Application Service│
│ (UI Layer)  │                    │ (Use-case handler) │
└─────────────┘                    └────────┬──────────┘
                                            │
                          ┌─────────────────┼─────────────────┐
                          ▼                 ▼                 ▼
                   ┌────────────┐   ┌─────────────┐   ┌────────────┐
                   │ Repository │   │  Aggregate   │   │  Domain    │
                   │ (interface)│   │  Root        │   │  Service   │
                   └──────┬─────┘   └─────────────┘   └────────────┘
                          │
                          ▼
                   ┌─────────────┐
                   │ DB / ORM    │  (Infrastructure)
                   └─────────────┘
```

## Примеры
```text
Когда домен сложный (много правил), DDD помогает держать модель и язык едиными.
Когда CRUD простой — DDD часто избыточен.
```

## Доп. теория
- DDD — это не про папки/слои, а про работу с доменной сложностью и языком предметной области.
- На практике часто применяют «облегчённый» DDD без полного набора паттернов.
