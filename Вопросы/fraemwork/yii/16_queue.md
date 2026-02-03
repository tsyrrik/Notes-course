## Вопрос: Очереди
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Через расширение `yiisoft/yii2-queue` можно использовать DB, Redis, AMQP, RabbitMQ, Beanstalkd и др.

## Ответ

Очереди в Yii2 обычно реализуют через `yiisoft/yii2-queue`. Поддерживаются драйверы DB, Redis, AMQP/RabbitMQ, Beanstalkd и другие. Задания реализуют `JobInterface`.

Как выглядит обработчик задачи:
```php
class SendEmailJob extends yii\\base\\BaseObject implements yii\\queue\\JobInterface {
    public function execute($queue) {
        // логика отправки
    }
}
```

Как запустить воркер:
- `php yii queue/run`
- или `php yii queue/listen`

## Примеры

1. Пуш задачи: `Yii::$app->queue->push(new SendEmailJob([...]))`.
2. Воркер: `php yii queue/listen --verbose`.
3. Очередь на Redis в конфиге `components['queue']`.

## Доп. теория

1. `queue/run` выполняет разово, `queue/listen` — бесконечный воркер.
2. Для продакшена нужен supervisor/systemd для перезапуска воркеров.
