# Брокеры сообщений: RabbitMQ vs Kafka
| Тема | RabbitMQ | Kafka |
| --- | --- | --- |
| Модель | Очереди + обменники | Топики с партициями (лог) |
| Потребление | Сообщение забирается и исчезает | Читается по оффсету, хранится |
| Маршрутизация | Богатая (direct/fanout/topic/headers) | Простая: ключ → партиция |
| Доставка | At-most/At-least-once | At-least-once, exactly-once с идемпотентностью |
| Нагрузка | Умеренная, задания/ретраи | Очень высокая, стримы/телеметрия |
- Выбор: RabbitMQ — задачи/ретраи/webhooks; Kafka — события/логи, история и реплей, stream processing.

```php
// RabbitMQ publish (php-amqplib)
$channel->basic_publish(new AMQPMessage('data'), exchange: 'events', routing_key: 'user.created');

// Kafka produce (librdkafka)
$producer->newTopic('order.created')->produce(RD_KAFKA_PARTITION_UA, 0, json_encode($payload));
```
