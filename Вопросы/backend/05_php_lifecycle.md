## Вопрос: Lifecycle PHP-запроса

## Простой ответ
- nginx/Apache держит фронт, PHP-FPM отдаёт запрос воркеру, воркер исполняет код и возвращает ответ.
- Код компилируется в опкоды и кешируется в OpCache, чтобы не парсить каждый раз.
- Воркер после ответа остаётся тёплым в пуле; очищаются только данные конкретного запроса.

## Ответ
Сводная схема:
1. Клиент отправляет HTTP-запрос → веб-сервер (nginx / Apache).
2. Веб-сервер передаёт .php-запрос в PHP-FPM (через FastCGI).
3. FPM выбирает свободный воркер, инициализирует SAPI и суперглобальные ($_GET, $_POST, $_SERVER и т.д.).
4. PHP парсит код → Tokenizer → AST (транслируется)→ Opcodes, кеширует в OpCache.
5. Zend VM исполняет опкоды (при необходимости — JIT компиляция).
6. Пользовательский код выполняется: файлы, БД, логика, вывод.
7. Результат возвращается через FPM → веб-сервер → клиент.
8. Воркер возвращается в пул; OpCache и расширения остаются в памяти.

Схема: клиент → веб-сервер (nginx/Apache) → PHP-FPM → Zend Engine (Tokenizer → AST → Opcodes → OpCache → Zend VM/JIT) → ответ → клиент. Воркеры остаются в пуле, OpCache и расширения живут между запросами.

## Кратко по шагам
| Этап | Что происходит |
| --- | --- |
| 1 | nginx/Apache передаёт .php в PHP-FPM (FastCGI) |
| 2 | FPM выбирает воркер |
| 3 | Инициализация окружения: php.ini, параметры пула, SAPI, суперглобальные |
| 4 | Парсинг кода: токены → AST → опкоды |
| 5 | Опкоды кладутся/читаются из OpCache |
| 6 | Zend VM исполняет байткод |
| 7 | (опц.) JIT компилирует горячие трассы |
| 8 | Выполняется пользовательский код/автозагрузка |
| 9 | Ответ возвращается через FPM → веб-сервер |
| 10 | Воркер остаётся в пуле; OpCache/расширения — в памяти |

## Подробно
1) Запуск через PHP-FPM: master слушает сокет/порт, выбирает свободный воркер.  
2) Воркер уже прогрет: Zend Engine загружен, php.ini прочитан при старте пула.  
3) Инициализация SAPI (fpm-fcgi), суперглобальных (`$_SERVER`, `$_GET`, `$_POST`, `$_COOKIE`, `$_FILES`, `$_ENV`, `$_REQUEST`, сессия при auto_start).  
4) Токенизация → AST → опкоды.  
5) OpCache: пропускает парсер/компилятор, если байткод в кеше (`opcache.enable=1`).  
6) Исполнение в Zend VM — стековая машина; ошибки через `zend_error_cb`.  
7) JIT (PHP 8+): компилирует горячие трассы в машинный код, прирост на CPU-heavy задачах.  
8) Autoload/require: при первом обращении файл компилируется и (если включено) кешируется в OpCache.  
9) Shutdown: вызываются `register_shutdown_function()`, освобождаются переменные запроса, закрывается вывод.  
10) Ответ идёт назад по FastCGI → веб-сервер → клиент; воркер возвращён в пул.

## Что сохраняется между запросами
- Интерпретатор, Zend Engine, расширения.
- OpCache (скомпилированные опкоды).
- Настройки php.ini, параметры пула FPM.

## Что очищается каждый запрос
- Пользовательские переменные/объекты.
- Суперглобальные, буферы вывода/ошибок, контекст запроса.

## Визуально
```
[nginx] -> FastCGI -> [php-fpm master] -> [php-fpm worker]
  Zend Engine: Tokenizer -> AST -> Opcodes -> OpCache
  Zend VM -> (JIT) -> машинный код -> Output -> FastCGI -> nginx -> клиент
```

---

### Детальный разбор каждого этапа

#### Этап 1: PHP-FPM Master Process

При запуске PHP-FPM (`systemctl start php-fpm`) master-процесс выполняет инициализацию, которая происходит **один раз** и больше не повторяется:

