## Вопрос: События и подписчики
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
EventDispatcher вызывает слушателей/подписчиков на события ядра и пользовательские.

## Ответ
```php
class UserRegisteredEvent { public function __construct(public User $user) {} }
```
```yaml
# services.yaml
App\EventSubscriber\UserSubscriber:
  tags:
    - { name: kernel.event_subscriber }
```
```php
class UserSubscriber implements EventSubscriberInterface {
    public static function getSubscribedEvents(): array {
        return [UserRegisteredEvent::class => 'onRegistered'];
    }
    public function onRegistered(UserRegisteredEvent $e) { /* ... */ }
}
```

## Примеры

1. Событие `kernel.response` добавляет заголовок.
2. Пользовательское событие `UserRegisteredEvent` отправляет email.
3. Подписчик на `kernel.exception` логирует ошибки.

## Доп. теория

1. Subscriber удобнее listener‑ов, когда нужно подписаться на несколько событий.
2. Слишком много событий усложняет трассировку потока.
