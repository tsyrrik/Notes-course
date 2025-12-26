## Вопрос: Hash индексы и скорость

## Простой ответ
- Хороши только для `=` по конкретному значению.
- Бесполезны для `<`, `>`, `ORDER BY` и префиксных LIKE.

Пример (PostgreSQL)
```sql
CREATE INDEX idx_sessions_token_hash ON sessions USING hash (token);
SELECT * FROM sessions WHERE token = 'abc123';
```

Кейс: быстрые lookup по токену/ключу API
```sql
CREATE INDEX idx_api_keys_hash ON api_keys USING hash (key);
EXPLAIN SELECT id FROM api_keys WHERE key = 'k_123';
-- план: Index Scan using idx_api_keys_hash
```

## Ответ
В хэш-таблице поиск точных совпадений — `O(1)`. Не работают для диапазонов и сортировок; возможны коллизии.
