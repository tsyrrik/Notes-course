## Вопрос: Covering / Index-Only Scan

## Простой ответ
- Если все колонки запроса лежат в индексе, движок не лезет в таблицу.
- Быстро для выборок и сортировок по этим полям.

Примеры — в конце.

## Ответ

### Что такое покрывающий индекс

Покрывающий индекс (covering index) — это индекс, который содержит все столбцы, необходимые для выполнения запроса: столбцы из SELECT, WHERE, ORDER BY и JOIN. Когда СУБД может получить все данные из индекса, она не обращается к основной таблице (heap). Это называется Index Only Scan в PostgreSQL и Using index в MySQL.

Обращение к heap — это random I/O, одна из самых дорогих операций. Покрывающий индекс полностью устраняет её, что может ускорить запрос в 2-10 раз по сравнению с обычным Index Scan.

### Как это работает

При обычном Index Scan происходит два шага:
1. Поиск ключа в индексе -> получение tuple ID (указателя на строку).
2. Обращение к heap (таблице) по tuple ID -> чтение полных данных строки.

При Index Only Scan:
1. Поиск ключа в индексе -> все нужные данные уже здесь. Готово.

```sql
-- Обычный индекс
CREATE INDEX idx_users_email ON users (email);

-- Запрос требует столбец name — придётся лезть в heap
SELECT name FROM users WHERE email = 'a@b.c';
-- Index Scan (не Index Only!)

-- Покрывающий индекс
CREATE INDEX idx_users_email_name ON users (email, name);

-- Теперь все столбцы в индексе
SELECT name FROM users WHERE email = 'a@b.c';
-- Index Only Scan!
```

### Синтаксис INCLUDE (PostgreSQL 11+)

В PostgreSQL 11 появился оператор `INCLUDE`, который позволяет добавлять столбцы в индекс, не включая их в ключ. Эти столбцы хранятся только в листовых узлах и не участвуют в поиске, но доступны для покрытия запроса.

```sql
-- Без INCLUDE: оба столбца — часть ключа B-Tree
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
-- Индекс сортирован по (user_id, status)

-- С INCLUDE: только user_id — ключ, status — дополнительные данные
CREATE INDEX idx_orders_user_incl_status ON orders (user_id) INCLUDE (status);
-- Индекс сортирован только по user_id, но status доступен для Index Only Scan
```

Преимущества `INCLUDE`:
- Не увеличивает глубину B-Tree (included-столбцы не в ключе).
- Позволяет включать типы данных, не поддерживающие B-Tree сортировку (например, jsonb).
- Индекс может быть меньше и быстрее обновляться.

### Покрывающие индексы в MySQL InnoDB

В InnoDB первичный ключ — это кластеризованный индекс (данные таблицы хранятся в порядке PK). Вторичные индексы хранят значение первичного ключа, а не указатель на строку. Поэтому вторичный индекс автоматически «покрывает» столбцы PK.

```sql
-- MySQL InnoDB
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT,
  status VARCHAR(20),
  total DECIMAL(10,2),
  INDEX idx_user_status (user_id, status)
);

-- Этот запрос — Index Only (Using index в EXPLAIN)
-- потому что id (PK) автоматически есть во вторичном индексе
EXPLAIN SELECT id, status FROM orders WHERE user_id = 42;
-- Extra: Using index
```

### Visibility Map в PostgreSQL

В PostgreSQL Index Only Scan имеет важную особенность: из-за MVCC (Multi-Version Concurrency Control) индекс не хранит информацию о видимости строк. СУБД должна проверить visibility map — битовую карту, показывающую, все ли строки на данной странице видимы всем транзакциям. Если страница не «all-visible», PostgreSQL обращается к heap для проверки видимости.

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT name FROM users WHERE email = 'a@b.c';
-- Index Only Scan using idx_users_email_name
--   Heap Fetches: 5    <- 5 раз пришлось обратиться к heap

-- Обновим visibility map
VACUUM users;

-- Повторно
EXPLAIN (ANALYZE, BUFFERS) SELECT name FROM users WHERE email = 'a@b.c';
-- Index Only Scan using idx_users_email_name
--   Heap Fetches: 0    <- идеально, heap не читается
```

Регулярный `VACUUM` (или autovacuum) критически важен для эффективности Index Only Scan.

### Когда покрывающий индекс полезен

1. **Часто выполняемые запросы** с фиксированным набором столбцов — API-эндпоинты, дашборды.
2. **Запросы COUNT** — `SELECT COUNT(*) FROM orders WHERE user_id = 42` с покрывающим индексом не трогает heap.
3. **Пагинация** — `SELECT id, title FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 100`.
4. **JOIN с lookup** — когда из joined-таблицы нужно только 1-2 столбца.

### Когда НЕ стоит создавать покрывающий индекс

- Запрос использует `SELECT *` — покрыть все столбцы нереально, а индекс будет огромным.
- Столбец часто обновляется — каждый UPDATE потребует обновления индекса.
- Слишком много разных запросов к таблице — невозможно покрыть все, а множество больших индексов замедлят запись.

### Практический пример оптимизации

```sql
-- Исходный запрос (частый, на дашборде)
SELECT user_id, count(*), sum(total)
FROM orders
WHERE status = 'completed' AND created_at > now() - interval '30 days'
GROUP BY user_id;

-- Без покрывающего индекса: Seq Scan или Bitmap Scan + Heap Scan
-- С покрывающим индексом:
CREATE INDEX idx_orders_covering ON orders (status, created_at)
  INCLUDE (user_id, total);

-- Теперь: Index Only Scan, все данные из индекса
-- Ускорение в 5-10 раз на таблице с миллионами строк
```

## Примеры
```sql
CREATE INDEX idx_users_covering ON users (email, name);
-- SELECT name FROM users WHERE email = 'a@b.c'; -- читается только индекс
```

## Доп. теория
- В PostgreSQL Index Only Scan зависит от visibility map — без VACUUM могут быть heap fetches.
- В MySQL вторичные индексы всегда содержат PK, поэтому часть запросов «покрываются» автоматически.
