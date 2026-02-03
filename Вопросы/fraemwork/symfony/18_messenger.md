## Вопрос: Messenger (очереди/шина сообщений)
Версия: Symfony 8.0 (stable), LTS 7.4; PHP 8.4+ / 8.2+.

## Простой ответ
компонент для асинхронных задач и обработки сообщений; поддерживает транспорт (AMQP, Redis, Doctrine), ретраи, отложенные сообщения, middleware.

## Ответ
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

## Примеры

1. Маршрутизация `SendEmailMessage` в `async` транспорт.
2. Воркер: `php bin/console messenger:consume async`.
3. Retry‑политика для временных ошибок доставки.

## Доп. теория

1. Синхронные сообщения работают через `sync` транспорт.
2. В проде нужен supervisor/systemd для воркеров.
