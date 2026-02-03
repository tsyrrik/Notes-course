## Вопрос: Mailer
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Компонент для отправки писем; шаблоны Yii2 используют Symfony Mailer через расширение `yiisoft/yii2-symfonymailer`, SwiftMailer считается устаревшим.

## Ответ

Mailer — компонент для отправки писем. В актуальных шаблонах Yii2 используется Symfony Mailer через расширение `yiisoft/yii2-symfonymailer`, SwiftMailer считается устаревшим.

Пример отправки письма:
```php
Yii::$app->mailer->compose()
    ->setTo('user@example.com')
    ->setFrom('noreply@example.com')
    ->setSubject('Hello')
    ->setTextBody('Test')
    ->send();
```

Где настраивается:
В `components` конфигурации, указывается класс и транспорт.

## Примеры

1. Отправка письма с `setTextBody()` и `setHtmlBody()`.
2. Письмо с шаблоном: `->compose('welcome', ['user' => $user])`.
3. Транспорт SMTP в `components['mailer']`.

## Доп. теория

1. В dev можно включить `useFileTransport` для записи писем в файлы.
2. SMTP‑ошибки лучше логировать и повторять через очередь.
