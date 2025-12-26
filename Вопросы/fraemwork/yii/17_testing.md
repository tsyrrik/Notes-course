## Вопрос: Тестирование
Версия: Yii2 (ветка 2.0.x); Yii3 в разработке.

## Простой ответ
Обычно PHPUnit, плюс Codeception в шаблонах приложений.

## Ответ
## Вопрос: Что используется для тестов в Yii2?
Ответ: Обычно PHPUnit, плюс Codeception в шаблонах приложений.

## Вопрос: Какие виды тестов есть?
Ответ:
- Unit (модели, сервисы).
- Functional (контроллеры через тестовый клиент).
- Acceptance (браузерные).

## Вопрос: Пример unit-теста
Ответ:
```php
public function testValidation() {
    $model = new Post(['title' => '']);
    $this->assertFalse($model->validate());
}
```