```text
1. Чтение основного php.ini
2. Загрузка всех расширений (модулей):
   - Ядро: Core, date, pcre, Reflection, SPL, standard
   - Внешние: pdo, pdo_mysql, mbstring, curl, openssl, json, xml
   - Пользовательские: redis, imagick, xdebug
3. Вызов MINIT (Module Init) для каждого расширения
4. Чтение конфигурации пулов (/etc/php/8.3/fpm/pool.d/*.conf)
5. Создание сокета/порта для каждого пула
6. Fork воркеров согласно pm.start_servers
```

**MINIT** (Module Initialization) — это функция, которую каждое расширение реализует для однократной инициализации. Например, расширение PDO в MINIT регистрирует класс `PDO`, расширение `opcache` аллоцирует shared memory для кэша.

```c
// Пример: что делает расширение в MINIT (упрощённо)
PHP_MINIT_FUNCTION(pdo) {
    zend_class_entry ce;
    INIT_CLASS_ENTRY(ce, "PDO", pdo_methods);
    pdo_ce = zend_register_internal_class(&ce);
    return SUCCESS;
}
```

#### Этап 2: Воркер получает запрос (RINIT)

Когда FPM master получает запрос по FastCGI, он передаёт его свободному воркеру. Воркер вызывает **RINIT** (Request Init) для каждого расширения:

```text
RINIT для каждого расширения:
├── session: инициализирует обработчик сессий
├── output: сбрасывает буферы вывода
├── error: сбрасывает error_reporting
└── ...

Затем инициализируются суперглобальные:
├── $_SERVER  — из FCGI_PARAMS (REMOTE_ADDR, REQUEST_URI, ...)
├── $_GET     — парсинг QUERY_STRING
├── $_POST    — парсинг FCGI_STDIN (при Content-Type: form/json)
├── $_COOKIE  — парсинг HTTP_COOKIE
├── $_FILES   — парсинг multipart/form-data
├── $_ENV     — переменные окружения процесса
├── $_REQUEST — merge $_GET + $_POST + $_COOKIE (зависит от request_order)
└── $_SESSION — (если session.auto_start=1)
```

#### Этап 3: Компиляция PHP-кода

PHP — компилируемо-интерпретируемый язык. Исходный код проходит через несколько этапов перед выполнением:

```text
┌──────────────────────────────────────────────────────────┐
│                    PHP Source Code                        │
│  <?php echo "Hello " . $name;                           │
└───────────────────────┬──────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│               1. Лексический анализ (Tokenizer)          │
│  T_OPEN_TAG, T_ECHO, T_CONSTANT_ENCAPSED_STRING,        │
│  T_CONCAT('.'), T_VARIABLE('$name'), T_SEMICOLON        │
└───────────────────────┬──────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│               2. Синтаксический анализ (Parser)          │
│  AST (Abstract Syntax Tree):                             │
│  └── Statement List                                      │
│       └── Echo                                           │
│            └── BinaryOp (CONCAT)                         │
│                 ├── String "Hello "                       │
│                 └── Variable $name                       │
└───────────────────────┬──────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│               3. Компиляция в опкоды                     │
│  0000 ECHO        string("Hello ")                       │
│  0001 ECHO        CV0($name)                             │
│  0002 RETURN      1                                      │
└───────────────────────┬──────────────────────────────────┘
                        ↓
┌──────────────────────────────────────────────────────────┐
│               4. OpCache (shared memory)                 │
│  Опкоды сохраняются → при следующем запросе               │
│  этапы 1-3 пропускаются полностью                        │
└──────────────────────────────────────────────────────────┘
```

Вы можете увидеть опкоды с помощью функции `opcache_get_status()` или расширения `phpdbg` / `vld`:

```bash
php -d vld.active=1 -d vld.execute=0 script.php
```

#### Этап 4: OpCache — как работает

OpCache — расширение, которое кэширует скомпилированные опкоды в **shared memory** (общую память между всеми воркерами FPM):

```text
┌──────────────────────────────────────────────┐
│           Shared Memory (OpCache)            │
│                                              │
│  /var/www/index.php     → [opcodes, mtime]   │
│  /var/www/app/Kernel.php → [opcodes, mtime]  │
│  /vendor/autoload.php   → [opcodes, mtime]   │
│  ...                                         │
│                                              │
│  Доступна ВСЕМ воркерам одного FPM-пула      │
└──────────────────────────────────────────────┘

Worker 1 ─┐
Worker 2 ─┤── читают из shared memory (без копирования!)
Worker 3 ─┘
```

Ключевые настройки OpCache:

