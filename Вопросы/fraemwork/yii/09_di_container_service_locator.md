## Вопрос: DI Container и Service Locator
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Есть DI-контейнер и сервис-локатор. Компоненты доступны через `Yii::$app`, а зависимости можно описывать в контейнере.

## Ответ

### Как устроены зависимости в Yii2?

Yii2 предоставляет два механизма управления зависимостями: **DI Container** (`yii\di\Container`) и **Service Locator** (`yii\di\ServiceLocator`). DI Container — это глобальный контейнер, доступный через `Yii::$container`, который умеет автоматически разрешать зависимости через constructor injection. Service Locator — это паттерн, реализованный в `yii\base\Application`, который позволяет обращаться к компонентам по имени через `Yii::$app->componentName`. Оба механизма дополняют друг друга: Service Locator используется для «именованных» сервисов (db, cache, mailer), а DI Container — для автоматического внедрения зависимостей по типу.

### DI Container — автоматическое разрешение зависимостей

`Yii::$container` поддерживает три типа зависимостей: привязку интерфейса к реализации (`set`), синглтоны (`setSingleton`) и определения с параметрами. Когда объект создается через `Yii::createObject()` или через DI Container, контейнер автоматически анализирует конструктор и внедряет зависимости.

```php
// Привязка интерфейса к реализации
Yii::$container->set(
    'app\interfaces\PaymentGatewayInterface',
    'app\services\StripeGateway'
);

// Привязка с параметрами конструктора
Yii::$container->set(
    'app\interfaces\PaymentGatewayInterface',
    [
        'class' => 'app\services\StripeGateway',
        'apiKey' => 'sk_test_xxx',
        'currency' => 'RUB',
    ]
);

// Синглтон — один экземпляр на все приложение
Yii::$container->setSingleton(
    'app\interfaces\LoggerInterface',
    'app\services\FileLogger'
);

// Привязка через callable
Yii::$container->set('app\services\ReportService', function ($container, $params, $config) {
    $logger = $container->get('app\interfaces\LoggerInterface');
    return new \app\services\ReportService($logger, $config);
});
```

### Constructor Injection

DI Container анализирует type hints параметров конструктора и автоматически создает нужные объекты. Это основной механизм внедрения зависимостей в Yii2.

```php
namespace app\services;

use app\interfaces\PaymentGatewayInterface;
use app\interfaces\NotificationServiceInterface;

class OrderService
{
    private $paymentGateway;
    private $notificationService;

    // DI Container автоматически внедрит зависимости
    public function __construct(
        PaymentGatewayInterface $paymentGateway,
        NotificationServiceInterface $notificationService
    ) {
        $this->paymentGateway = $paymentGateway;
        $this->notificationService = $notificationService;
    }

    public function createOrder(array $data): Order
    {
        $order = new Order($data);
        $order->save();

        $this->paymentGateway->charge($order->total);
        $this->notificationService->sendOrderConfirmation($order);

        return $order;
    }
}

// Создание через контейнер — зависимости внедрятся автоматически
$orderService = Yii::$container->get('app\services\OrderService');
// или
$orderService = Yii::createObject('app\services\OrderService');
```

### Service Locator — компоненты приложения

`Yii::$app` — это Service Locator. Компоненты регистрируются в конфигурации и создаются лениво (при первом обращении). Каждый компонент — это объект, наследующий `yii\base\Component`, с уникальным именем.

```php
// Регистрация в конфиге
'components' => [
    'db' => [
        'class' => 'yii\db\Connection',
        'dsn' => 'mysql:host=localhost;dbname=myapp',
        'username' => 'root',
        'password' => '',
    ],
    'cache' => [
        'class' => 'yii\caching\RedisCache',
    ],
    'paymentService' => [
        'class' => 'app\services\PaymentService',
        'gateway' => 'stripe',
        'testMode' => YII_ENV_DEV,
    ],
    'smsService' => [
        'class' => 'app\services\SmsService',
        'provider' => 'twilio',
        'apiKey' => 'your-api-key',
    ],
],

// Использование
$db = Yii::$app->db;                     // создается при первом вызове
$cache = Yii::$app->cache;
$payment = Yii::$app->paymentService;
$sms = Yii::$app->smsService;

// Проверка существования компонента
if (Yii::$app->has('redis')) {
    $redis = Yii::$app->redis;
}
```

### Разница между DI Container и Service Locator

Service Locator хранит компоненты **по имени** и вызывается явно (`Yii::$app->db`). DI Container работает **по типу** и внедряет зависимости **автоматически** через конструктор. Service Locator удобен для инфраструктурных сервисов (db, cache, mailer), а DI Container — для бизнес-логики с явными зависимостями.

```php
// Service Locator — явный вызов, скрывает зависимости
class OrderController extends Controller
{
    public function actionCreate()
    {
        // Зависимость скрыта внутри метода
        $gateway = Yii::$app->paymentGateway;
        $gateway->charge(100);
    }
}

// DI Container — явные зависимости, лучше для тестирования
class OrderController extends Controller
{
    private $gateway;

    public function __construct($id, $module, PaymentGatewayInterface $gateway, $config = [])
    {
        $this->gateway = $gateway;
        parent::__construct($id, $module, $config);
    }

    public function actionCreate()
    {
        $this->gateway->charge(100);
    }
}
```

### Конфигурация DI Container в bootstrap

Привязки DI Container обычно настраиваются в файле `bootstrap.php` или в конфигурации приложения через ключ `container`.

```php
// config/web.php
return [
    'id' => 'app',
    // ...
    'container' => [
        'definitions' => [
            'app\interfaces\PaymentGatewayInterface' => 'app\services\StripeGateway',
            'app\interfaces\LoggerInterface' => 'app\services\FileLogger',
            'yii\mail\MailerInterface' => function () {
                return Yii::$app->mailer;
            },
        ],
        'singletons' => [
            'app\services\EventBus' => [
                'class' => 'app\services\EventBus',
            ],
        ],
    ],
];

// Или в bootstrap.php
Yii::$container->set('app\interfaces\CacheInterface', function () {
    return Yii::$app->cache;
});
```

### Практические советы

Для бизнес-логики предпочитайте DI Container и constructor injection — это делает зависимости явными и упрощает тестирование (можно подменить зависимость на mock). Service Locator (`Yii::$app->...`) удобен для быстрого доступа к инфраструктурным компонентам, но злоупотребление им приводит к скрытым зависимостям и трудностям при unit-тестировании. Используйте интерфейсы для определения контрактов — это позволяет легко переключать реализации (например, подменить платежный шлюз для тестов). Помните, что `Yii::createObject()` всегда использует DI Container, поэтому для создания объектов с зависимостями предпочитайте его вместо оператора `new`.

## Примеры
1. `Yii::$container->set(LoggerInterface::class, FileLogger::class);`.
2. `Yii::$app->cache` — доступ к компоненту через Service Locator.
3. Контроллер с constructor injection для `PaymentGatewayInterface`.

## Доп. теория
1. Service Locator скрывает зависимости, DI делает их явными и тестируемыми.
2. `Yii::createObject()` и `Yii::$container->get()` используют контейнер.
