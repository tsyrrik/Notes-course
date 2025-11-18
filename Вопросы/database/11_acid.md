# ACID
- Atomicity — всё или ничего (при ошибке откат).
- Consistency — переход в валидное состояние (ограничения, FK, CHECK).
- Isolation — транзакции не мешают (уровни изоляции).
- Durability — данные сохраняются после коммита (WAL и пр.).

## Простыми словами
- Транзакция либо проходит целиком, либо откатывается, не нарушает ограничения, изолирована от чужих незавершённых данных и не теряется после коммита.

Пример транзакции
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- либо оба UPDATE, либо ROLLBACK
```
