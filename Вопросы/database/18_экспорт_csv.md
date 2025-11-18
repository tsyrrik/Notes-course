# Экспорт больших таблиц в CSV
Используйте батчи/стриминг. PostgreSQL: `COPY (SELECT ...) TO '/path/file.csv' WITH CSV HEADER;` Можно параллелить (pg_dump -j), сжимать (gzip/zstd).
