## Вопрос: Конфигурация и окружения
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Конфиг — это PHP-массивы, которые объединяются из нескольких файлов (main, params, db, local), чтобы разделять прод и дев.

## Ответ

### Принципы конфигурации в Yii2

Конфигурация в Yii2 — это обычные PHP-массивы, которые передаются в конструктор приложения. Каждый ключ массива соответствует свойству объекта приложения или его компоненту. Такой подход обеспечивает полную гибкость — можно использовать любые PHP-конструкции: условия, циклы, вызовы функций, подключение внешних файлов. Конфигурация применяется не только к приложению, но и к любому объекту, наследующему `yii\base\Configurable` — при создании объекта через `Yii::createObject()` массив конфигурации автоматически применяется к свойствам.

### Структура конфигурационных файлов

В basic-шаблоне конфигурация обычно разбита на несколько файлов: `web.php` (основной конфиг веб-приложения), `console.php` (консольное приложение), `db.php` (настройки БД), `params.php` (произвольные параметры). В advanced-шаблоне структура сложнее — есть общие файлы (`common/config/`) и файлы для каждого приложения (`frontend/config/`, `backend/config/`), которые мержатся через `yii\helpers\ArrayHelper::merge()`.

```php
// config/web.php — основной конфиг basic-шаблона
$params = require __DIR__ . '/params.php';
$db = require __DIR__ . '/db.php';

return [
    'id' => 'basic',
    'name' => 'My Application',
    'basePath' => dirname(__DIR__),
    'language' => 'ru-RU',
    'sourceLanguage' => 'en-US',
    'timeZone' => 'Europe/Moscow',
    'bootstrap' => ['log'],
    'components' => [
        'db' => $db,
        'cache' => [
            'class' => 'yii\caching\FileCache',
        ],
        'urlManager' => [
            'enablePrettyUrl' => true,
            'showScriptName' => false,
            'rules' => [],
        ],
        'log' => [
            'traceLevel' => YII_DEBUG ? 3 : 0,
            'targets' => [
                ['class' => 'yii\log\FileTarget', 'levels' => ['error', 'warning']],
            ],
        ],
    ],
    'params' => $params,
];
```

### Окружения (Environments)

В advanced-шаблоне используется механизм **environments** — каталог `environments/` содержит конфигурации для dev и prod. Команда `php init` копирует нужные файлы (`.env`, `*-local.php`) в рабочие директории. Локальные файлы (`main-local.php`, `params-local.php`) добавлены в `.gitignore` и содержат секреты (пароли БД, ключи API). Основные файлы (`main.php`) содержат общую конфигурацию и хранятся в Git.

```php
// environments/dev/common/config/main-local.php
return [
    'components' => [
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host=localhost;dbname=myapp_dev',
            'username' => 'root',
            'password' => '',
            'charset' => 'utf8',
        ],
    ],
];

// environments/prod/common/config/main-local.php
return [
    'components' => [
        'db' => [
            'class' => 'yii\db\Connection',
            'dsn' => 'mysql:host=db-server;dbname=myapp_prod',
            'username' => 'app_user',
            'password' => 'secure_password',
            'charset' => 'utf8',
            'enableSchemaCache' => true,
            'schemaCacheDuration' => 3600,
        ],
    ],
];
```

### Слияние конфигураций

Конфигурации объединяются с помощью `ArrayHelper::merge()`, который рекурсивно мержит массивы. Порядок слияния важен — локальные файлы подключаются последними и переопределяют общие настройки. Это позволяет каждому разработчику иметь свои параметры подключения к БД, не затрагивая общий конфиг.

```php
// frontend/config/main.php в advanced-шаблоне — точка входа
$params = array_merge(
    require __DIR__ . '/../../common/config/params.php',
    require __DIR__ . '/../../common/config/params-local.php',
    require __DIR__ . '/params.php',
    require __DIR__ . '/params-local.php'
);

return yii\helpers\ArrayHelper::merge(
    require __DIR__ . '/../../common/config/main.php',
    require __DIR__ . '/../../common/config/main-local.php',
    [
        'id' => 'app-frontend',
        'basePath' => dirname(__DIR__),
        'controllerNamespace' => 'frontend\controllers',
        'components' => [
            'request' => ['csrfParam' => '_csrf-frontend'],
            'session' => ['name' => 'frontend-session'],
        ],
        'params' => $params,
    ],
    require __DIR__ . '/main-local.php'
);
```

### Подключение Debug и Gii в dev

В dev-окружении обычно подключают модули Debug Toolbar и Gii. Это делается в `main-local.php`, чтобы модули никогда не попали на продакшен. Константы `YII_DEBUG` и `YII_ENV` устанавливаются в точке входа и влияют на поведение приложения — включение подробных ошибок, уровень трассировки логов и доступность отладочных инструментов.

```php
// web/index.php — точка входа
defined('YII_DEBUG') or define('YII_DEBUG', true);
defined('YII_ENV') or define('YII_ENV', 'dev');

// config/main-local.php для dev
$config = [];

if (YII_ENV_DEV) {
    $config['bootstrap'][] = 'debug';
    $config['modules']['debug'] = [
        'class' => 'yii\debug\Module',
        'allowedIPs' => ['127.0.0.1', '::1', '192.168.*'],
    ];

    $config['bootstrap'][] = 'gii';
    $config['modules']['gii'] = [
        'class' => 'yii\gii\Module',
        'allowedIPs' => ['127.0.0.1', '::1'],
    ];
}

return $config;
```

### Работа с параметрами (params)

Параметры (`params`) — это произвольный массив значений, доступный через `Yii::$app->params`. Сюда помещают настройки, не связанные с компонентами: email администратора, количество элементов на странице, ключи API сторонних сервисов. Параметры удобно разделять на общие и локальные.

```php
// config/params.php
return [
    'adminEmail' => 'admin@example.com',
    'supportEmail' => 'support@example.com',
    'paginationSize' => 20,
    'uploadMaxSize' => 5 * 1024 * 1024, // 5 MB
];

// Использование в коде
$adminEmail = Yii::$app->params['adminEmail'];
$pageSize = Yii::$app->params['paginationSize'];
```

### Практические советы

Никогда не храните пароли и ключи API в основных конфигурационных файлах — используйте `*-local.php` или переменные окружения (`getenv()`). В production обязательно устанавливайте `YII_DEBUG = false` и `YII_ENV = 'prod'` — это отключает подробные сообщения об ошибках и повышает безопасность. Включайте кэширование схемы БД (`enableSchemaCache`) на проде для ускорения. Используйте `ArrayHelper::merge()` для слияния, а не `array_merge()` — последний не мержит рекурсивно вложенные массивы.

## Примеры
1. Локальный пароль БД в `config/db-local.php`, общий DSN в `config/db.php`.
2. В dev подключить `debug` и `gii` в `main-local.php`, на проде — отключить.
3. Параметр `paginationSize` в `params.php`, чтение через `Yii::$app->params`.

## Доп. теория
1. Конфиг — это PHP, поэтому допустимы условия по окружению и переменные окружения.
2. `ArrayHelper::merge()` делает рекурсивное слияние и безопаснее `array_merge()` для вложенных массивов.
