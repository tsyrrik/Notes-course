# GROUP BY
Группирует строки и агрегирует.
```sql
SELECT department, COUNT(*) FROM employees GROUP BY department;
```
Все поля вне агрегатных функций должны быть в списке GROUP BY.
