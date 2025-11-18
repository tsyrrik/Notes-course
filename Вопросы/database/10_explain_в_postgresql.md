# EXPLAIN в PostgreSQL
Простыми словами: EXPLAIN показывает, как PostgreSQL собирается выполнять запрос (или как выполнил, если ANALYZE), какие узлы/индексы/буферы использует и сколько строк ожидает.

## Варианты
- `EXPLAIN` — план без выполнения.
- `EXPLAIN ANALYZE` — выполняет и показывает фактические метрики.
- `EXPLAIN (ANALYZE, BUFFERS)` — добавляет работу с памятью/диском.
- `EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)` — максимум деталей и JSON-вывод.

Пример
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'a@b.c';
```

## Ключевые метрики
- `cost=startup..total` — оценка планировщика (не время), старта и полного прохода.
- `rows` — сколько строк ждёт узел.
- `width` — средний размер строки (байт).
- `Actual Time start..end`, `Actual Rows` — фактические метрики (только с ANALYZE).
- `Loops` — сколько раз узел выполнялся.
- `Buffers: shared hit/read/dirtied/written` — кеш/диск (только BUFFERS).

## Work_mem и сортировки/хеши
- Sort: `Sort Method: quicksort Memory: ...` или `external merge Disk: ...` — если ушло на диск, `work_mem` мало.
- Hash: `Buckets/ Batches/ Memory Usage` — если Batches > 1, хеш не влез в память.

## Основные узлы плана
- `Seq Scan` — полное сканирование таблицы.
- `Index Scan` — по индексу, потом таблица.
- `Index Only Scan` — всё из индекса.
- `Bitmap Index + Bitmap Heap Scan` — собирает адреса, читает блоки пачками.
- `Nested Loop` — хорошо при малом внешнем наборе.
- `Hash Join` — быстрый при достаточной памяти.
- `Merge Join` — по отсортированным потокам.

## Как читать
1) Найти самый дорогой по `Actual Time`.  
2) Сравнить `rows` vs `Actual Rows`.  
3) Смотреть `Buffers`: много `read` → I/O узкое место.  
4) Проверить Sort/Hash на выход в диск (work_mem).  
5) `Loops` — нет ли многократного выполнения.  
6) Тип скана — нужен ли индекс/покрытие.
