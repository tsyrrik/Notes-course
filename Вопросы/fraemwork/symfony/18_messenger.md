# Messenger (очереди/шина сообщений)

Простыми словами: компонент для асинхронных задач и обработки сообщений; поддерживает транспорт (AMQP, Redis, Doctrine), ретраи, отложенные сообщения, middleware.

```bash
php bin/console make:message SendEmailMessage
```
```php
class SendEmailMessage { public function __construct(public int $userId) {} }

class SendEmailHandler implements MessageHandlerInterface {
    public function __invoke(SendEmailMessage $message) { /* отправить письмо */ }
}
```
```yaml
# config/packages/messenger.yaml
framework:
  messenger:
    transports:
      async: '%env(MESSENGER_TRANSPORT_DSN)%' # например amqp://guest:guest@rabbit:5672/%2f/messages
    routing:
      App\Message\SendEmailMessage: async
```
Worker: `php bin/console messenger:consume async -vv`.
