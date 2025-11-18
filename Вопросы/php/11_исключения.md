# Исключения
Используйте `try/catch/finally`.
- `Throwable` — базовый интерфейс для `Exception` и `Error`.
- Можно ловить `Throwable`, чтобы перехватывать и ошибки движка.
```php
try { throw new Exception('Ошибка'); }
catch (Throwable $e) { echo $e->getMessage(); }
finally { echo 'Всегда выполняется'; }
```
