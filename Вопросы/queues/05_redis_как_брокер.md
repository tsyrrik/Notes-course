## Вопрос: Redis как брокер и очередь

## Простой ответ
Redis - это высокопроизводительное хранилище данных в памяти (in-memory key-value) с опциональной персистентностью на диск. Часто используется как кэш, хранилище с низкой задержкой, брокер сообщений и очередь задач.

## Ответ

### Что такое Redis

Redis (Remote Dictionary Server) — это высокопроизводительное in-memory хранилище данных, поддерживающее множество структур данных: строки, хэши, списки, множества, sorted sets, streams, bitmaps и HyperLogLog. Работает в оперативной памяти, что обеспечивает задержки менее 1 мс. Опциональная персистентность (RDB snapshots и/или AOF) позволяет сохранять данные на диск.

### Зачем используют Redis

- **Кэширование** — ответы API, сессии, результаты запросов (TTL на ключах).
- **Rate Limiting** — ограничение частоты запросов через INCR + EXPIRE.
- **Счётчики и рейтинги** — INCR, Sorted Sets для leaderboard.
- **Pub/Sub** — уведомления и сигналы в реальном времени.
- **Очереди задач** — Lists (LPUSH/BRPOP) и Streams (XADD/XREADGROUP).
- **Распределённые блокировки** — Redlock алгоритм через SET NX EX.
- **Хранение временных данных** — сессии, OTP-коды, токены с TTL.

### Три способа реализации очередей в Redis

#### 1. Списки (Lists) — простая FIFO-очередь

Самый простой способ: LPUSH добавляет задачу в начало, BRPOP блокирующе извлекает из конца. Подходит для фоновых задач, когда потеря пары сообщений при сбое некритична.

```redis
# Продюсер
LPUSH tasks '{"type":"email","to":"user@test.com","template":"welcome"}'
LPUSH tasks '{"type":"email","to":"admin@test.com","template":"report"}'

# Консьюмер (блокирующее извлечение, timeout 0 = бесконечно)
BRPOP tasks 0
# Возвращает: ["tasks", "{...payload...}"]
```

Для at-least-once доставки используется `RPOPLPUSH` (или `LMOVE` в Redis 6.2+), который атомарно перемещает сообщение в очередь обработки:

```redis
# Атомарно: извлечь из tasks, положить в processing
LMOVE tasks processing RIGHT LEFT

# После успешной обработки — удалить из processing
LREM processing 1 '{"type":"email",...}'

# Если воркер упал — сообщение остаётся в processing,
# мониторинг может вернуть его обратно в tasks
```

#### 2. Streams — продвинутая очередь с consumer groups

Redis Streams (с Redis 5.0) — это append-only лог, похожий на Kafka partitions. Поддерживает consumer groups, offset tracking и повторную доставку неподтверждённых сообщений.

```redis
# Добавление записей
XADD orders * action created order_id 1001 total 5000
XADD orders * action created order_id 1002 total 3000

# Создание consumer group ($ = с текущего момента, 0 = с начала)
XGROUP CREATE orders billing $ MKSTREAM

# Чтение consumer'ом из группы
XREADGROUP GROUP billing consumer-1 BLOCK 5000 COUNT 10 STREAMS orders >
# > означает: только новые, ещё не доставленные сообщения

# Подтверждение обработки
XACK orders billing 1234567890-0

# Просмотр необработанных (pending) сообщений
XPENDING orders billing - + 10

# Перевыдача «зависших» сообщений (если consumer упал)
XCLAIM orders billing consumer-2 60000 1234567890-0
# 60000 ms = перевыдать, если idle > 1 минуты

# Обрезка стрима (retention)
XTRIM orders MAXLEN ~ 10000      # примерно 10000 записей
XTRIM orders MINID 1234567890-0  # удалить записи старше ID
```

#### 3. Pub/Sub — широковещательные сигналы

Pub/Sub — это fire-and-forget: нет истории, нет подтверждений. Если subscriber не подключён — сообщение теряется. Используется для сигналов и уведомлений, но не для очередей задач.

```redis
# Подписка
SUBSCRIBE notifications
PSUBSCRIBE events.*    # по шаблону

# Публикация
PUBLISH notifications '{"type":"cache_clear","key":"users"}'
PUBLISH events.order.created '{"id":1}'
```

