## Вопрос: Sessions и Cookies
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Через `Yii::$app->session` — открыть, читать/писать значения.

## Ответ
## Вопрос: Как работать с сессиями?
Ответ: Через `Yii::$app->session` — открыть, читать/писать значения.

## Вопрос: Пример
Ответ:
```php
$session = Yii::$app->session;
$session->set('user_id', 10);
$userId = $session->get('user_id');
```

## Вопрос: Как работать с cookies?
Ответ:
- Чтение: `Yii::$app->request->cookies`.
- Запись: `Yii::$app->response->cookies`.
