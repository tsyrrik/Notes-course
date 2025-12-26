## Вопрос: События и подписчики
Версия: Symfony 7.x (LTS 6.4), синтаксис и подходы 6.4+.

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