```ini
; /etc/php/8.3/fpm/conf.d/10-opcache.ini
opcache.enable=1
opcache.memory_consumption=256          ; MB shared memory для опкодов
opcache.interned_strings_buffer=16      ; MB для интернированных строк
opcache.max_accelerated_files=20000     ; макс. количество файлов в кэше
opcache.validate_timestamps=1           ; проверять mtime файлов (0 на production!)
opcache.revalidate_freq=2               ; как часто проверять (сек), при validate_timestamps=1
opcache.save_comments=1                 ; сохранять PHPDoc (нужно для аннотаций Doctrine/Laravel)
opcache.preload=/var/www/preload.php    ; PHP 7.4+: предзагрузка файлов при старте FPM
opcache.preload_user=www-data
opcache.jit=1255                        ; PHP 8.0+: включить JIT
opcache.jit_buffer_size=128M            ; память для JIT-скомпилированного кода
```

Важно: на **production** рекомендуется `validate_timestamps=0` — OpCache не будет проверять изменения файлов, что даёт максимальную производительность. При деплое нужно сбросить кэш через `opcache_reset()` или перезапуск FPM.

#### Этап 5: Zend VM — исполнение опкодов

Zend VM — это стековая виртуальная машина, которая интерпретирует опкоды. Каждый опкод — это структура с handler-функцией, операндами и типом результата:

```text
Структура опкода:
┌─────────────────────────────────────────────┐
│ opcode   = ZEND_ADD                         │
│ op1      = CV0 ($a)     (compiled variable) │
│ op2      = CONST (5)    (литерал)           │
│ result   = TMP1         (временная перем.)  │
│ handler  = ZEND_ADD_SPEC_CV_CONST_HANDLER   │
└─────────────────────────────────────────────┘

Типы операндов:
  CV    — Compiled Variable (пользовательская переменная: $a, $b)
  CONST — Константа (числа, строки, true/false)
  TMP   — Временная переменная (промежуточные результаты)
  VAR   — Внутренняя переменная VM
```

Zend VM использует **специализированные handler'ы** — для каждой комбинации типов операндов генерируется отдельная C-функция. Например, `ZEND_ADD` имеет handler'ы для `LONG+LONG`, `DOUBLE+DOUBLE`, `CV+CONST` и т.д. Это устраняет динамическую диспетчеризацию типов и ускоряет выполнение.

#### Этап 6: JIT-компиляция (PHP 8.0+)

JIT (Just-In-Time) компилятор транслирует горячие опкоды в **нативный машинный код** (x86-64), минуя VM:

```text
Без JIT:                        С JIT:
Opcode → VM Handler (C)         Opcode → Машинный код (x86-64)
  ↓ вызов функции                 ↓ прямое выполнение CPU
  ↓ интерпретация                 ↓ без overhead VM
  ~медленнее                      ~быстрее для CPU-heavy

JIT режимы (opcache.jit=CRTO):
  C = CPU-specific optimizations (0-1)
  R = Register allocation (0-2)
  T = Trigger:
      0 = Выключен
      1 = При первом вызове
      2 = При первом вызове, с профилированием
      3 = По счётчику вызовов (горячий код)
      4 = По счётчику + при старте
      5 = По трассировке (tracing JIT)
  O = Оптимизация (0-5)

Рекомендуемые значения:
  opcache.jit=1255   — tracing JIT, максимальная оптимизация
  opcache.jit=disable — отключить JIT
```

JIT наиболее эффективен для **CPU-bound** задач (математика, обработка данных, рендеринг). Для типичных **I/O-bound** веб-приложений (запросы к БД, HTTP-вызовы) прирост от JIT минимален (~5-10%), потому что большая часть времени тратится на ожидание I/O.

#### Этап 7: Shutdown (RSHUTDOWN)

После выполнения кода воркер выполняет очистку:

```text
1. Вызов register_shutdown_function() callback'ов
2. Вызов деструкторов (__destruct) объектов
3. Flush буферов вывода (ob_end_flush)
4. RSHUTDOWN для каждого расширения:
   ├── session: session_write_close (сохранение сессии)
   ├── pdo: закрытие непостоянных соединений
   └── ...
5. Освобождение всех zval (переменных запроса)
6. Сброс symbol table (таблицы символов)
7. Очистка error/exception handler'ов
8. Отправка ответа через FastCGI → Nginx → клиент
9. Воркер возвращается в пул, готов к следующему запросу
```

### PHP Preloading (PHP 7.4+)

Preloading позволяет загрузить и скомпилировать файлы **один раз при старте FPM**, и они будут доступны всем воркерам без обращения к OpCache:

