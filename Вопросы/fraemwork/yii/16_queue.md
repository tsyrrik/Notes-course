## Вопрос: Очереди
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Через расширение `yiisoft/yii2-queue` можно использовать DB, Redis, AMQP, RabbitMQ, Beanstalkd и др.

## Ответ
## Вопрос: Какие очереди поддерживает Yii2?
Ответ: Через расширение `yiisoft/yii2-queue` можно использовать DB, Redis, AMQP, RabbitMQ, Beanstalkd и др.

## Вопрос: Как выглядит обработчик задачи?
Ответ:
```php
class SendEmailJob extends yii\\base\\BaseObject implements yii\\queue\\JobInterface {
    public function execute($queue) {
        // логика отправки
    }
}
```

## Вопрос: Как запустить воркер?
Ответ:
- `php yii queue/run`
- или `php yii queue/listen`
