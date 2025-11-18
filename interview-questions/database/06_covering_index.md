# Covering / Index-Only Scan
Индекс содержит все поля, нужные запросу, поэтому таблица не читается. В PostgreSQL — Index Only Scan; в MySQL InnoDB — secondary index с нужными столбцами.
