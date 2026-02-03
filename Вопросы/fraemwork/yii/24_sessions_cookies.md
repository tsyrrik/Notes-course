## Вопрос: Sessions и Cookies
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Через `Yii::$app->session` — открыть, читать/писать значения.

## Ответ

Сессии доступны через `Yii::$app->session`, cookies — через `request/response`.

Пример:
```php
$session = Yii::$app->session;
$session->set('user_id', 10);
$userId = $session->get('user_id');
```

Как работать с cookies:
- Чтение: `Yii::$app->request->cookies`.
- Запись: `Yii::$app->response->cookies`.

## Примеры

1. `Yii::$app->session->setFlash('success', 'Saved')`.
2. Установить cookie: `Yii::$app->response->cookies->add(...)`.
3. Прочитать cookie: `$theme = Yii::$app->request->cookies->getValue('theme');`.

## Доп. теория

1. Сессионную cookie нужно защищать `HttpOnly`, `Secure`, `SameSite`.
2. `flash`‑сообщения живут один запрос.
