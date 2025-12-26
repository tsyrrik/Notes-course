## Вопрос: Events & Listeners
Версия: Laravel 11 (актуально; базовые принципы применимы к 10+).

## Простой ответ
Event фиксирует факт, Listener реагирует (отправить письмо, записать лог).

## Ответ
События описывают факт, слушатели — реакцию.
```bash
php artisan make:event UserRegistered
php artisan make:listener SendWelcomeEmail --event=UserRegistered
```
```php
class UserRegistered { use Dispatchable, InteractsWithSockets, SerializesModels; public function __construct(public User $user) {} }
class SendWelcomeEmail { public function handle(UserRegistered $event) { Mail::to($event->user->email)->send(new WelcomeMail($event->user)); } }

// EventServiceProvider
protected $listen = [UserRegistered::class => [SendWelcomeEmail::class]];

// Генерация
event(new UserRegistered($user));
UserRegistered::dispatch($user);
```
