## Вопрос: Что ставит Composer внутри
Версия: PHP 8.4; где есть исторические детали — см. по тексту.

## Простой ответ
`composer install` кладёт зависимости в `vendor/` и генерирует файлы автозагрузки (`vendor/autoload.php`, `autoload_runtime.php`). Подключаете `vendor/autoload.php` — классы пакетов и ваши (из секций autoload) будут подгружаться автоматически.

## Ответ

### Структура vendor/

После `composer install` создаётся каталог `vendor/` со следующей структурой:

```
vendor/
├── autoload.php              ← точка входа, подключаете в коде
├── composer/
│   ├── autoload_classmap.php ← карта класс → файл (для classmap/optimized)
│   ├── autoload_namespaces.php ← PSR-0 маппинги
│   ├── autoload_psr4.php     ← PSR-4 маппинги (namespace → каталог)
│   ├── autoload_files.php    ← безусловно подключаемые файлы (helpers)
│   ├── autoload_real.php     ← основная логика загрузчика
│   ├── autoload_static.php   ← статическая версия (быстрее)
│   ├── ClassLoader.php       ← класс Composer\Autoload\ClassLoader
│   └── installed.php         ← метаданные установленных пакетов
├── package-name/             ← каталоги зависимостей
│   └── src/
└── ...
```

### Как работает autoload.php

```php
// vendor/autoload.php (упрощённо)
require_once __DIR__ . '/composer/autoload_real.php';
return ComposerAutoloaderInit::getLoader();
```

`getLoader()` делает следующее:
1. Создаёт экземпляр `ClassLoader`
2. Регистрирует PSR-4 маппинги из `autoload_psr4.php`
3. Загружает classmap из `autoload_classmap.php`
4. Подключает файлы из `autoload_files.php` (helpers)
5. Вызывает `spl_autoload_register()` для регистрации загрузчика

### ClassLoader — алгоритм поиска файла

При обращении к неизвестному классу:

1. **Classmap** — ищет в массиве `класс → файл` (O(1), самый быстрый)
2. **PSR-4** — по namespace определяет каталог, строит путь к файлу
3. Если нашёл — `require` файла, если нет — передаёт управление следующему autoloader'у

### Команды Composer

| Команда | Что делает |
| --- | --- |
| `composer install` | Устанавливает по `composer.lock` (точные версии) |
| `composer update` | Обновляет зависимости, перезаписывает `composer.lock` |
| `composer dump-autoload` | Перегенерирует файлы автозагрузки |
| `composer dump-autoload -o` | Генерирует optimized classmap (для production) |
| `composer dump-autoload --apcu` | Кеширует classmap в APCu |

### composer.lock

- Фиксирует точные версии всех зависимостей (включая транзитивные)
- **Коммитится в git** — чтобы у всех были одинаковые версии
- `composer install` использует `lock`, `composer update` обновляет `lock`

## Примеры
```php
require __DIR__ . '/vendor/autoload.php';

use Monolog\Logger;
use Monolog\Handler\StreamHandler;

$log = new Logger('app');
$log->pushHandler(new StreamHandler('php://stdout'));
$log->info('autoload ok');
```

## Доп. теория
- `composer dump-autoload -o` генерирует classmap и ускоряет автозагрузку в проде.
- PSR‑0 считается устаревшим, но его поддержка в autoload сохраняется для старых библиотек.