```php
// /var/www/preload.php
require_once __DIR__ . '/vendor/autoload.php';

// Предзагрузка ядра фреймворка
$files = new RecursiveIteratorIterator(
    new RecursiveDirectoryIterator(__DIR__ . '/vendor/laravel/framework/src')
);
foreach ($files as $file) {
    if ($file->getExtension() === 'php') {
        opcache_compile_file($file->getRealPath());
    }
}
```

```text
Без preloading:
Запрос → OpCache lookup → (miss) → Tokenize → Parse → Compile → Cache → Execute

С preloading:
Запрос → Файл уже в памяти (загружен при старте FPM) → Execute
```

Ограничение: preloaded-файлы нельзя изменить без перезапуска FPM. Подходит для production.

### Полная диаграмма жизненного цикла

```text
                    ┌─────────────────────────────────────────────┐
                    │            PHP-FPM MASTER                   │
                    │  (однократно при старте)                     │
                    │  ┌─────────────────────┐                    │
                    │  │ php.ini loading      │                    │
                    │  │ Extensions MINIT     │                    │
                    │  │ Pools configuration  │                    │
                    │  │ Preloading (opt.)    │                    │
                    │  │ Fork workers         │                    │
                    │  └─────────────────────┘                    │
                    └────────────┬────────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ↓                  ↓                  ↓
        ┌──────────┐      ┌──────────┐      ┌──────────┐
        │ Worker 1 │      │ Worker 2 │      │ Worker N │
        └────┬─────┘      └──────────┘      └──────────┘
             │
    ┌────────↓────────────────────────────────────────────┐
    │              ЦИКЛ ЗАПРОСА (повторяется)             │
    │                                                     │
    │  1. Accept FastCGI connection                       │
    │  2. RINIT (Request Init для расширений)             │
    │  3. Инициализация $_GET, $_POST, $_SERVER...        │
    │  4. ┌─ OpCache HIT? ──→ Загрузить опкоды           │
    │     └─ OpCache MISS? ──→ Tokenize → AST → Compile  │
    │                           ↓ сохранить в OpCache     │
    │  5. Zend VM исполняет опкоды                        │
    │     ├── JIT компиляция горячих путей (PHP 8+)       │
    │     ├── Autoload: require → compile → cache         │
    │     └── Бизнес-логика: DB, API, файлы               │
    │  6. Формирование output (echo, template rendering)  │
    │  7. RSHUTDOWN (shutdown functions, деструкторы)      │
    │  8. Отправка ответа через FastCGI                   │
    │  9. Очистка переменных, буферов, контекста          │
    │ 10. Возврат в пул (goto 1)                          │
    └─────────────────────────────────────────────────────┘

Сохраняется между запросами:          Очищается каждый запрос:
  ✓ Zend Engine                         ✗ Пользовательские переменные
  ✓ Loaded extensions                   ✗ $_GET, $_POST, $_SERVER...
  ✓ OpCache (shared memory)             ✗ Буферы вывода
  ✓ php.ini settings                    ✗ Error/exception handlers
  ✓ Preloaded classes                   ✗ Открытые файлы (кроме persistent)
  ✓ Persistent DB connections           ✗ Объекты и их деструкторы
  ✓ Interned strings                    ✗ Symbol table
```

### Мониторинг и отладка

Полезные инструменты для диагностики lifecycle:

```php
// Информация об OpCache
$status = opcache_get_status();
echo $status['opcache_statistics']['hits'];          // попадания в кэш
echo $status['opcache_statistics']['misses'];         // промахи
echo $status['memory_usage']['used_memory'];          // используемая память

// Информация о FPM (через status page)
// В pool.d/www.conf: pm.status_path = /fpm-status
// В Nginx: location /fpm-status { fastcgi_pass ...; include fastcgi_params; }
```

```bash
# Мониторинг воркеров FPM
curl http://localhost/fpm-status?full

# Пример вывода:
# pool:                 www
# process manager:      dynamic
# start since:          86400
# accepted conn:        150432
# listen queue:         0          ← если > 0, нужно больше воркеров!
# active processes:     12
# idle processes:       38
# total processes:      50
```

## Примеры
```php
// Быстрый check: OpCache включён?
var_dump(function_exists('opcache_get_status'));
```

## Доп. теория
- Ключевое отличие PHP‑FPM от CGI — пул долгоживущих воркеров, а не процесс на запрос.
- Большинство «магии» жизненного цикла привязано к хукам расширений: MINIT/RINIT/RSHUTDOWN/MSHUTDOWN.
