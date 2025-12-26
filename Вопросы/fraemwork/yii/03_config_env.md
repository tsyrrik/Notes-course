## Вопрос: Конфигурация и окружения
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Конфиг — это PHP-массивы, которые объединяются из нескольких файлов (main, params, db, local), чтобы разделять прод и дев.

## Ответ
## Вопрос: Как задается конфиг в Yii2?
Ответ: Конфиг — это PHP-массивы, которые объединяются из нескольких файлов (main, params, db, local), чтобы разделять прод и дев.

## Вопрос: Пример конфигурации
Ответ:
```php
return [
    'id' => 'app',
    'basePath' => dirname(__DIR__),
    'components' => [
        'db' => require __DIR__ . '/db.php',
        'cache' => ['class' => 'yii\\caching\\FileCache'],
    ],
    'params' => require __DIR__ . '/params.php',
];
```

## Вопрос: Как устроены переопределения?
Ответ:
- `main.php` + `main-local.php`.
- `params.php` + `params-local.php`.
- Для dev обычно подключают Debug/Gii модули.
