## Вопрос: RabbitMQ

## Простой ответ
- Умеет сложную маршрутизацию: по ключам, шаблонам, заголовкам.
- Сообщение живёт до ack; можно ставить TTL, приоритет, dead-letter.
- Подходит для джобов, ретраев, вебхуков и фан-аут рассылок.

## Ответ

### Что такое RabbitMQ

RabbitMQ — это брокер сообщений с открытым исходным кодом, реализующий протокол AMQP (Advanced Message Queuing Protocol). Он принимает сообщения от продюсеров, маршрутизирует их через exchange (обменники) и доставляет в очереди, откуда их забирают потребители. Написан на Erlang/OTP, что обеспечивает высокую отказоустойчивость и поддержку кластеризации.

### Архитектура: ключевые компоненты

Основной принцип RabbitMQ: продюсер никогда не отправляет сообщение напрямую в очередь — он отправляет его в exchange, который решает, куда маршрутизировать.

```
Producer → Exchange → Binding (routing key) → Queue → Consumer
```

- **Exchange** — принимает сообщения и маршрутизирует их в очереди по правилам.
- **Queue** — буфер, хранящий сообщения до обработки потребителем.
- **Binding** — связь между exchange и queue с правилом маршрутизации.
- **Routing Key** — ключ, по которому exchange решает, в какую очередь направить сообщение.
- **Virtual Host (vhost)** — логическая изоляция (как namespace): свои exchange, queue, пользователи.

### Типы Exchange

| Тип | Маршрутизация | Использование |
|---|---|---|
| `direct` | Точное совпадение routing key | Конкретные задачи: email.send, order.process |
| `fanout` | Во все привязанные очереди (broadcast) | Рассылка уведомлений всем подписчикам |
| `topic` | По шаблону routing key (*.log, order.#) | Гибкая маршрутизация по категориям |
| `headers` | По заголовкам сообщения (не по routing key) | Сложная маршрутизация по метаданным |

```bash
# Шаблоны для topic exchange:
# * — ровно одно слово:      order.* → order.created, order.deleted
# # — ноль или более слов:   order.# → order.created, order.item.added
```

### Гарантии доставки

RabbitMQ обеспечивает надёжную доставку через несколько механизмов:

- **Acknowledgement (ACK)** — потребитель подтверждает обработку. Без ACK сообщение перевыдаётся.
- **Publisher Confirms** — брокер подтверждает продюсеру, что сообщение записано.
- **Persistent Messages** — сообщения записываются на диск (delivery_mode=2).
- **Durable Queues** — очередь переживает перезапуск RabbitMQ.
- **Dead Letter Exchange (DLX)** — сообщения, которые не удалось обработать, перенаправляются в DLX.
- **TTL** — время жизни сообщения и/или очереди.
- **Priority Queues** — приоритетные сообщения обрабатываются первыми (до 255 уровней).

### Пример: полная настройка с DLX (php-amqplib)

```php
$conn = new AMQPStreamConnection('rabbit', 5672, 'guest', 'guest');
$ch = $conn->channel();

// Dead Letter Exchange
$ch->exchange_declare('dlx', 'direct', false, true, false);
$ch->queue_declare('failed_jobs', false, true, false, false);
$ch->queue_bind('failed_jobs', 'dlx', 'dead');

// Основной exchange и очередь с DLX
$ch->exchange_declare('events', 'topic', false, true, false);
$ch->queue_declare('email.send', false, true, false, false, false, new AMQPTable([
    'x-dead-letter-exchange'    => 'dlx',
    'x-dead-letter-routing-key' => 'dead',
    'x-message-ttl'             => 60000,   // 60 секунд TTL
    'x-max-length'              => 10000,   // максимум 10000 сообщений
]));
$ch->queue_bind('email.send', 'events', 'user.created');

// Publisher с confirms
$ch->confirm_select();
$msg = new AMQPMessage(json_encode(['id' => 1, 'email' => 'user@test.com']), [
    'delivery_mode' => AMQPMessage::DELIVERY_MODE_PERSISTENT,
    'content_type'  => 'application/json',
]);
$ch->basic_publish($msg, 'events', 'user.created');
$ch->wait_for_pending_acks();

// Consumer с prefetch и manual ack
$ch->basic_qos(0, 10, false);  // prefetch: не более 10 сообщений за раз
$ch->basic_consume('email.send', '', false, false, false, false, function(AMQPMessage $m) {
    try {
        $data = json_decode($m->body, true);
        sendEmail($data['email']);
        $m->ack();
    } catch (\Exception $e) {
        $m->nack(false, false);  // reject без requeue → уйдёт в DLX
    }
});
while ($ch->is_consuming()) { $ch->wait(); }
```

### Кластеризация и высокая доступность

RabbitMQ поддерживает кластеризацию из нескольких нод:

- **Classic Mirrored Queues** (устаревший) — реплицирует очереди между нодами.
- **Quorum Queues** (рекомендуемый) — консенсус на основе Raft, надёжнее и производительнее.
- **Streams** (с RabbitMQ 3.9+) — append-only лог, похожий на Kafka, для высокой пропускной способности.

```bash
# Объявление quorum queue
# В PHP: arguments x-queue-type => quorum
```

### Мониторинг и управление

```bash
# Management UI — веб-интерфейс (порт 15672)
# rabbitmqctl — CLI утилита

rabbitmqctl list_queues name messages consumers
rabbitmqctl list_exchanges name type
rabbitmqctl list_bindings

# Метрики для Prometheus (плагин rabbitmq_prometheus)
# Endpoint: http://rabbit:15692/metrics
```

### Когда использовать

- Задачи/джобы с ретраями и приоритетами.
- Сложная маршрутизация (topic, headers).
- Вебхуки, email-рассылки, fanout-уведомления.
- RPC через очереди (request-reply pattern).
- Умеренные нагрузки (тысячи-десятки тысяч msg/sec).

### Не лучший выбор, если

- Нужен event streaming с длительным хранением и replay истории — лучше Kafka.
- Нужна экстремальная пропускная способность (миллионы msg/sec) — Kafka эффективнее.
- Строгий порядок при множестве ретраев — порядок может нарушиться (сообщение возвращается в конец очереди).

## Примеры
```text
RabbitMQ удобен для задач/джобов с ретраями и сложной маршрутизацией.
```

## Доп. теория
- Quorum queues — современная замена mirrored queues.
- Для гарантии доставки используйте durable queue + persistent messages + publisher confirms.
