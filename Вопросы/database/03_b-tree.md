# B-Tree устройство и скорость
Сбалансированное дерево; листья указывают на строки. Поиск/вставка/удаление — `O(log n)`. Подходит для `=, <, >, BETWEEN, ORDER BY`.

Пример создания
```sql
CREATE INDEX idx_users_name ON users (name);
```

Кейс: сортировка с лимитом
```sql
CREATE INDEX idx_orders_created_at ON orders (created_at DESC);
SELECT id FROM orders ORDER BY created_at DESC LIMIT 20;
-- план: Index Only Scan по idx_orders_created_at (если покрывает запрос)
```
