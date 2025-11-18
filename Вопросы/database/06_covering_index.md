# Covering / Index-Only Scan
Индекс содержит все поля, нужные запросу, поэтому таблица не читается. В PostgreSQL — Index Only Scan; в MySQL InnoDB — secondary index с нужными столбцами.

Пример (PostgreSQL)
```sql
CREATE INDEX idx_users_covering ON users (email, name);
-- SELECT name FROM users WHERE email = 'a@b.c'; -- читается только индекс
```
