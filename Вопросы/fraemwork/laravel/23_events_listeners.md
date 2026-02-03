## Вопрос: Events & Listeners
Версия: Laravel 12.x (актуальная), 11.x в security-fixes; PHP 8.2–8.4.

## Простой ответ
Event фиксирует факт, Listener реагирует (отправить письмо, записать лог).

## Ответ

Система событий (Events) и слушателей (Listeners) в Laravel реализует паттерн Observer/Pub-Sub. Event описывает факт, который произошёл в системе ("пользователь зарегистрировался", "заказ оплачен"), а Listener реагирует на этот факт (отправляет письмо, записывает лог, обновляет статистику). Это позволяет разделить ответственность: контроллер не должен знать обо всех побочных эффектах.

Главное преимущество -- слабая связанность (loose coupling). Один event может иметь множество listeners, и каждый из них выполняет свою задачу независимо. Добавление нового поведения при регистрации пользователя не требует изменения кода регистрации -- достаточно добавить нового listener.

### Создание Event и Listener

```bash
php artisan make:event UserRegistered
php artisan make:listener SendWelcomeEmail --event=UserRegistered
php artisan make:listener CreateUserProfile --event=UserRegistered
php artisan make:listener NotifyAdmins --event=UserRegistered
```

### Event -- класс события

```php
namespace App\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use Illuminate\Broadcasting\InteractsWithSockets;

class UserRegistered
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(
        public User $user,
        public string $registrationSource = 'web'
    ) {}
}
```

### Listener -- класс слушателя

```php
namespace App\Listeners;

use App\Events\UserRegistered;

class SendWelcomeEmail
{
    public function __construct(
        private MailService $mailService  // Dependency Injection через контейнер
    ) {}

    public function handle(UserRegistered $event): void
    {
        $this->mailService->send(
            $event->user->email,
            new WelcomeMail($event->user)
        );
    }
}
```

### Асинхронные Listeners (через очередь)

Listener может быть помещён в очередь, чтобы не блокировать основной процесс:

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;

    public string $queue = 'emails';       // Очередь
    public int $tries = 3;                 // Попытки
    public int $timeout = 60;              // Таймаут

    public function handle(UserRegistered $event): void
    {
        Mail::to($event->user->email)->send(new WelcomeMail($event->user));
    }

    // Условие: listener выполнится только если вернёт true
    public function shouldQueue(UserRegistered $event): bool
    {
        return $event->user->wants_emails;
    }

    public function failed(UserRegistered $event, Throwable $e): void
    {
        Log::error('Welcome email failed', ['user_id' => $event->user->id]);
    }
}
```

### Регистрация Events и Listeners

**Способ 1: EventServiceProvider (Laravel 10 и ранее)**
```php
// app/Providers/EventServiceProvider.php
protected $listen = [
    UserRegistered::class => [
        SendWelcomeEmail::class,
        CreateUserProfile::class,
        NotifyAdmins::class,
    ],
    OrderPaid::class => [
        SendReceiptEmail::class,
        UpdateInventory::class,
    ],
];
```

**Способ 2: Автоматическое обнаружение (auto-discovery)**

Laravel может автоматически находить listeners по type-hint в методе `handle()`. В Laravel 11 auto-discovery включено по умолчанию.

```php
// EventServiceProvider
public function shouldDiscoverEvents(): bool
{
    return true;  // Включаем автообнаружение
}
```

**Способ 3: В AppServiceProvider (Laravel 11)**
```php
// app/Providers/AppServiceProvider.php
use Illuminate\Support\Facades\Event;

public function boot(): void
{
    Event::listen(UserRegistered::class, SendWelcomeEmail::class);

    // Или closure-listener для простых случаев
    Event::listen(UserRegistered::class, function (UserRegistered $event) {
        Log::info('User registered', ['id' => $event->user->id]);
    });
}
```

### Dispatch (генерация события)

```php
// Через хелпер
event(new UserRegistered($user));

// Через статический метод
UserRegistered::dispatch($user, 'mobile_app');

// Условный dispatch
UserRegistered::dispatchIf($shouldNotify, $user);
UserRegistered::dispatchUnless($isTest, $user);
```

### Event Subscribers

Subscriber объединяет несколько listeners в одном классе -- удобно для группировки связанной логики:

```php
class UserEventSubscriber
{
    public function handleRegistered(UserRegistered $event): void
    {
        // ...
    }

    public function handleDeleted(UserDeleted $event): void
    {
        // ...
    }

    // Метод subscribe связывает события с методами
    public function subscribe(Dispatcher $events): array
    {
        return [
            UserRegistered::class => 'handleRegistered',
            UserDeleted::class => 'handleDeleted',
        ];
    }
}

// Регистрация в EventServiceProvider:
protected $subscribe = [UserEventSubscriber::class];
```

### Практические советы

Используйте events для побочных эффектов (email, логи, уведомления), а не для основной бизнес-логики. Если listener тяжёлый (HTTP-запросы, отправка писем), обязательно реализуйте `ShouldQueue`. Не забывайте про метод `shouldQueue()` для условного помещения в очередь. В тестах можно использовать `Event::fake()` для проверки, что событие было вызвано, без реального выполнения listeners. Для Laravel 11 рекомендуется полагаться на auto-discovery вместо ручной регистрации в `$listen`.

## Примеры

1. `event(new OrderCreated($order))`.
2. `php artisan make:listener SendOrderEmail`.
3. Очередные listener‑ы через `ShouldQueue`.

## Доп. теория

1. События уменьшают связанность.
2. Listener‑ы можно ставить в очередь.
