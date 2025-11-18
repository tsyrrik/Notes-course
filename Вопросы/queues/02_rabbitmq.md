# RabbitMQ
Очередь/роутер сообщений с обменниками и маршрутными ключами.

## Простыми словами
- Умеет сложную маршрутизацию: по ключам, шаблонам, заголовкам.
- Сообщение живёт до ack; можно ставить TTL, приоритет, dead-letter.
- Подходит для джобов, ретраев, вебхуков и фан-аут рассылок.

## Основы
- Exchange принимает сообщения и маршрутизирует в очереди: `direct`, `fanout`, `topic`, `headers`.
- Queue хранит сообщения до обработки; ack гарантирует доставку.
- Binding связывает exchange и queue с правилом (routing key, шаблон).
- Поддерживает TTL, DLX (dead-letter), приоритеты, отложенные сообщения.

## Пример (php-amqplib)
```php
$conn = new AMQPStreamConnection('rabbit', 5672, 'guest', 'guest');
$ch = $conn->channel();
$ch->exchange_declare('events', 'topic', false, true, false);
$ch->queue_declare('email.send', false, true, false, false);
$ch->queue_bind('email.send', 'events', 'user.created');
$ch->basic_publish(new AMQPMessage(json_encode(['id' => 1])), 'events', 'user.created');
$ch->basic_consume('email.send', '', false, false, false, false, function(AMQPMessage $m) {
    sendEmail($m->body);
    $m->ack();
});
while ($ch->is_consuming()) { $ch->wait(); }
```

## Когда использовать
- Задачи/джобы, ретраи, вебхуки, fanout рассылки, сложная маршрутизация.
- Нагрузки умеренные; нужен управляемый порядок/приоритеты.

## Не лучший выбор, если
- Нужен event streaming и длительное хранение с «переигрыванием» истории.
- Вся логика держится на строгом порядке при множестве ретраев — может сместиться.
