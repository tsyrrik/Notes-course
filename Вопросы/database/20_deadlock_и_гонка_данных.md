## Вопрос: Deadlock и Race Condition

## Простой ответ
- Deadlock — два потока ждут друг друга бесконечно; лечится единым порядком блокировок и ретраями.
- Race — выигрыш гонки меняет результат; решается блокировками/версионированием/очередями.

## Ответ

### Deadlock (взаимная блокировка)

Deadlock возникает, когда **две или более транзакции** ожидают друг друга, образуя цикл зависимостей. Ни одна из них не может продолжить работу, потому что каждая держит ресурс, нужный другой. Без внешнего вмешательства такое ожидание длилось бы бесконечно.

**Классический пример**:

```
Транзакция 1:                           Транзакция 2:
BEGIN;                                   BEGIN;
UPDATE accounts SET balance=100          UPDATE accounts SET balance=200
  WHERE id = 1;  -- блокирует строку 1    WHERE id = 2;  -- блокирует строку 2

UPDATE accounts SET balance=200          UPDATE accounts SET balance=100
  WHERE id = 2;  -- ждёт строку 2 ⏳       WHERE id = 1;  -- ждёт строку 1 ⏳

-- DEADLOCK! Обе транзакции ждут друг друга
```

PostgreSQL автоматически обнаруживает deadlock через **deadlock detector** (проверка каждые `deadlock_timeout` миллисекунд, по умолчанию 1 секунда). Одна из транзакций будет принудительно прервана с ошибкой:

```
ERROR: deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5678;
blocked by process 9012.
Process 9012 waits for ShareLock on transaction 1234;
blocked by process 1234.
```

### Как предотвратить Deadlock

**1. Единый порядок блокировок (canonical ordering)**

Самый надёжный метод — всегда захватывать ресурсы в одном и том же порядке. Например, всегда обновлять строки по возрастанию ID:

```sql
-- ✅ Обе транзакции блокируют строки в порядке id ASC
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = LEAST(1, 2);  -- сначала id=1
UPDATE accounts SET balance = balance + 100 WHERE id = GREATEST(1, 2); -- потом id=2
COMMIT;
```

**2. Таймауты**

```sql
-- Ограничить время ожидания блокировки
SET lock_timeout = '5s';      -- ошибка, если не получили блокировку за 5 секунд
SET deadlock_timeout = '1s';  -- как быстро детектор обнаружит deadlock
```

**3. Уменьшение длительности транзакций**

Чем короче транзакция, тем меньше вероятность deadlock. Не выполняйте длительные вычисления или HTTP-запросы внутри транзакции.

**4. Использование одного оператора вместо нескольких**

```sql
-- ❌ Два оператора — потенциальный deadlock
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

-- ✅ Один оператор — атомарная операция, нет deadlock
UPDATE accounts SET balance = CASE
    WHEN id = 1 THEN balance - 100
    WHEN id = 2 THEN balance + 100
END
WHERE id IN (1, 2);
```

**5. Advisory locks**

Для сложных сценариев PostgreSQL предоставляет рекомендательные блокировки:

```sql
SELECT pg_advisory_lock(hashtext('transfer_' || LEAST(1,2) || '_' || GREATEST(1,2)));
-- выполняем перевод
SELECT pg_advisory_unlock(hashtext('transfer_' || LEAST(1,2) || '_' || GREATEST(1,2)));
```

### Race Condition (гонка данных)

Race condition — ситуация, когда **результат операции зависит от порядка** выполнения конкурирующих процессов, и этот порядок не контролируется. В контексте баз данных это часто проявляется как «потерянное обновление» (lost update) или неконсистентное чтение.

**Классический пример — Lost Update**:

```
Транзакция 1:                           Транзакция 2:
SELECT balance FROM accounts WHERE id=1;
-- balance = 1000
                                         SELECT balance FROM accounts WHERE id=1;
                                         -- balance = 1000
UPDATE accounts SET balance = 1000 - 100
  WHERE id = 1;  -- balance = 900
                                         UPDATE accounts SET balance = 1000 - 200
                                           WHERE id = 1;  -- balance = 800
-- Ожидаемый результат: 700
-- Фактический результат: 800 (обновление транзакции 1 потеряно!)
```

