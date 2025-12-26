## Вопрос: Routing и UrlManager
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Преобразует URL в маршрут `controller/action` и обратно, поддерживает «красивые» URL.

## Ответ
## Вопрос: Что делает UrlManager?
Ответ: Преобразует URL в маршрут `controller/action` и обратно, поддерживает «красивые» URL.

## Вопрос: Пример правил
Ответ:
```php
'components' => [
    'urlManager' => [
        'enablePrettyUrl' => true,
        'showScriptName' => false,
        'rules' => [
            'post/<id:\\d+>' => 'post/view',
            'posts' => 'post/index',
        ],
    ],
],
```

## Вопрос: Что такое маршрут в Yii2?
Ответ:
- `site/index` -> `SiteController::actionIndex()`.
- Для REST есть `yii\\rest\\UrlRule`.
