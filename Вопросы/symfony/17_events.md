# События и подписчики

Простыми словами: EventDispatcher вызывает слушателей/подписчиков на события ядра и пользовательские.

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
