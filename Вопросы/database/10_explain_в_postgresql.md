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
