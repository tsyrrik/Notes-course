## Вопрос: GROUP BY

## Простой ответ
- Складываем строки по ключу в корзины, для каждой корзины считаем агрегаты.

## Ответ

### Что делает GROUP BY

Оператор `GROUP BY` разбивает результат запроса на **группы** по значениям одного или нескольких столбцов. Для каждой группы можно вычислить агрегатные функции: `COUNT()`, `SUM()`, `AVG()`, `MIN()`, `MAX()` и другие. Без GROUP BY агрегатная функция работает по всей таблице как по одной группе. С GROUP BY она вычисляется отдельно для каждой уникальной комбинации значений указанных столбцов.

### Основное правило

**Все столбцы в SELECT, которые не обёрнуты в агрегатную функцию, должны быть перечислены в GROUP BY.** Это логично: если вы группируете по отделу, то для каждой группы значение отдела одно, а вот конкретное имя сотрудника — неоднозначно (в группе много имён). PostgreSQL строго проверяет это правило (в отличие от MySQL, который в старых версиях позволял «свободные» столбцы).

```sql
-- ✅ Корректно: department в GROUP BY
SELECT department, COUNT(*), AVG(salary)
FROM employees
GROUP BY department;

-- ❌ Ошибка: name не в GROUP BY и не в агрегатной функции
SELECT department, name, COUNT(*)
FROM employees
GROUP BY department;
-- ERROR: column "name" must appear in the GROUP BY clause or be used in an aggregate function
```

### Порядок выполнения SQL-запроса с GROUP BY

Понимание порядка выполнения критично для правильного использования GROUP BY:

```
1. FROM / JOIN     — определяются таблицы
2. WHERE           — фильтруются строки (до группировки!)
3. GROUP BY        — строки разбиваются на группы
4. HAVING          — фильтруются группы
5. SELECT          — вычисляются выражения и агрегаты
6. DISTINCT        — удаляются дубликаты
7. ORDER BY        — сортировка
8. LIMIT / OFFSET  — ограничение результата
```

### Группировка по нескольким столбцам

Можно группировать по нескольким столбцам — каждая уникальная комбинация значений образует отдельную группу:

```sql
SELECT department, position, COUNT(*), AVG(salary)
FROM employees
GROUP BY department, position
ORDER BY department, AVG(salary) DESC;
```

| department | position | count | avg    |
|-----------|----------|-------|--------|
| IT        | Senior   | 5     | 180000 |
| IT        | Junior   | 12    | 90000  |
| Sales     | Manager  | 3     | 150000 |

### Группировка по выражениям

GROUP BY может использовать выражения, а не только имена столбцов:

```sql
-- Группировка по году регистрации
SELECT EXTRACT(YEAR FROM created_at) AS reg_year, COUNT(*)
FROM users
GROUP BY EXTRACT(YEAR FROM created_at)
ORDER BY reg_year;

-- Группировка по первой букве имени
SELECT LEFT(name, 1) AS first_letter, COUNT(*)
FROM users
GROUP BY LEFT(name, 1);
```

### GROUPING SETS, ROLLUP и CUBE

PostgreSQL поддерживает расширенные варианты группировки для аналитических отчётов:

```sql
-- ROLLUP — иерархические итоги
SELECT department, position, COUNT(*), SUM(salary)
FROM employees
GROUP BY ROLLUP(department, position);
-- Результат: группы по (department, position), итоги по (department), общий итог

-- CUBE — все возможные комбинации
SELECT department, position, COUNT(*)
FROM employees
GROUP BY CUBE(department, position);
-- Результат: все комбинации + все частичные итоги

-- GROUPING SETS — явное указание нужных группировок
SELECT department, position, COUNT(*)
FROM employees
GROUP BY GROUPING SETS ((department), (position), ());
-- Группировка по department, отдельно по position, и общий итог
```

### Агрегатные функции

| Функция | Описание | Пример |
|---------|----------|--------|
| `COUNT(*)` | Количество строк в группе | `COUNT(*)` |
| `COUNT(column)` | Количество не-NULL значений | `COUNT(email)` |
| `COUNT(DISTINCT col)` | Количество уникальных значений | `COUNT(DISTINCT city)` |
| `SUM(col)` | Сумма | `SUM(salary)` |
| `AVG(col)` | Среднее | `AVG(salary)` |
| `MIN(col)` / `MAX(col)` | Минимум / максимум | `MIN(created_at)` |
| `ARRAY_AGG(col)` | Собирает значения в массив | `ARRAY_AGG(name)` |
| `STRING_AGG(col, sep)` | Конкатенация строк | `STRING_AGG(name, ', ')` |
| `BOOL_AND` / `BOOL_OR` | Логическое И / ИЛИ | `BOOL_AND(is_active)` |

```sql
-- Практический пример: отчёт по продажам
SELECT
    DATE_TRUNC('month', order_date) AS month,
    COUNT(*) AS total_orders,
    COUNT(DISTINCT user_id) AS unique_customers,
    SUM(amount) AS revenue,
    AVG(amount) AS avg_order_value,
    STRING_AGG(DISTINCT status, ', ') AS statuses
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY DATE_TRUNC('month', order_date)
ORDER BY month;
```

### Практические советы

1. **Используйте GROUP BY с индексами** — если часто группируете по определённому столбцу, индекс на него ускорит сортировку (PostgreSQL может использовать GroupAggregate вместо HashAggregate).
2. **Фильтруйте строки в WHERE, а не в HAVING** — WHERE отсекает строки до группировки, что уменьшает объём данных для агрегации.
3. **COUNT(*) vs COUNT(column)** — `COUNT(*)` считает все строки, `COUNT(column)` пропускает NULL. Это частая причина багов.
4. **Для больших таблиц** следите за `work_mem` — HashAggregate хранит хеш-таблицу в памяти, и при большом количестве групп может уходить на диск.

## Примеры
```sql
SELECT department, COUNT(*) AS total
FROM employees
GROUP BY department
ORDER BY total DESC;
```

## Доп. теория
- GROUP BY выполняется после WHERE и до HAVING — это ключ к правильной логике запросов.
- В разных СУБД поведение «свободных» столбцов отличается; в PostgreSQL оно запрещено.