### Сравнение подходов

| Критерий | Lists | Streams | Pub/Sub |
|---|---|---|---|
| Модель | Point-to-point | Consumer groups | Broadcast |
| Порядок | FIFO | По ID (append-only) | Нет гарантии |
| Подтверждение (ACK) | Вручную (LMOVE) | XACK | Нет |
| Повторная доставка | Через LMOVE | XCLAIM/XAUTOCLAIM | Нет |
| История | Нет | Да (XRANGE, XREAD) | Нет |
| Consumer groups | Нет | Да | Нет |
| Блокирующее чтение | BRPOP | XREADGROUP BLOCK | SUBSCRIBE (блокирующее) |

### Пример: очередь задач на PHP (predis)

```php
use Predis\Client;

$redis = new Client(['host' => 'redis', 'port' => 6379]);

// === Producer ===
$redis->xadd('jobs', '*', [
    'type'    => 'send_email',
    'payload' => json_encode(['to' => 'user@test.com', 'template' => 'welcome']),
]);

// === Consumer с consumer group ===
// Создание группы (один раз)
try {
    $redis->xgroup('CREATE', 'jobs', 'workers', '0', true); // MKSTREAM
} catch (\Exception $e) {
    // группа уже существует
}

while (true) {
    $messages = $redis->xreadgroup('workers', 'worker-1', ['jobs' => '>'], 10, 5000);

    if ($messages) {
        foreach ($messages['jobs'] as $id => $fields) {
            try {
                $payload = json_decode($fields['payload'], true);
                processJob($fields['type'], $payload);
                $redis->xack('jobs', 'workers', $id);
            } catch (\Exception $e) {
                // Не делаем ACK — сообщение останется в pending
                logger()->error("Job failed: {$id}", ['error' => $e->getMessage()]);
            }
        }
    }
}
```

### Персистентность и надёжность

Redis хранит данные в памяти, поэтому при сбое без персистентности данные теряются:

| Режим | Описание | Потеря данных |
|---|---|---|
| **Без персистентности** | Только RAM | Всё при перезапуске |
| **RDB** | Периодические snapshots | Данные между snapshot'ами |
| **AOF** | Лог каждой операции | Минимальная (зависит от fsync) |
| **RDB + AOF** | Оба режима | Минимальная |

```bash
# redis.conf
appendonly yes
appendfsync everysec     # fsync каждую секунду (баланс скорости и надёжности)
```

### Redis vs специализированные брокеры

| Критерий | Redis | RabbitMQ | Kafka |
|---|---|---|---|
| Задержка | < 1 мс | 1-10 мс | 5-50 мс |
| Пропускная | Сотни тысяч msg/sec | Десятки тысяч | Миллионы |
| Маршрутизация | Нет (только key/channel) | Богатая (exchange types) | Простая (key → partition) |
| DLQ | Нет из коробки | Да (DLX) | Нет из коробки |
| Хранение | Ограничено RAM | До обработки | По retention |
| Replay | Streams: ограничено | Нет | Полный |
| Сложность | Минимальная | Средняя | Высокая |

### Когда использовать Redis как брокер

- Уже есть Redis в стеке (кэш/сессии) — не нужна отдельная инфраструктура.
- Фоновые задачи с низкой задержкой и умеренным объёмом.
- Простые очереди для Laravel Horizon, Sidekiq, Bull.
- Сигналы/уведомления через Pub/Sub.
- Прототипирование, MVP, когда полноценный брокер избыточен.

### Когда НЕ использовать

- Данные не помещаются в RAM — Kafka/RabbitMQ хранят на диске.
- Нужна сложная маршрутизация — RabbitMQ.
- Нужен длительный replay и event sourcing — Kafka.
- Строгие гарантии доставки в распределённой системе — специализированный брокер надёжнее.

## Примеры
```text
Redis подходит для простых очередей и низкой задержки, когда Redis уже есть в стеке.
```

## Доп. теория
- Streams дают более надёжную модель очереди, чем списки.
- Redis‑очереди ограничены памятью и требуют аккуратной персистентности.
