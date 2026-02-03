## Вопрос: Экспорт больших таблиц в CSV

## Простой ответ
- Делите выгрузку на части и стримьте, чтобы не ложиться в память/транзакцию.
- COPY быстрее, чем SELECT через клиент.

## Ответ

### Почему экспорт больших таблиц — это проблема

При экспорте таблицы с миллионами или миллиардами строк возникают несколько проблем: **нехватка оперативной памяти** (если пытаться загрузить всё в память), **длинная транзакция** (блокирует VACUUM, увеличивает bloat), **нагрузка на диск** и сеть, а также **отсутствие возможности возобновить** прерванный экспорт. Поэтому наивный подход `SELECT * FROM huge_table` через клиент — это путь к проблемам.

### Способ 1: COPY — самый быстрый в PostgreSQL

Команда `COPY` — встроенный механизм PostgreSQL для массового экспорта/импорта. Он работает **на стороне сервера**, минуя клиентский протокол, и значительно быстрее, чем выгрузка через SELECT.

```sql
-- Экспорт всей таблицы
COPY users TO '/tmp/users.csv' WITH (FORMAT CSV, HEADER true);

-- Экспорт с фильтрацией (подзапрос)
COPY (SELECT id, name, email FROM users WHERE created_at > '2024-01-01')
TO '/tmp/active_users.csv' WITH (FORMAT CSV, HEADER true, DELIMITER ';');

-- Экспорт с NULL-значениями как пустыми строками
COPY users TO '/tmp/users.csv' WITH (FORMAT CSV, HEADER true, NULL '');
```

**Ограничение**: `COPY TO` записывает файл на **сервер**, а не на клиент. Если нужен файл на клиенте, используйте `\copy` в psql:

```bash
psql -d mydb -c "\copy users TO '/local/path/users.csv' WITH (FORMAT CSV, HEADER true)"
```

`\copy` работает через клиентский протокол и пишет файл на стороне клиента.

### Способ 2: Батчевый экспорт (для очень больших таблиц)

Для таблиц с сотнями миллионов строк имеет смысл разбить экспорт на части (batches), чтобы не держать длинную транзакцию и иметь возможность возобновления:

```sql
-- Батч по диапазону ID
COPY (SELECT * FROM orders WHERE id BETWEEN 1 AND 1000000)
TO '/tmp/orders_batch_1.csv' WITH CSV HEADER;

COPY (SELECT * FROM orders WHERE id BETWEEN 1000001 AND 2000000)
TO '/tmp/orders_batch_2.csv' WITH CSV HEADER;
```

Автоматизация через скрипт:

```bash
#!/bin/bash
TABLE="orders"
BATCH_SIZE=1000000
MAX_ID=$(psql -t -c "SELECT MAX(id) FROM $TABLE")

for ((start=1; start<=MAX_ID; start+=BATCH_SIZE)); do
    end=$((start + BATCH_SIZE - 1))
    psql -c "\copy (SELECT * FROM $TABLE WHERE id BETWEEN $start AND $end) TO '/tmp/${TABLE}_${start}_${end}.csv' WITH CSV HEADER"
done
```

### Способ 3: Серверный курсор (streaming)

Для приложений на Python, Java и других языках используйте серверный курсор, который читает данные порциями, не загружая всё в память:

```python
import psycopg2

conn = psycopg2.connect("dbname=mydb")
cursor = conn.cursor(name='export_cursor')  # name делает его серверным
cursor.itersize = 10000  # размер порции

cursor.execute("SELECT * FROM huge_table")

with open('output.csv', 'w') as f:
    for row in cursor:
        f.write(','.join(str(col) for col in row) + '\n')

cursor.close()
conn.close()
```

### Способ 4: Экспорт с параллелизмом

Для максимальной скорости можно экспортировать параллельно несколько диапазонов:

```bash
# Параллельный экспорт через GNU parallel
seq 0 9 | parallel -j4 "psql -c \"\copy (SELECT * FROM orders WHERE id % 10 = {}) TO '/tmp/orders_part_{}.csv' WITH CSV HEADER\""
```

Или через `pg_dump` для полного дампа:

```bash
# Параллельный дамп в custom-формате (сжатый)
pg_dump -Fd -j 4 -t huge_table -f /tmp/dump_dir mydb
```

### Сжатие на лету

CSV-файлы хорошо сжимаются (обычно в 5-10 раз). Можно сжимать при экспорте:

```bash
# psql + gzip
psql -c "\copy users TO STDOUT WITH CSV HEADER" | gzip > users.csv.gz

# psql + zstd (быстрее gzip, лучше сжатие)
psql -c "\copy users TO STDOUT WITH CSV HEADER" | zstd > users.csv.zst

# PostgreSQL 16+: встроенная поддержка сжатия в COPY
COPY users TO PROGRAM 'gzip > /tmp/users.csv.gz' WITH CSV HEADER;
```

### Мониторинг и оценка времени

```sql
-- Оценить размер данных до экспорта
SELECT pg_size_pretty(pg_total_relation_size('huge_table'));
SELECT COUNT(*) FROM huge_table;

-- Посмотреть прогресс длительного запроса (PostgreSQL 14+)
SELECT * FROM pg_stat_progress_copy;
```

### Альтернативные форматы

| Формат | Плюсы | Минусы |
|--------|-------|--------|
| CSV | Универсальный, читаемый | Нет типизации, проблемы с разделителями |
| TSV | Меньше проблем с кавычками | Не все инструменты поддерживают |
| BINARY | Самый быстрый | Непереносимый, нечитаемый |
| Parquet | Колоночный, сжатый, типизированный | Нужен внешний инструмент (DuckDB, Spark) |

```sql
-- Экспорт в бинарном формате PostgreSQL (самый быстрый)
COPY users TO '/tmp/users.bin' WITH (FORMAT BINARY);

-- Экспорт через DuckDB в Parquet (из PostgreSQL)
-- duckdb -c "COPY (SELECT * FROM postgres_scan('dbname=mydb', 'public', 'users')) TO 'users.parquet'"
```

### Практические советы

1. **Для таблиц < 10 млн строк** — простой `\copy ... WITH CSV HEADER` обычно достаточно (секунды-минуты).
2. **Для таблиц > 100 млн строк** — батчевый экспорт с возможностью возобновления.
3. **Не экспортируйте с продакшн-сервера** напрямую — используйте реплику для чтения.
4. **Всегда указывайте конкретные столбцы** вместо `SELECT *` — избегайте выгрузки blob-столбцов и внутренних полей.
5. **Проверяйте результат**: `wc -l file.csv` (число строк) и `head -5 file.csv` (формат).
6. **Учитывайте часовой пояс** — timestamp может экспортироваться по-разному в зависимости от `timezone` сессии.

## Примеры
```bash
psql -d mydb -c "\\copy users TO 'users.csv' WITH (FORMAT CSV, HEADER true)"
```

## Доп. теория
- Для больших выгрузок используйте реплику чтения, чтобы не нагружать primary.
- CSV удобен, но для аналитики часто лучше Parquet.
