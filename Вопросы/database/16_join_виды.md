# JOIN виды
INNER, LEFT, RIGHT, FULL, CROSS, SELF JOIN. LEFT/RIGHT возвращают все строки из своей стороны, даже без совпадений.

Пример
```sql
SELECT u.id, u.name, o.id AS order_id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```
