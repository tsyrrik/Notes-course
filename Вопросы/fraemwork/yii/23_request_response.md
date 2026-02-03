## Вопрос: Request и Response
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Через `Yii::$app->request` — данные GET/POST, headers, cookies.

## Ответ

Работа с запросом идёт через `Yii::$app->request` (GET/POST, headers, cookies).

Пример чтения параметров:
```php
$id = Yii::$app->request->get('id');
$name = Yii::$app->request->post('name');
```

Как формировать ответ:
- `Yii::$app->response->format = Response::FORMAT_JSON;`
- `return ['ok' => true];`

## Примеры

1. `Yii::$app->request->isPost` для проверки метода.
2. JSON‑ответ через `Response::FORMAT_JSON`.
3. Отправка файла: `return Yii::$app->response->sendFile($path);`.

## Доп. теория

1. `request` и `response` — компоненты приложения, их можно конфигурировать.
2. `response->format` влияет на сериализацию результатов action.
