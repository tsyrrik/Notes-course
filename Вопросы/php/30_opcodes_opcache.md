## Вопрос: OpCodes и OpCache
Версия: PHP 8.4; где есть исторические детали — см. по тексту.

## Простой ответ
PHP-файл парсится в AST, компилируется в opcodes (байткод), которые исполняет Zend VM. OpCache кеширует opcodes между запросами, JIT (PHP 8) может превращать “горячие” opcodes в машинный код.

## Ответ

### Путь выполнения PHP-кода

```
PHP-файл → Lexer (токены) → Parser (AST) → Compiler (opcodes) → Zend VM (выполнение)
```

1. **Lexer** — разбивает исходный код на токены (`T_FUNCTION`, `T_VARIABLE` и т.д.)
2. **Parser** — строит абстрактное синтаксическое дерево (AST)
3. **Compiler** — преобразует AST в opcodes (байткод)
4. **Zend VM** — выполняет opcodes

Без OpCache этот цикл повторяется **при каждом запросе**.

### OpCache

OpCache — расширение, кеширующее скомпилированные opcodes в разделяемой памяти. При повторном запросе шаги 1-3 пропускаются.

**Ключевые настройки php.ini:**

```ini
opcache.enable=1                    # включить
opcache.memory_consumption=256      # МБ для кеша (default 128)
opcache.max_accelerated_files=20000 # макс. файлов (ближайшее простое число)
opcache.validate_timestamps=1       # проверять изменения файлов
opcache.revalidate_freq=2           # интервал проверки (сек)

# Production:
opcache.validate_timestamps=0       # НЕ проверять — нужен перезапуск PHP при деплое
opcache.preload=/app/preload.php    # предзагрузка (PHP 7.4+)
```

### Preloading (PHP 7.4+)

Загружает указанные файлы в память при старте PHP-FPM. Они остаются в памяти до перезапуска:

```php
// preload.php
require_once __DIR__ . '/vendor/autoload.php';
// Загружаем часто используемые классы
$classes = [App\Models\User::class, App\Services\AuthService::class];
foreach ($classes as $class) {
    class_exists($class); // триггерит автозагрузку
}
```

### JIT (Just-In-Time, PHP 8.0+)

JIT компилирует «горячие» opcodes в машинный код процессора:

```ini
opcache.jit_buffer_size=64M
opcache.jit=1255   # tracing JIT (рекомендуется)
```

**Когда JIT даёт эффект:**
- CPU-bound задачи: математика, обработка изображений, парсинг
- Долгоживущие процессы (CLI-воркеры)

**Когда JIT бесполезен:**
- IO-bound приложения (веб-API, CRUD) — основное время уходит на БД/сеть
- Типичные веб-приложения ускоряются на 1-3%, что незаметно

## Примеры

### Просмотр opcodes

```bash
# Через расширение VLD
php -d vld.active=1 script.php

# Через phpdbg
phpdbg -p script.php

# Онлайн: 3v4l.org (показывает opcodes)
```

```php
// Пример: что видит Zend VM
$a = 1 + 2;
// Opcodes:
// ASSIGN $a, 3  ← компилятор оптимизирует константное выражение
```

## Доп. теория
- OpCache ускоряет CPU‑часть PHP, но не заменяет кэш запросов к БД/HTTP.
- JIT особенно полезен в CPU‑bound задачах; в типичных CRUD‑приложениях выигрыш минимален.
