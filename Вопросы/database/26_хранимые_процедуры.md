# Хранимые процедуры/функции
SQL/PL-код на стороне БД.
Пример (PL/pgSQL):
```sql
CREATE FUNCTION add(a int, b int) RETURNS int AS $$
BEGIN RETURN a + b; END; $$ LANGUAGE plpgsql;
```
Используются для логики ближе к данным, снижения трафика, инкапсуляции.
