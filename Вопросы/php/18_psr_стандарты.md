# PSR стандарты
- PSR-1 — базовые правила кодирования.
- PSR-3 — интерфейс логгирования.
- PSR-4 — автозагрузка по неймспейсам.
- PSR-6 — кеш.
- PSR-7 — HTTP messages.
- PSR-11 — контейнер зависимостей.
- PSR-12 — стиль и форматирование.

Пример автозагрузки PSR-4 (composer.json)
```json
{
  "autoload": { "psr-4": { "App\\\\": "src/" } }
}
```
```bash
composer dump-autoload
```
