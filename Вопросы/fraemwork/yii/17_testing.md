## Вопрос: Тестирование
Версия: Yii2 2.0.54; Yii3 релизнут (2025‑12‑31).

## Простой ответ
Обычно PHPUnit, плюс Codeception в шаблонах приложений.

## Ответ

В Yii2 обычно используют PHPUnit, а в шаблонах приложений часто подключён Codeception. Типы тестов: unit, functional, acceptance.

Какие виды тестов есть:
- Unit (модели, сервисы).
- Functional (контроллеры через тестовый клиент).
- Acceptance (браузерные).

Пример unit‑теста:
```php
public function testValidation() {
    $model = new Post(['title' => '']);
    $this->assertFalse($model->validate());
}
```

## Примеры

1. Unit: проверить `rules()` модели.
2. Functional: `GET /post/index` возвращает 200.
3. Acceptance: сценарий логина через браузер.

## Доп. теория

1. Фикстуры ускоряют подготовку данных для тестов.
2. Моки помогут изолировать внешние сервисы (email, платежи).
