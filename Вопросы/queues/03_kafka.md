## Вопрос: Kafka

## Простой ответ
- Потоковая «чёрная коробка» хранит события в порядке и даёт читать их разным сервисам.
- Сообщения не исчезают сразу: можно переигрывать историю и строить витрины/аналитику.
- Раскрывается при больших объёмах событий и нескольких независимых потребителях.

## Ответ

### Что такое Apache Kafka

Apache Kafka — это распределённая платформа потоковой обработки событий (event streaming platform). В отличие от классических брокеров сообщений, Kafka хранит сообщения как append-only лог с настраиваемым retention. Сообщения не удаляются после обработки — их можно читать повторно (replay), что делает Kafka идеальной для event sourcing, аналитики и CDC.

### Архитектура

```
Producer → Broker (Cluster) → Topic → Partitions → Consumer Group → Consumer
```

**Основные компоненты:**

| Компонент | Описание |
|---|---|
| **Broker** | Сервер Kafka, хранит данные. Кластер состоит из нескольких брокеров |
| **Topic** | Логическая категория сообщений (аналог таблицы в БД) |
| **Partition** | Физическое разбиение topic. Append-only лог с гарантией порядка внутри |
| **Offset** | Порядковый номер сообщения в partition. Consumer читает по offset |
| **Consumer Group** | Группа потребителей, делящих partition между собой. Каждый partition — один consumer |
| **Replication Factor** | Количество копий partition на разных брокерах (обычно 3) |
| **ZooKeeper / KRaft** | Координация кластера. KRaft (с Kafka 3.3+) заменяет ZooKeeper |

### Партиционирование и порядок

Порядок сообщений гарантирован **только внутри одной partition**. Producer указывает ключ сообщения (message key) — по нему определяется partition (hash(key) % num_partitions). Сообщения с одним ключом всегда попадают в одну partition и обрабатываются в порядке отправки.

```
Topic: orders (3 partitions)
├── Partition 0: [order-1, order-4, order-7]   → Consumer A
├── Partition 1: [order-2, order-5, order-8]   → Consumer B
└── Partition 2: [order-3, order-6, order-9]   → Consumer C
```

### Consumer Groups

Consumer Group — это механизм параллельной обработки. Каждая partition назначается ровно одному consumer в группе. Разные группы читают независимо друг от друга (каждая со своим offset).

```
Topic: events (3 partitions)

Group "billing":  Consumer1 → P0, P1  |  Consumer2 → P2
Group "analytics": Consumer3 → P0, P1, P2   (отдельный offset)
```

Правило: количество consumers в группе не должно превышать количество partitions. Лишние consumers будут простаивать.

### Гарантии доставки

| Гарантия | Настройка | Описание |
|---|---|---|
| At-most-once | `enable.auto.commit=true` | Offset коммитится до обработки. Потеря возможна |
| At-least-once | Manual commit после обработки | Дубли возможны при сбое после обработки, но до commit |
| Exactly-once | `enable.idempotence=true` + транзакции | Kafka Transactions (producer + consumer в одной транзакции) |

### Пример: Producer и Consumer (PHP, librdkafka)

```php
// === Producer с идемпотентностью ===
$conf = new RdKafka\Conf();
$conf->set('bootstrap.servers', 'kafka1:9092,kafka2:9092');
$conf->set('enable.idempotence', 'true');
$conf->set('acks', 'all');               // ждать записи во все реплики
$conf->set('max.in.flight.requests.per.connection', '5');

$producer = new RdKafka\Producer($conf);
$topic = $producer->newTopic('order.created');

// Ключ определяет partition (order-1 всегда в одну partition)
$topic->produce(
    RD_KAFKA_PARTITION_UA,
    0,
    json_encode(['id' => 1, 'total' => 1000, 'items' => [...]]),
    'order-1'   // message key
);
$producer->flush(5000);

// === Consumer с manual commit ===
$conf = new RdKafka\Conf();
$conf->set('bootstrap.servers', 'kafka1:9092,kafka2:9092');
$conf->set('group.id', 'billing');
$conf->set('auto.offset.reset', 'earliest');   // начать с начала если нет offset
$conf->set('enable.auto.commit', 'false');       // ручной commit

$consumer = new RdKafka\KafkaConsumer($conf);
$consumer->subscribe(['order.created']);

while (true) {
    $msg = $consumer->consume(1000);   // timeout 1 сек
    if ($msg === null) continue;

    switch ($msg->err) {
        case RD_KAFKA_RESP_ERR_NO_ERROR:
            $data = json_decode($msg->payload, true);
            processOrder($data);
            $consumer->commit($msg);   // commit после успешной обработки
            break;
        case RD_KAFKA_RESP_ERR__PARTITION_EOF:
            // Достигнут конец partition — ждём новых сообщений
            break;
        case RD_KAFKA_RESP_ERR__TIMED_OUT:
            break;
        default:
            throw new \Exception($msg->errstr());
    }
}
```

### Retention и хранение

```bash
# Настройки retention для topic
# По времени (по умолчанию 7 дней)
retention.ms=604800000

# По размеру (без лимита по умолчанию)
retention.bytes=-1

# Compacted topics — хранят только последнее значение для каждого key
cleanup.policy=compact

# Создание topic через CLI
kafka-topics.sh --create --topic orders \
  --bootstrap-server kafka:9092 \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=2592000000   # 30 дней
```

### Kafka Connect и Stream Processing

Kafka — не просто очередь, а целая экосистема:

- **Kafka Connect** — коннекторы для интеграции с БД, файлами, S3, Elasticsearch (CDC через Debezium).
- **Kafka Streams** — библиотека для stream processing (JVM).
- **ksqlDB** — SQL-интерфейс поверх Kafka Streams.
- **Schema Registry** — управление схемами сообщений (Avro, Protobuf, JSON Schema).

```bash
# Пример: CDC с Debezium (MySQL → Kafka)
# Debezium читает binlog MySQL и пишет изменения в Kafka topic
# Topic: dbserver1.inventory.orders
```

### Когда использовать

- Потоки событий/логов, телеметрия, clickstream, аналитика.
- CDC (Change Data Capture) — синхронизация данных между системами.
- Event Sourcing — хранение всех изменений как событий.
- Много независимых потребителей, высокая нагрузка (миллионы msg/sec).
- Stream processing и построение read-моделей (CQRS).
- Replay истории — перечитать события с любого момента.

### Не лучший выбор, если

- Нужна сложная маршрутизация (topic/headers/fanout) — RabbitMQ гибче.
- Маленький проект с простыми фоновыми задачами — overhead слишком велик.
- Нужны приоритеты сообщений — Kafka не поддерживает.
- Нужна задержка доставки (delayed messages) из коробки.

## Примеры
```text
Kafka подходит для event streaming и replay истории событий.
```

## Доп. теория
- Порядок гарантирован только внутри одной partition.
- Для exactly‑once требуются транзакции и идемпотентные продюсеры.