### Как предотвратить Race Condition

**1. Атомарные операции (лучший способ)**

```sql
-- ❌ Read-then-write: подвержено гонке
-- (в коде приложения: balance = query("SELECT balance ..."); update(balance - 100))

-- ✅ Атомарное обновление: одна SQL-операция
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- PostgreSQL гарантирует, что это атомарно
```

**2. Пессимистическая блокировка (SELECT FOR UPDATE)**

Явно блокирует строку, пока транзакция не завершится:

```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- строка заблокирована
-- другие транзакции с FOR UPDATE будут ждать
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
```

Варианты `FOR UPDATE`:
- `FOR UPDATE` — эксклюзивная блокировка.
- `FOR SHARE` — разделяемая блокировка (для чтения, запрещает запись другим).
- `FOR UPDATE NOWAIT` — ошибка вместо ожидания.
- `FOR UPDATE SKIP LOCKED` — пропускает заблокированные строки (для очередей).

```sql
-- Паттерн "очередь задач" с SKIP LOCKED
SELECT id, payload FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- Каждый воркер берёт свою задачу, не мешая другим
```

**3. Оптимистическая блокировка (Optimistic Locking)**

Не блокирует строку, а проверяет при обновлении, что данные не изменились. Реализуется через столбец `version` или `updated_at`:

```sql
-- Приложение прочитало: id=1, balance=1000, version=5
UPDATE accounts
SET balance = 900, version = version + 1
WHERE id = 1 AND version = 5;
-- Если affected rows = 0, значит кто-то изменил строку → retry

-- В Django/SQLAlchemy это часто реализуется автоматически через ORM
```

**4. Уровни изоляции**

Более высокие уровни изоляции (Repeatable Read, Serializable) защищают от ряда аномалий, но не от всех race condition. Serializable + retry — самый строгий вариант.

**5. Уникальные ограничения и INSERT ON CONFLICT**

```sql
-- Защита от дублирования при параллельной вставке
INSERT INTO user_actions (user_id, action, created_at)
VALUES (1, 'login', NOW())
ON CONFLICT (user_id, action) DO UPDATE
SET created_at = EXCLUDED.created_at;
```

### Сравнение Deadlock и Race Condition

| Аспект | Deadlock | Race Condition |
|--------|----------|----------------|
| Суть | Взаимное ожидание | Зависимость от порядка выполнения |
| Обнаружение | СУБД детектирует автоматически | Сложно обнаружить, проявляется как «баг с данными» |
| Результат | Ошибка, одна транзакция прерывается | Некорректные данные (потерянное обновление, дубликаты) |
| Решение | Единый порядок блокировок, retry | Атомарные операции, блокировки, оптимистический контроль |
| Опасность | Заметная (ошибка в логах) | Скрытая (данные тихо портятся) |

### Практические советы

1. **Race condition опаснее deadlock** — deadlock обнаруживается СУБД и логгируется, а гонка данных тихо портит данные.
2. **Первый шаг — атомарные SQL-операции**: `UPDATE ... SET x = x + 1` вместо SELECT → вычисление → UPDATE.
3. **FOR UPDATE SKIP LOCKED** — идеальный паттерн для очередей задач в PostgreSQL.
4. **Мониторьте deadlock-и**: `SELECT * FROM pg_stat_activity WHERE wait_event_type = 'Lock';` и логи PostgreSQL (`log_lock_waits = on`).
5. **Тестируйте конкурентность**: используйте нагрузочное тестирование (pgbench, k6) для обнаружения race condition до продакшена.

## Примеры
```sql
BEGIN;
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

## Доп. теория
- Deadlock — это ошибка «взаимного ожидания», а race condition — ошибка логики конкурентного доступа.
- В большинстве систем лучше иметь retry‑логику на `serialization_failure` и `deadlock detected`.
