## Вопрос: Kafka vs RabbitMQ (кратко)

## Простой ответ
- RabbitMQ — про маршрутизацию задач/сообщений и управление доставкой.
- Kafka — про поток событий с хранением и многими читателями.

| Тема | RabbitMQ | Kafka |
| --- | --- | --- |
| Модель | Очереди + обменники (direct/fanout/topic/headers) | Топики, партиции (append-only лог) |
| Порядок | Внутри очереди; с ретраями может мутировать | Гарантирован в партиции |
| Потребление | Сообщение забирают и ack’ают, обычно исчезает | Читают по оффсету, не удаляется сразу |
| Маршрутизация | Богатая: ключи/шаблоны/headers | Простая: ключ → партиция; сложное — в приложении |
| Доставка | At-most/At-least-once, DLX, TTL, приоритеты | At-least-once (exactly-once с идемпотентностью/транзакциями) |
| Нагрузка | Умеренная; таски/ретраи/webhooks | Высокая пропускная; события/логи/CDC/аналитика |
| Хранение | Очередь для доставки, длительное хранение не основное | Хранение по retention (время/объём), “replay” истории |
| Кейсы | Задачи, RPC, почтовый ящик, вебхуки, fanout | Event streaming, telemetry, clickstream, stream processing |

### Мини-пример выбора
- Нужна маршрутизация, ретраи, dead-letter, приоритеты → RabbitMQ.
- Нужно хранить поток событий, несколько потребителей, переигрывать историю → Kafka.

## Ответ

### Фундаментальное различие

RabbitMQ и Kafka построены на разных моделях и решают разные задачи, хотя внешне могут выглядеть похоже. RabbitMQ — это **smart broker / dumb consumer**: брокер управляет доставкой, маршрутизацией, ретраями и приоритетами. Kafka — это **dumb broker / smart consumer**: брокер хранит лог, а потребитель сам управляет своим offset и решает, что и когда читать.

### Модель данных

В RabbitMQ сообщение — это единица работы. После ACK оно удаляется из очереди. В Kafka сообщение — это запись в append-only логе, которая хранится по retention policy и не удаляется после чтения. Это фундаментальная разница: RabbitMQ удаляет обработанное, Kafka хранит всё.

### Подробная сравнительная таблица

| Критерий | RabbitMQ | Kafka |
|---|---|---|
| **Модель** | Очереди + exchange (push модель) | Append-only лог + partitions (pull модель) |
| **Протокол** | AMQP, MQTT, STOMP | Собственный бинарный протокол |
| **Порядок** | В пределах одной очереди (ретраи могут нарушить) | Гарантирован в пределах partition |
| **Маршрутизация** | Богатая: direct, fanout, topic, headers | Простая: key → partition. Сложная — в приложении |
| **Хранение** | До обработки (ACK → удаление) | По retention (дни/недели/навсегда) |
| **Replay** | Нет (сообщение удалено) | Да (перемотка offset) |
| **Потребители** | Competing consumers (один msg → один consumer) | Consumer groups + независимые группы |
| **Гарантии** | At-most/At-least-once, DLX, TTL, приоритеты | At-least-once, exactly-once (транзакции) |
| **Пропускная способность** | Тысячи–десятки тысяч msg/sec | Миллионы msg/sec |
| **Задержка** | Низкая (push модель) | Зависит от polling интервала |
| **Масштабирование** | Вертикальное + кластер (quorum queues) | Горизонтальное (добавить брокеры/partitions) |
| **Приоритеты** | Да (до 255 уровней) | Нет |
| **Delayed messages** | Да (плагин / TTL + DLX) | Нет из коробки |
| **Язык** | Erlang/OTP | Java/Scala |
| **Управление** | Web UI (Management Plugin) | CLI + сторонние UI (Conduktor, AKHQ) |

### Архитектурные паттерны

**RabbitMQ — Task Queue:**
```
Producer → Exchange (direct) → Queue → Worker 1
                                    → Worker 2
                                    → Worker 3
Каждое сообщение обрабатывает один worker. ACK → удаление.
```

**Kafka — Event Streaming:**
```
Producer → Topic (3 partitions) → Consumer Group "billing"  (свой offset)
                                → Consumer Group "analytics" (свой offset)
                                → Consumer Group "search"    (свой offset)
Каждая группа читает ВСЕ сообщения независимо.
```

### Примеры выбора по сценариям

| Сценарий | Выбор | Почему |
|---|---|---|
| Отправка email/SMS | RabbitMQ | Задача, ретраи, DLQ, приоритеты |
| Обработка заказов | RabbitMQ | Задачи с гарантией, routing по типу |
| Логирование/телеметрия | Kafka | Огромный поток, много потребителей, retention |
| CDC (Change Data Capture) | Kafka | Replay, порядок, Debezium |
| Event Sourcing / CQRS | Kafka | Хранение всех событий, rebuild state |
| Вебхуки | RabbitMQ | Ретраи с backoff, DLQ для неудачных |
| Real-time аналитика | Kafka | Stream processing, высокая пропускная |
| RPC через очередь | RabbitMQ | Request-reply pattern, correlation ID |
| Микросервисная шина | Зависит | Kafka для событий, RabbitMQ для команд |

### Можно ли использовать оба?

Да, в крупных системах часто используют оба брокера:

```
Kafka: Events (order.created, user.registered, payment.completed)
   ↓
Consumer → Публикует команды в RabbitMQ
   ↓
RabbitMQ: Commands (send.email, generate.invoice, sync.crm)
   ↓
Workers: обработка с ретраями и DLQ
```

Kafka хранит бизнес-события (аудит, replay, аналитика), а RabbitMQ управляет конкретными задачами (ретраи, приоритеты, routing).

### Итого: правило выбора

- **Нужна маршрутизация, ретраи, DLQ, приоритеты** → RabbitMQ.
- **Нужно хранить поток событий, несколько потребителей, replay, высокая пропускная** → Kafka.
- **Нужно и то, и другое** → оба, каждый для своей задачи.

## Примеры
```text
Команды/задачи → RabbitMQ, события/стриминг → Kafka.
```

## Доп. теория
- RabbitMQ хорошо для task‑queue, Kafka — для event‑streaming.
- Разный подход к хранению: удалить после ACK vs хранить по retention.
