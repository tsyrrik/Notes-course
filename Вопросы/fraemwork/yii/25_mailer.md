## Вопрос: Mailer
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Компонент для отправки писем; чаще используют расширение на Symfony Mailer, SwiftMailer считается устаревшим.

## Ответ
## Вопрос: Что такое Mailer в Yii2?
Ответ: Компонент для отправки писем. Сейчас чаще подключают Symfony Mailer через расширение, SwiftMailer официально устарел.

## Вопрос: Пример отправки письма
Ответ:
```php
Yii::$app->mailer->compose()
    ->setTo('user@example.com')
    ->setFrom('noreply@example.com')
    ->setSubject('Hello')
    ->setTextBody('Test')
    ->send();
```

## Вопрос: Где настраивается?
Ответ: В `components` конфигурации, указывается класс и транспорт.
