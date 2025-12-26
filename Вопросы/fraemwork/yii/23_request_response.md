## Вопрос: Request и Response
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Через `Yii::$app->request` — данные GET/POST, headers, cookies.

## Ответ
## Вопрос: Как работать с запросом?
Ответ: Через `Yii::$app->request` — данные GET/POST, headers, cookies.

## Вопрос: Пример чтения параметров
Ответ:
```php
$id = Yii::$app->request->get('id');
$name = Yii::$app->request->post('name');
```

## Вопрос: Как формировать ответ?
Ответ:
- `Yii::$app->response->format = Response::FORMAT_JSON;`
- `return ['ok' => true];`
