## Вопрос: DI контейнер и автоконфигурация
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
все сервисы регистрируются в контейнере; автоконфигурация/автовайринг подставляет зависимости по типам автоматически.

## Ответ

### Что такое DI-контейнер

DI-контейнер (Dependency Injection Container) -- это центральный компонент Symfony, который управляет созданием, конфигурацией и внедрением зависимостей всех сервисов приложения. Контейнер реализует паттерн **Inversion of Control (IoC)** -- вместо того, чтобы классы сами создавали свои зависимости, контейнер инжектит их извне. Это обеспечивает слабую связанность, тестируемость и гибкость.

### Autowiring

Autowiring -- это способность контейнера автоматически определять зависимости сервиса по type-hints аргументов конструктора. Если в конструкторе указан тип `MailerInterface`, контейнер найдёт сервис, реализующий этот интерфейс, и подставит его:

```php
class NotificationService
{
    // Symfony автоматически подставит реализацию MailerInterface
    public function __construct(
        private readonly MailerInterface $mailer,
        private readonly LoggerInterface $logger,
        private readonly UserRepository $userRepository,
    ) {}
}
```

### Autoconfigure

Autoconfigure автоматически применяет теги к сервисам на основе реализуемых интерфейсов. Например, если класс реализует `EventSubscriberInterface`, контейнер автоматически добавит тег `kernel.event_subscriber`:

```yaml
# config/services.yaml
services:
    _defaults:
        autowire: true       # Автоматическое внедрение зависимостей
        autoconfigure: true  # Автоматическое применение тегов
        public: false        # Сервисы приватные по умолчанию

    App\:
        resource: '../src/'
        exclude:
            - '../src/DependencyInjection/'
            - '../src/Entity/'
            - '../src/Kernel.php'
```

### Ручная конфигурация сервисов

Для случаев, когда autowiring не может определить зависимость (несколько реализаций интерфейса, скалярные параметры):

```yaml
services:
    # Привязка интерфейса к конкретной реализации
    App\Service\PaymentGatewayInterface:
        alias: App\Service\StripePaymentGateway

    # Ручная конфигурация с аргументами
    App\Service\ReportGenerator:
        arguments:
            $exportDir: '%kernel.project_dir%/var/exports'
            $defaultFormat: 'pdf'

    # Привязка по named autowiring
    App\Service\SlowApiClient:
        arguments:
            $client: '@http_client.slow'
```

### Bind -- привязка скалярных параметров

```yaml
services:
    _defaults:
        autowire: true
        autoconfigure: true
        bind:
            $projectDir: '%kernel.project_dir%'
            $adminEmail: '%env(ADMIN_EMAIL)%'
            $uploadDir: '%kernel.project_dir%/public/uploads'
            Psr\Log\LoggerInterface $auditLogger: '@monolog.logger.audit'
```

```php
class FileUploader
{
    // $uploadDir будет автоматически подставлен из bind
    public function __construct(
        private readonly string $uploadDir,
    ) {}
}
```

### Атрибуты для DI (Symfony 6.1+/7)

В Symfony 7 активно используются PHP-атрибуты для конфигурации сервисов:

```php
use Symfony\Component\DependencyInjection\Attribute\Autowire;
use Symfony\Component\DependencyInjection\Attribute\AsAlias;
use Symfony\Component\DependencyInjection\Attribute\TaggedIterator;
use Symfony\Component\DependencyInjection\Attribute\When;

class ReportGenerator
{
    public function __construct(
        #[Autowire('%kernel.project_dir%/var/exports')]
        private readonly string $exportDir,

        #[Autowire(env: 'ADMIN_EMAIL')]
        private readonly string $adminEmail,

        #[Autowire(service: 'monolog.logger.audit')]
        private readonly LoggerInterface $auditLogger,

        // Получить все сервисы с тегом 'app.report_formatter'
        #[TaggedIterator('app.report_formatter')]
        private readonly iterable $formatters,
    ) {}
}

// Регистрация только для определённого окружения
#[When(env: 'dev')]
class DebugMailer implements MailerInterface { /* ... */ }

// Атрибут AsAlias для привязки интерфейса
#[AsAlias(PaymentGatewayInterface::class)]
class StripePaymentGateway implements PaymentGatewayInterface { /* ... */ }
```

### Scope и Lifecycle сервисов

По умолчанию все сервисы являются **shared** (singleton) -- создаются один раз и переиспользуются. Для создания нового экземпляра при каждом запросе:

```yaml
services:
    App\Service\NonSharedService:
        shared: false   # Новый экземпляр при каждом получении
```

### Декораторы и приоритеты

```php
// Декоратор сервиса через атрибут
use Symfony\Component\DependencyInjection\Attribute\AsDecorator;

#[AsDecorator(decorates: PaymentGatewayInterface::class)]
class LoggingPaymentGateway implements PaymentGatewayInterface
{
    public function __construct(
        private readonly PaymentGatewayInterface $inner,
        private readonly LoggerInterface $logger,
    ) {}

    public function charge(float $amount): bool
    {
        $this->logger->info('Charging {amount}', ['amount' => $amount]);
        return $this->inner->charge($amount);
    }
}
```

### Отладка контейнера

```bash
# Все сервисы
php bin/console debug:container

# Поиск сервиса
php bin/console debug:container mailer

# Autowiring-кандидаты для интерфейса
php bin/console debug:autowiring MailerInterface

# Сервисы с определённым тегом
php bin/console debug:container --tag=kernel.event_subscriber

# Параметры контейнера
php bin/console debug:container --parameters
```

### Практические советы

- Не инжектите `ContainerInterface` напрямую -- это Service Locator антипаттерн. Используйте конкретные сервисы через autowiring
- Для нескольких реализаций одного интерфейса используйте named autowiring (`#[Autowire(service: '...')]`) или `#[AsAlias]`
- Все сервисы по умолчанию private -- это правильно; получать их из контейнера напрямую не нужно
- Используйте `#[TaggedIterator]` или `#[TaggedLocator]` для паттерна Strategy/Chain of Responsibility
- В тестах сервисы можно подменять через `self::getContainer()->set('service_id', $mock)`

## Примеры

1. Autowiring: инжект `MailerInterface` в сервис.
2. `bind` для `$uploadDir` в `services.yaml`.
3. `TaggedIterator` для набора обработчиков.

## Доп. теория

1. Все сервисы по умолчанию private и shared.
2. Для нескольких реализаций используйте alias или named autowiring.
