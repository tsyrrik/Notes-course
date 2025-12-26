## Вопрос: Kafka

## Простой ответ
- Потоковая «чёрная коробка» хранит события в порядке и даёт читать их разным сервисам.
- Сообщения не исчезают сразу: можно переигрывать историю и строить витрины/аналитику.
- Раскрывается при больших объёмах событий и нескольких независимых потребителях.

## Ответ
Шина событий и лог изменений с высокой пропускной способностью.

## Основы
- Topic разбит на партиции (append-only лог); порядок гарантирован внутри партиции.
- Producer пишет с ключом → определяет партицию.
- Consumer читают по оффсету; consumer group делит партиции между инстансами.
- Сообщения не исчезают сразу: retention по времени/объёму, можно “переигрывать”.

## Пример (librdkafka, PHP)
```php
// Producer
$producer = new RdKafka\Producer();
$producer->addBrokers('kafka:9092');
$topic = $producer->newTopic('order.created');
$topic->produce(RD_KAFKA_PARTITION_UA, 0, json_encode(['id'=>1,'total'=>1000]), 'order-1');
$producer->flush(5000);

// Consumer
$conf = new RdKafka\Conf();
$conf->set('group.id', 'billing');
$conf->set('enable.auto.commit', 'false');
$consumer = new RdKafka\KafkaConsumer($conf);
$consumer->subscribe(['order.created']);
while (true) {
    $msg = $consumer->consume(1000);
    if ($msg && $msg->err === RD_KAFKA_RESP_ERR_NO_ERROR) {
        process($msg->payload);
        $consumer->commit($msg);
    }
}
```

## Когда использовать
- Потоки событий/логов, телеметрия, аналитика, CDC, “replay” истории.
- Много независимых потребителей, высокая нагрузка.
- Stream processing (Kafka Streams/Flink/ksqlDB) и построение read-моделей.
