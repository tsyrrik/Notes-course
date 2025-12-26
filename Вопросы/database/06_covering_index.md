## Вопрос: Covering / Index-Only Scan

## Простой ответ
- Если все колонки запроса лежат в индексе, движок не лезет в таблицу.
- Быстро для выборок и сортировок по этим полям.

Пример (PostgreSQL)
```sql
CREATE INDEX idx_users_covering ON users (email, name);
-- SELECT name FROM users WHERE email = 'a@b.c'; -- читается только индекс
```

## Ответ
Индекс содержит все поля, нужные запросу, поэтому таблица не читается. В PostgreSQL — Index Only Scan; в MySQL InnoDB — secondary index с нужными столбцами.
