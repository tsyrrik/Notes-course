## Вопрос: WHERE vs HAVING

## Простой ответ
- WHERE отсекает строки перед агрегацией, HAVING — уже готовые группы.

Пример
```sql
SELECT department, COUNT(*)
FROM employees
WHERE salary > 1000      -- фильтрация строк
GROUP BY department
HAVING COUNT(*) > 5;     -- фильтрация сгруппированных результатов
```

## Ответ
- `WHERE` — фильтрация строк до группировки.
- `HAVING` — фильтрация групп после агрегации.
