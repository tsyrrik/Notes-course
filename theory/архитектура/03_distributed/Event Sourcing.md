**Event Sourcing** — это способ **хранить состояние системы через поток событий**,  
а не через текущее “снимок-состояние” (snapshot) в базе данных.

---

### 1. Идея

Обычно в базе хранится текущее состояние:

|id|balance|
|---|---|
|1|1500|

При **Event Sourcing** хранится история событий, которые к нему привели:

|id|type|amount|date|
|---|---|---|---|
|1|DepositMade|+1000|2025-01-01|
|2|DepositMade|+700|2025-02-01|
|3|WithdrawalMade|-200|2025-03-01|

Баланс не хранится — он вычисляется **агрегацией событий**.  
Состояние счёта = сумма всех транзакций.

---

### 2. Пример

```php
interface Event {}

class DepositMade implements Event
{
    public function __construct(public float $amount) {}
}

class WithdrawalMade implements Event
{
    public function __construct(public float $amount) {}
}

class BankAccount
{
    private float $balance = 0;

    public function apply(Event $event): void
    {
        match (get_class($event)) {
            DepositMade::class => $this->balance += $event->amount,
            WithdrawalMade::class => $this->balance -= $event->amount,
        };
    }

    public function getBalance(): float
    {
        return $this->balance;
    }
}
```

Чтобы получить текущее состояние, просто “проигрываем” события:

```php
$events = [
    new DepositMade(1000),
    new DepositMade(700),
    new WithdrawalMade(200),
];

$account = new BankAccount();
foreach ($events as $event) {
    $account->apply($event);
}

echo $account->getBalance(); // 1500
```

---

### 3. Зачем так усложнять

- **Аудит:** можно точно узнать, _почему_ и _когда_ система пришла к состоянию.
    
- **Историчность:** состояние можно восстановить на любую дату.
    
- **Повторное воспроизведение:** удобно для отладки или реплеев (в играх).
    
- **Гибкость:** можно пересчитать агрегаты другими способами без потери данных.
    
- **Checkpoints:** для производительности можно хранить промежуточные снимки (snapshots) и применять события только после них.
    

---

### 4. Где это используется

- Банковские системы (история транзакций — источник истины).
    
- Игры (реплей — это просто список событий).
    
- Логистика (отслеживание изменений статуса заказа).
    
- Аналитика (ClickHouse/OLAP на основе потока событий).
    

---

### 5. События ≠ Команды

|Команда (Command)|Событие (Event)|
|---|---|
|Просит _сделать что-то_|Сообщает, что _уже произошло_|
|`WithdrawMoney(100)`|`MoneyWithdrawn(100)`|

---

### 6. Event Store

Хранилище событий — это просто **append-only лог**, куда добавляются факты:  
никаких апдейтов, только новые строки.

Может быть реализовано:

- SQL-таблицей `events`
    
- Kafka topic
    
- Специализированным хранилищем (EventStoreDB, Axon, etc.)
    

---

### 7. Rebuilding state

Чтобы получить состояние агрегата:

1. Загружаем все события по ID сущности.
    
2. Применяем их в порядке времени.
    
3. Получаем финальное состояние.
    

Если данных слишком много — используем snapshot (чекпойнт):

```
[события 1..1000] -> snapshot -> [события 1001..∞]
```

