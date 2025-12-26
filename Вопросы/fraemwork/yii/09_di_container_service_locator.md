## Вопрос: DI Container и Service Locator
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Есть DI-контейнер и сервис-локатор. Компоненты доступны через `Yii::$app`, а зависимости можно описывать в контейнере.

## Ответ
## Вопрос: Как устроены зависимости в Yii2?
Ответ: Есть DI-контейнер и сервис-локатор. Компоненты доступны через `Yii::$app`, а зависимости можно описывать в контейнере.

## Вопрос: Пример DI контейнера
Ответ:
```php
Yii::$container->set(LoggerInterface::class, FileLogger::class);

class ReportService {
    public function __construct(LoggerInterface $logger) {}
}
```

## Вопрос: Что такое Service Locator в Yii2?
Ответ:
- `Yii::$app->db`, `Yii::$app->cache`, `Yii::$app->mailer`.
- Удобно, но может скрывать зависимости.
