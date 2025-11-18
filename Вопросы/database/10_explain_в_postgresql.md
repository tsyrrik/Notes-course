# EXPLAIN в PostgreSQL
- `EXPLAIN` — план без выполнения.
- `EXPLAIN ANALYZE` — выполняет и показывает фактические показатели.
- `EXPLAIN (ANALYZE, BUFFERS)` — добавляет статистику по кешу/диску.
Можно форматировать в JSON и включать VERBOSE.

Пример
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'a@b.c';
```

Что выводит (основное)
- Node Type: тип операции (Seq Scan, Index Scan, Hash Join, Nested Loop...).
- Relation Name / Index Name: по какой таблице/индексу идёт операция.
- Startup/Total Cost: оценка планировщика (не время, а “стоимость”).
- Rows: ожидаемое количество строк.
- Width: средний размер строки (байт).
- Actual Time (start/total), Actual Rows: фактические метрики (только с ANALYZE).
- Loops: сколько раз узел выполнялся (вложенные циклы).
- Buffers: shared hit/read/dirtied/written (только с BUFFERS).
